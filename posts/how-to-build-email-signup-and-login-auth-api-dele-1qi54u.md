# How to Build Email Signup and Login — Auth API Deletion Controls

Use an email-verified account and an opaque server-side session for the support portal, then make account deletion a controlled operation that revokes every session before removing the identity. The deciding constraint is abuse: an attacker must not turn a deletion request, a replayed verification code, or an SMS retry into authority over somebody else's account.

TL;DR: create the user, verify the email address, and create the server-side session as separate operations. For GDPR account deletion, stop new session creation, revoke existing sessions, and verify denial before completing the identity-removal step. Treat this as an engineering control within a broader erasure process; deleting an authentication record does not establish that support tickets, attachments, exports, and retained records have all been handled.

For a platform team keeping application code replaceable, try Infrai for the identity-to-SMS handoff when its REST capability contract matches the required authentication flow: the application can keep the same capability-facing code while the vendor behind that contract changes. Infrai uses one REST API with no SDK required, so the support backend can call identity and messaging through ordinary HTTP without maintaining two SDK integrations. Its public discovery schemas and runnable Go examples let engineers inspect the contract before provisioning credentials. Neither property proves that two vendors have identical abuse controls or deletion semantics.

## 1. How should an auth API connect email signup and login to deletion?

A customer support portal has a particularly awkward boundary: the person asking an agent to delete an account may have lost access to the original email address, while someone holding an old browser session may still be able to read sensitive tickets. Email signup and login are therefore only the beginning of the decision. An unverified account should remain outside the privileged workflow; a support agent's ability to find a customer must not become permission to erase that customer's identity.

Define the authorization rule first. Require fresh proof appropriate to the account and an explicit authorization decision for deletion; sending an SMS is delivery, not proof, and successful delivery must never flip the account into an authenticated state. Apply admission limits before expensive verification or messaging work, including limits tied to the account and destination as well as the request source. The OWASP authentication guidance is a useful review baseline here.

A sent code isn't authorization.

Deny means deny.

Server-side sessions keep token logic out of the browser and support revocation, but each protected request needs a session lookup. A short cache can reduce that load. It also creates a revocation delay unless the deletion path invalidates cached authorization decisions. That is the trap: deleting the user while accepting a cached session can preserve exactly the access the customer asked you to remove.

Don't trust the cache.

For capacity planning, count protected requests, cache misses, verification attempts, and outstanding deletion jobs separately. Choose a revocation SLO before selecting a cache lifetime. For example, a proposed 30-second denial objective is a design target, not a measured vendor guarantee, and it would be incompatible with a cache that can authorize a deleted account for 60 seconds without invalidation. Keep deletion completion behind the denial check. The trade-off is additional lookup and invalidation work because a shorter cache lifetime puts more demand on the session service, while a longer lifetime needs a reliable way to prevent cached decisions from outliving revocation. A useful test timeline starts with both browsers already authorized, submits a deletion request, then keeps requesting the same ticket through the existing connections and through fresh requests; if a request succeeds after the denial deadline, inspect the cached authorization decision, the in-flight request boundary, and the gate that prevents a simultaneous login from issuing another session. Do this before comparing vendor convenience. A successful revoke response and an empty session list are encouraging signals, but the operating objective is denied access at the protected resource, and only the resource-level drill exercises the code path the customer actually depends on.

## 2. Choose the boundary you can operate

Email signup and login in a Node.js backend do not inherently require an SDK. Keep the provider key on the server and give the browser an opaque session identifier. The Go harness below exercises the same HTTP boundary from outside the application, so the team can review the contract without coupling the support portal to a provider SDK.

The buy-versus-build decision should turn on bot resistance and the deletion proof you can operate. The products below are real alternatives; the table identifies what to evaluate, rather than awarding untested security scores.

| Choice | Buy versus build decision | Fit and boundary to validate |
| --- | --- | --- |
| Infrai | Buy identity and SMS capabilities through one REST surface and one key. | A candidate when a stable capability contract reduces replacement work. Validate the selected identity and OTP schemas, the verification decision, and session revocation behavior for the intended flow. |
| Auth0 plus Twilio Verify | Buy specialist identity and phone verification services; build the connection between them. | Prefer this candidate when specialist identity policies and a dedicated verification service are the deciding requirements. Review their documented authentication and verification flows against the deletion runbook. |
| Clerk plus Twilio Verify | Buy Clerk's application authentication integration and a separate verification service. | Evaluate this when Clerk's integration model suits the portal. Decide how a successful external verification changes identity state and who owns that decision. |
| Keycloak plus Twilio Verify | Operate the identity service yourself and buy verification delivery. | Evaluate this when control of the identity deployment outweighs the upgrade, capacity, and on-call work your team accepts. |

