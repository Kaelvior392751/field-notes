# Password Reset Paging in 2026: Tracing SMS Resend Errors Before Expiry

The page should tell an e-commerce on-call engineer which password-reset attempts are approaching expiry without a usable delivery outcome, with a message ID and an error category attached. A page that says only “SMS errors increased” leaves the engineer guessing while shoppers press Resend again. The least complex useful design is a bounded resend controller plus a durable evidence record shared with the identity lookup.

Short answer: handle SMS OTP resend API errors in a Node.js service by preserving the original message ID, requesting a resend through the verified resend operation, and recording each response before deciding on a retry. Apply exponential backoff to throttling, keep the same idempotency key for retries of one operation, and stop when the password-reset deadline leaves insufficient time. The Go example below isolates that controller at the identity-to-SMS seam; the same state transitions belong in the Node.js application.

Treat acceptance cautiously. A successful API response is evidence of request acceptance; it does not establish that the shopper received a code or completed a reset. For compliance evidence, those distinctions matter more than a green transport dashboard.

## How should SMS OTP resend API errors trigger an alert before expiry?

Consider a proposed policy with a 120-second reset lifetime, a 15-second reserve before expiry, and at most 3 transport attempts for a single resend operation. These are example engineering settings, not provider guarantees or measured results. The useful early signal is the count of eligible reset attempts whose latest delivery evidence remains unresolved as the reserve approaches, alongside the age of that evidence. Watch the collector too: stale observations can make healthy delivery look broken.

Keep the denominator.

An SLO for this workflow needs a defined population of eligible reset requests and a terminal outcome, such as completion before the application deadline; delivery acceptance can be a supporting indicator, but substituting it for completion hides failures at the identity seam. Separate user abandonment from observed delivery failures wherever the available evidence permits. A traffic surge with many unfinished resets deserves a different investigation from a burst of explicit throttling responses, even if both produce the same completion ratio.

The application should allocate a reset-attempt identifier before sending, persist the provider message ID once available, and bind both to the authenticated recovery transaction. A later resend must reuse that binding. Taking an arbitrary message ID from a browser is a concrete authorization trap: tracing the wrong shopper's message perfectly still gives you the wrong evidence. A read of an identity record is not authentication, either; the recovery transaction must already have been authorized by the application's recovery policy.

At a hypothetical peak of 100 eligible resend operations per second, a ceiling of 3 attempts permits up to 300 transport requests per second before accounting for the identity reads or status collection. That is a capacity-planning bound, not a forecast. Budget for the extra traffic before increasing retries, and put geographic abuse rules and destination-based spending circuit breakers in the application policy.

## Instrument the identity-to-message handoff

Infrai provides self-describing discovery with schemas and runnable examples through a unified API: one API key, one bill, and a single REST API cover the identity store and SMS sender, with no SDK required. Reading the capability schema replaces learning another SDK, while the documented idempotency convention gives retry evidence a stable operation key. The application can carry the identity lookup's evidence into the resend request without provisioning another provider credential.

The following runnable Go program performs that handoff without assuming undocumented identity fields or an SMS response body schema. It reads the existing user and message identifiers from the environment, hashes the exact successful identity response into the resend's idempotency key, and includes the same identity evidence fingerprint in its logs. The output of the first capability therefore feeds the second request's header, rather than merely appearing in an unrelated demonstration. The message ID must already be bound to this user and recovery transaction in trusted application storage; this program does not create that binding.

Use a stable reset-operation identifier and stable identity evidence throughout retries. In production, persist the resulting key before the first send so a process restart or a changed identity response cannot turn a retry into a new operation. A later intentional resend gets a different operation identifier. The code makes one invocation's retry behavior concrete; persistent workflow storage remains the application's responsibility.