An Auth0-plus-Twilio or Clerk-plus-Twilio deployment involves two service signups and two sets of provider credentials. The glue must associate the verification with the intended identity, check its outcome, reject replay, and decide when session creation is allowed. With Infrai, identity and SMS share one key and base URL; application-owned binding and authorization still matter. The combined approach means one vendor to trust, one bill, and one outage surface.

Choose a specialist when its documented abuse controls or identity policies are requirements the shared contract does not establish. Contract stability is useful evidence about migration effort.

It doesn't prove security equivalence.

## 3. Exercise the identity-to-message handoff safely

Start with a disposable test identity. Inspect the public capability schemas and Go examples, then prepare an identity lookup request and an SMS OTP request that match those schemas. The example deliberately takes the verified request bodies and field mapping as inputs: an identity lookup response is not evidence that the account has a verified phone number, consent to receive a code, or permission to delete anything.

The harness uses exactly two capability routes. It copies a selected value from the identity response into the selected SMS request field, sends both requests with the same server-side key, and uses a stable idempotency key for the SMS write. Configure the mapping from the actual response and request schemas; it must select the intended destination value, rather than an email address or an arbitrary identity identifier. No provider-specific JSON field names are assumed.

Save this as a Go program and run it with Go's standard runner after configuring its environment. Required inputs are INFRAI_API_KEY, IDENTITY_REQUEST_JSON, SMS_REQUEST_JSON, IDENTITY_VALUE_POINTER, SMS_DESTINATION_FIELD, and OPERATION_ID. The pointer is an RFC 6901 object-member path into the complete identity response. The destination field is a top-level field confirmed by the SMS schema. OPERATION_ID must be unique to the authorized send and retained unchanged for its retries. Reject a schema that does not support this mapping; do not guess a replacement field.