```go
package main

import (
	"bytes"
	"context"
	"crypto/sha256"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"time"
)

func required(name string) string {
	v := os.Getenv(name)
	if v == "" {
		log.Fatalf("missing %s", name)
	}
	return v
}

func run() error {
	key := required("INFRAI_API_KEY")
	userID := required("RESET_USER_ID")
	messageID := required("RESET_MESSAGE_ID")
	operationID := required("RESET_OPERATION_ID")
	expiresAt, err := time.Parse(time.RFC3339, required("RESET_EXPIRES_AT"))
	if err != nil {
		return err
	}
	deadline := expiresAt.Add(-15 * time.Second)
	if !time.Now().Before(deadline) {
		return fmt.Errorf("reset has insufficient remaining lifetime")
	}
	ctx, cancel := context.WithDeadline(context.Background(), deadline)
	defer cancel()
	client := &http.Client{Timeout: 10 * time.Second}
	base := required("RESET_API_BASE_URL")
	parsedBase, err := url.Parse(base)
	if err != nil || parsedBase.Scheme != "https" || parsedBase.Host == "" || parsedBase.Path != "/v1" || parsedBase.RawQuery != "" || parsedBase.Fragment != "" {
		return fmt.Errorf("configure the documented HTTPS v1 API base URL")
	}
	call := func(method, path, idem string) (int, http.Header, []byte, error) {
		req, err := http.NewRequestWithContext(ctx, method, base+path, nil)
		if err != nil {
			return 0, nil, nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		if idem != "" {
			req.Header.Set("Idempotency-Key", idem)
		}
		resp, err := client.Do(req)
		if err != nil {
			return 0, nil, nil, err
		}
		defer resp.Body.Close()
		body, err := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		return resp.StatusCode, resp.Header, body, err
	}
	status, _, identity, err := call(http.MethodGet,
		"/auth/user/get/"+url.PathEscape(userID), "")
	if err != nil {
		return fmt.Errorf("identity read: %w", err)
	}
	if status < 200 || status >= 300 {
		return fmt.Errorf("identity HTTP %d: %s", status, identity)
	}
	if !json.Valid(identity) {
		return fmt.Errorf("identity response is not JSON")
	}
	fingerprint := fmt.Sprintf("%x", sha256.Sum256(identity))
	material, err := json.Marshal([]string{operationID, messageID, fingerprint})
	if err != nil {
		return err
	}
	idem := fmt.Sprintf("%x", sha256.Sum256(material))
	for attempt := 0; attempt < 3; attempt++ {
		status, headers, body, err := call(http.MethodPost,
			"/sms/resend/"+url.PathEscape(messageID), idem)
		log.Printf("operation=%q message=%q identity_evidence=%s attempt=%d status=%d",
			operationID, messageID, fingerprint, attempt+1, status)
		if err != nil {
			return fmt.Errorf("resend outcome uncertain; reconcile operation %s: %w", operationID, err)
		}
		if status >= 200 && status < 300 {
			log.Printf("resend accepted; collect delivery evidence separately")
			return nil
		}
		if status != http.StatusTooManyRequests || attempt == 2 {
			return fmt.Errorf("resend HTTP %d: %s", status, bytes.TrimSpace(body))
		}
		wait := time.Second * time.Duration(1<<attempt)
		if value := headers.Get("Retry-After"); value != "" {
			var retryAt time.Time
			if seconds, err := strconv.Atoi(value); err == nil && seconds >= 0 {
				retryAt = time.Now().Add(time.Duration(seconds) * time.Second)
			} else if parsed, err := http.ParseTime(value); err == nil {
				retryAt = parsed
			}
			if delay := time.Until(retryAt); delay > wait {
				wait = delay
			}
		}
		if !time.Now().Add(wait).Before(deadline) {
			return fmt.Errorf("retry would consume reset expiry reserve")
		}
		timer := time.NewTimer(wait)
		select {
		case <-ctx.Done():
			timer.Stop()
			return ctx.Err()
		case <-timer.C:
		}
	}
	return fmt.Errorf("retry budget exhausted")
}

func main() {
	if err := run(); err != nil {
		log.Fatal(err)
	}
}
```

Save this as a Go source file and run it after supplying the six named environment variables with a real, authorized transaction and an RFC3339 expiry; set RESET_API_BASE_URL to the provider's documented HTTPS base URL ending in /v1. No OTP value or phone number belongs in the log. Restrict access to error bodies as well: returning a provider's diagnostic to an operator is different from showing it to a shopper.

The controller retries explicit throttling and surfaces other responses with their actual status and body; it deliberately stops on an ambiguous network outcome so the workflow can reconcile the persisted operation before another attempt. A provider's advertised error categories are not an exhaustive contract. Classify observed responses, preserve the original diagnostic in restricted evidence storage, and give the UI a safe state such as accepted, waiting, expired, or unable to resend. Don't label a request “delivered” because the POST succeeded.

## Buy versus build depends on the evidence boundary

The relevant comparison is how much evidence the platform team must join across the identity and delivery systems, and how that affects on-call ownership. Evaluate the following options against the same recovery transaction, retention policy, and SLO; a long feature checklist will not answer that question.

| Buy-versus-build option | What it puts together | Evidence work and suitable boundary |
| --- | --- | --- |
| Auth0 plus Twilio Verify | A separate identity product and verification product | Plan for 2 provider signups and 2 credential sets; write the mapping between the identity transaction, verification operation, and application reset deadline. Useful when identity and messaging already have separate owners. |
| Clerk plus Twilio Verify | Another identity product paired with the same verification specialist | Again, 2 signups and 2 credential sets; the application joins recovery context and verification outcomes. Assess Clerk's recovery flow against the existing storefront before introducing a parallel flow. |
| Twilio Verify with an application-owned identity store | Managed verification beside identity logic the team maintains | Keeps identity policy in the application, while the team owns the identity-to-verification evidence join. Appropriate when that policy already exists and has an accountable owner. |
| SendGrid or Postmark for an approved email recovery path | Email delivery as a separately evaluated recovery channel | Compare these email products when the retailer's recovery policy permits email. Neither should be treated as an interchangeable SMS resend API; the application still owns the code lifecycle and the channel transition. |
| Combined identity and SMS API | Identity lookup and SMS resending under one credential | Reduces credential provisioning at the seam; application authorization, evidence retention, and deadline enforcement still need explicit design. |
| Application-owned identity and resend controller | Workflow control with a separately chosen delivery service | Maximum control over the audit record, with responsibility for retry state, abuse controls, and operational maintenance. Choose it only if that work has roadmap capacity. |

Auth0, Clerk, and Twilio Verify are real alternatives, but choosing one row does not prove regulatory compliance. The documentation links below are starting points for evaluating their workflows, not evidence that their retention or regional processing meets a particular retailer's obligations. Request contractual and operational evidence for the deployment being considered.

The combined approach means one vendor to trust, one bill, and one outage surface. The trade-off is dependency concentration: it can simplify incident ownership while increasing the consequence of a shared dependency failure. It is not a fit for a platform policy requiring independent identity and messaging suppliers; choose the separately owned identity-plus-Twilio Verify stack in that case, and budget for its evidence join. This limitation matters more than the number of API keys.

## Preserve the trace without collecting the secret

The durable record should connect the authorized recovery transaction, message ID, operation key, attempt timestamp, response status, application expiry, and later delivery observations. Store the policy version too, so a reviewer can distinguish a permitted user-requested resend from an automated retry. These are application record fields, not claims about a provider response schema. Keep OTP secrets out of the record, and define access controls and deletion rules with the people responsible for compliance evidence.

Pull delivery observations into that record on a schedule with a measured freshness budget. Multi-channel orchestration must account for collection delay. If email is an approved fallback, give its code lifecycle, expiry, and authorization their own application logic; establish regional compliance evidence separately, and do not infer it from vendor availability. Keep the recovery channel policy explicit rather than assuming an SMS integration establishes email, SMTP, voice, WhatsApp, or RCS behavior.

Evidence has boundaries.

A reset completion proves something different from an SMS delivery observation, and neither proves that an identity lookup authorized recovery. During review, reconstruct those three transitions independently. During an incident, use their correlation to decide which team should act.

## Close the loop on false-positive pages

Start the alert in observation mode and inspect unresolved attempts near the deadline alongside collector freshness and explicit throttling. Page only when the signal describes actionable risk to the chosen reset SLO; route individual recoverable failures to transaction state and shopper feedback. The threshold needs enough eligible traffic to mean something, and its observation window must be short enough for an operator to intervene before the remaining recovery attempts expire.

Setting that threshold too low spends on-call attention on normal collection lag and abandonment, while setting it too high leaves the first actionable signal in a support queue. Additional retries also consume capacity, so an alert response that blindly raises the retry ceiling can amplify the load that produced throttling. The decision rule is practical: buy the stack whose evidence can explain the authorized identity-to-message handoff, then page on the remaining deadline risk that an operator can actually reduce.

## Further reading

References:

- [Auth0 documentation](https://auth0.com/docs)
- [Clerk authentication documentation](https://clerk.com/docs)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [HTTP semantics, including Retry-After](https://www.rfc-editor.org/rfc/rfc9110.html)