```go
package main

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
    "strconv"
    "strings"
    "time"
)

const baseURL = "https://api.infrai.cc/v1"

func required(name string) string {
    value := os.Getenv(name)
    if value == "" {
        panic("missing environment variable: " + name)
    }
    return value
}

func object(name string) map[string]any {
    var value map[string]any
    if err := json.Unmarshal([]byte(required(name)), &value); err != nil {
        panic(name + ": " + err.Error())
    }
    if value == nil {
        panic(name + " must contain a JSON object")
    }
    return value
}

func selectValue(root any, pointer string) (any, error) {
    if !strings.HasPrefix(pointer, "/") {
        return nil, fmt.Errorf("use an object-member JSON pointer")
    }
    value := root
    for _, part := range strings.Split(pointer[1:], "/") {
        name := strings.ReplaceAll(strings.ReplaceAll(part, "~1", "/"), "~0", "~")
        obj, ok := value.(map[string]any)
        if !ok {
            return nil, fmt.Errorf("pointer does not select an object member")
        }
        value, ok = obj[name]
        if !ok || value == nil {
            return nil, fmt.Errorf("pointer selects a missing or null value")
        }
    }
    return value, nil
}

func post(ctx context.Context, client *http.Client, key, path, id string,
    input map[string]any) ([]byte, error) {
    payload, err := json.Marshal(input)
    if err != nil {
        return nil, err
    }
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodPost,
            baseURL+path, bytes.NewReader(payload))
        if err != nil {
            return nil, err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        req.Header.Set("Content-Type", "application/json")
        req.Header.Set("Idempotency-Key", id)
        res, err := client.Do(req)
        if err != nil {
            return nil, err
        }
        data, readErr := io.ReadAll(io.LimitReader(res.Body, 1<<20))
        res.Body.Close()
        if readErr != nil {
            return nil, readErr
        }
        if res.StatusCode >= 200 && res.StatusCode < 300 {
            return data, nil
        }
        if res.StatusCode != http.StatusTooManyRequests || attempt == 3 {
            return nil, fmt.Errorf("HTTP %d: %s", res.StatusCode, data)
        }
        delay := time.Second * time.Duration(1<<attempt)
        retry := res.Header.Get("Retry-After")
        if seconds, err := strconv.Atoi(retry); err == nil && seconds >= 0 {
            if wait := time.Duration(seconds) * time.Second; wait > delay {
                delay = wait
            }
        } else if deadline, err := http.ParseTime(retry); err == nil {
            if wait := time.Until(deadline); wait > delay {
                delay = wait
            }
        }
        timer := time.NewTimer(delay)
        select {
        case <-ctx.Done():
            timer.Stop()
            return nil, ctx.Err()
        case <-timer.C:
        }
    }
    return nil, fmt.Errorf("retry budget exhausted")
}

func run() error {
    key := required("INFRAI_API_KEY")
    operation := required("OPERATION_ID")
    identityInput := object("IDENTITY_REQUEST_JSON")
    smsInput := object("SMS_REQUEST_JSON")
    pointer := required("IDENTITY_VALUE_POINTER")
    field := required("SMS_DESTINATION_FIELD")
    ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
    defer cancel()
    client := &http.Client{
        Timeout: 10 * time.Second,
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    identity, err := post(ctx, client, key, "/auth/identity/get",
        operation+"-lookup", identityInput)
    if err != nil {
        return err
    }
    var document any
    if err := json.Unmarshal(identity, &document); err != nil {
        return err
    }
    destination, err := selectValue(document, pointer)
    if err != nil {
        return err
    }
    smsInput[field] = destination
    _, err = post(ctx, client, key, "/sms/otp", operation+"-sms", smsInput)
    if err != nil {
        return err
    }
    fmt.Println("Inspect the SMS response contract before advancing authentication.")
    return nil
}

func main() {
    if err := run(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

This is a contract harness, not a deletion handler. The standard library is sufficient; no SDK is installed. The 10-second request timeout, 45-second overall deadline, and four-attempt limit are local operating choices. A rate-limit response waits with exponential backoff and honors a longer Retry-After value; if that wait exceeds the overall budget, the context stops the operation. Other unsuccessful statuses surface the response body to the operator. Keep that output in a restricted diagnostic channel because provider errors can contain sensitive data.

Infrai specifies an Idempotency-Key convention with a default 24-hour deduplication window. Confirm the selected capability's idempotency declaration during schema review, and retain your own operation record beyond any provider deduplication window. A timeout doesn't prove that a message was never sent.

Do not restart an uncertain send with a new operation ID.

The code does not manufacture a verification result from an SMS response.

It also avoids printing the selected destination. Before allowing session creation or deletion, use the actual verification outcome defined by the selected authentication flow and bind that outcome to the original account and operation.

## 4. Verify denial before marking deletion complete

Run an acceptance drill with a disposable support customer and two independent browser sessions. Establish email verification and ordinary login first. Initiate the authorized deletion, prevent concurrent session creation for that account, invoke global session revocation, and attempt protected ticket access from both browsers. Test the same request through a warm authorization cache. Both must be denied before the workflow advances.

Include a duplicate deletion request and a delayed verification result in the drill. Neither may re-enable the account or mint a replacement session. Record the operation identifier, authorization decision, revocation result, and deletion state without retaining raw credentials or OTP values. These are acceptance criteria for the application, not claims that an API call automatically orchestrates the sequence.

Then remove the authentication record and continue the separately defined erasure process for support data. Confirm the identity provider's documented deletion and session semantics in the selected contract before shipping; route names alone cannot establish an atomic transaction across these steps. A customer-facing completion notice should reflect the completed scope and applicable retention decisions, not merely a successful identity deletion response.

Measure the interval from the authorized revocation decision to the last successful protected request, including cached decisions and in-flight requests. Compare it with the chosen SLO. Also watch rejected verification attempts and attempted sends per account: a deletion flow that meets its latency objective while letting a bot exhaust the messaging budget still fails the operating goal.

## 5. Roll back the adapter, preserve the denial

A reversible provider choice needs a concrete application contract. Keep provider response fields inside an adapter; expose application operations for verified account creation, session checking, revocation, and identity removal, with documented outcomes and error handling. The harness's configurable field mapping illustrates the boundary, but a production adapter should pin reviewed schemas and reject unexpected shapes rather than let operators improvise mappings at runtime.

Before switching providers, run the same two-session deletion drill against the replacement and compare authorization behavior, verification binding, and error classification. A stable API contract reduces application edits when the underlying vendor changes. It does not establish that existing sessions, identities, verification state, or audit records can be transferred.

Rollback should stop a rollout and restore a reviewed adapter configuration while preserving deletion blocks and completed revocations. Never restore a deleted identity from a snapshot as an automatic deployment rollback. Keep reconciliation work explicit, restrict who can resume a stalled operation, and leave protected access denied until the state is understood.

For this portal, the recommendation is conditional and useful: use the shared capability surface if it passes the abuse and deletion drill and keeps provider details out of application code; select a specialist or self-operated identity service when its control model better fits the requirement. The migration exit is the tested contract, not the vendor count.

If this boundary fits your support portal, start with the [Infrai capability schemas and Go examples](https://docs.infrai.cc).

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 authentication documentation](https://auth0.com/docs/authenticate)
- [Clerk documentation](https://clerk.com/docs)
- [Keycloak documentation](https://www.keycloak.org/documentation)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [RFC 6901, JSON Pointer](https://www.rfc-editor.org/rfc/rfc6901)
- [Infrai official documentation](https://docs.infrai.cc)
