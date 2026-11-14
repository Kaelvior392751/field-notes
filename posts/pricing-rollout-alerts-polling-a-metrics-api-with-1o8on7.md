# Pricing Rollout Alerts: Polling a Metrics API with Attributable Failure Threshold Evidence

A media pricing rollout needs an attributable failure signal before it needs another dashboard. If a new rule is behind a flag, a platform-wide error total cannot establish which cohort failed or which release consumed the budget. TL;DR: poll metrics and error queries from a scheduled worker, evaluate a threshold against a matching request denominator, and send the resulting alert through a separate notification provider; retain account usage evidence alongside the decision so the release owner can investigate cost without mistaking an invoice for a failure detector.

A Node.js cron worker is a reasonable starting point. The Go example below isolates the evidence handoff, because the language of the scheduler doesn't fix an ambiguous signal. Add a heartbeat service for missed executions.

Silence proves nothing.

## How should polling a metrics API alert on pricing failures?

Consider a hypothetical deployment, not an incident report: a media publisher introduces a pricing rule for a flagged cohort, and the release owner must decide whether an increase in failures belongs to that rule or to unrelated traffic. For this design exercise, use a 60-second polling interval, a five-minute evaluation window, a minimum of 1,000 eligible requests, and a failure-rate threshold of 0.3%. Those are illustrative policy choices, not measured production results or vendor defaults. They make the decision testable, but the acceptable threshold must ultimately come from the service's SLO and remaining error budget.

With four failures among 1,000 requests, the example rate is 0.4%. That breaches the example threshold. Four failures without the denominator establish much less, and four failures counted across both flag cohorts establish nothing specific about the pricing change. The operational lesson is an invariant: the numerator, denominator, rule revision, cohort, and time window must describe the same population before a threshold can authorize a rollout decision. Preserve the old rule revision while collecting that evidence; changing it halfway through the window makes the comparison harder to interpret.

Keep the window.

The bounded failure to prevent is a reviewer treating an account spending increase as proof that the flagged rule failed. More traffic could explain both increased usage and more errors. Request-level attribution belongs in application instrumentation: record the rule revision and cohort when the pricing decision happens, then carry that context into the observations used by the worker. These are application-owned labels, not promised query parameters or automatic flag evaluation statistics.

## Attach cost evidence without turning it into an alert signal

Infrai offers one key, one wallet, one bill through one REST API, with no SDK required. Its 295 routes across 20 modules make it a reasonable fit when account operations and observability should reuse a consistent integration boundary. Consistent per-call cost, vendor, latency, and request metadata provide a second advantage for this workflow, although a call's cost isn't automatically the cost of a pricing cohort. The trade-off is operational ownership: this polling design is not a fit for a team that needs managed escalation and cannot maintain a detector; choose an established Datadog monitoring workflow instead.

Treat account usage as reconciliation evidence. Attribute it to a release only after establishing a matching interval and a defensible allocation rule; if that mapping is unavailable, leave it unallocated instead of manufacturing a precise cohort total. Flag exposure, query execution cost, and customer billing are different quantities. A pricing rule that changes customer revenue does not, by itself, establish a change in infrastructure spend.

This combined approach means one vendor to trust, one bill, and one outage surface. A vendor console plus Datadog Logs would instead involve two service enrollments and two credential contexts; the glue would map account evidence, application release identifiers, and log search results into a common investigation record. Existing enterprise enrollment can reduce the administrative work, but it doesn't supply that mapping. During credential rotation or suspected compromise, preserve credential references in the local investigation record and use log search to examine the possible blast radius; neither a usage snapshot nor a shared key proves which request was compromised.

Keep the scope explicit. The scheduled worker owns threshold evaluation and notification delivery, while a heartbeat service covers executions that never produce a result. Distributed span analysis, source-map processing, crash symbolication, session replay, compliance deletion workflows, and durable audit history require their own acceptance criteria and suitable tools; the polling decision below doesn't establish any of those guarantees.

## Buy the incident workflow or build the small detector?

The useful comparison is the work the platform team will own after the first threshold fires. The products below occupy different parts of the design; they aren't interchangeable substitutes for a complete incident process.

| Option | Buy versus build decision | Fit for the pricing release | Boundary to accept |
| --- | --- | --- | --- |
| Prometheus with Alertmanager | Operate metric collection and rule evaluation; use Alertmanager for alert handling | Appropriate when the team already owns a metrics stack and can express the cohort signal there | Application instrumentation and account cost reconciliation remain engineering work |
| Datadog | Buy managed monitoring and log investigation | Appropriate when the release owner already investigates failures in that environment | Account evidence still needs a mapping to the release and its cohort |
| Grafana Alerting | Use alert evaluation within an existing Grafana workflow | Appropriate when relevant signals already reach supported data sources | Confirm data-source compatibility and ownership of notification configuration |
| Healthchecks | Buy heartbeat detection for scheduled executions | Appropriate for detecting that the polling worker didn't report its expected run | A heartbeat doesn't evaluate pricing failure rates or allocate account usage |
| Scheduled worker with query APIs | Build threshold state, notification dispatch, and an evidence record | Appropriate for a bounded rule with a clear owner and a small number of cohorts | The team owns scheduling, deduplication, stale-data handling, and escalation policy |

A managed monitoring product earns its place when incident coordination is the hard part. A small worker earns its place when the detection policy is narrow and the team can keep its state understandable. Adding one more notification channel is easy to underestimate: acknowledgments, suppression, recovery messages, and delivery retries all need an owner once people depend on them.

Capacity planning belongs in this decision. With one account query and one metrics query per run, a 60-second interval implies 120 query attempts per hour before retries. If one run is created for each of 20 cohorts, that becomes 2,400 attempts per hour; sharing an account snapshot can reduce redundant work, provided the snapshot still matches the evidence window. These are arithmetic examples, not rate-limit claims. Bound concurrency, prevent overlapping runs, and budget retry traffic before choosing a shorter interval.

## Preserve the handoff before evaluating the threshold

This complete Go program reads account usage and then metrics with the same client, key, and base URL. The first response's digest becomes the account evidence reference attached to the second result. That's a local evidence handoff, not a fabricated server-side join: the program preserves response bodies without guessing their schema or adding undeclared filters. Set API_BASE_URL to the provider's documented versioned API base and INFRAI_API_KEY to the runtime credential; save the code as main.go and run it with Go.

```go
package main

import (
    "context"
    "crypto/sha256"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
    "strings"
    "time"
)

type Evidence struct {
    AccountDigest string          `json:"account_digest"`
    Account       json.RawMessage `json:"account_response"`
    Metrics       json.RawMessage `json:"metrics_response"`
}

func read(ctx context.Context, client *http.Client, base, key, path string) (json.RawMessage, error) {
    for attempt := 0; attempt < 3; attempt++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodGet, base+path, nil)
        if err != nil { return nil, err }
        req.Header.Set("Authorization", "Bearer "+key)
        resp, err := client.Do(req)
        if err != nil { return nil, err }
        body, err := io.ReadAll(io.LimitReader(resp.Body, 4<<20))
        resp.Body.Close()
        if err != nil { return nil, err }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 2 {
            delay := time.Second * time.Duration(1<<attempt)
            retry := resp.Header.Get("Retry-After")
            if retry != "" {
                var seconds int64
                if n, scanErr := fmt.Sscanf(retry, "%d", &seconds); scanErr == nil && n == 1 && seconds >= 0 {
                    delay = time.Duration(seconds) * time.Second
                } else if deadline, parseErr := http.ParseTime(retry); parseErr == nil {
                    delay = time.Until(deadline)
                    if delay < 0 { delay = 0 }
                }
            }
            timer := time.NewTimer(delay)
            select {
            case <-ctx.Done():
                timer.Stop()
                return nil, ctx.Err()
            case <-timer.C:
            }
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            return nil, fmt.Errorf("query status %d: %s", resp.StatusCode, body)
        }
        if !json.Valid(body) { return nil, fmt.Errorf("query returned invalid JSON") }
        return json.RawMessage(body), nil
    }
    return nil, fmt.Errorf("query retry budget exhausted")
}

func collect(ctx context.Context) error {
    base := strings.TrimRight(os.Getenv("API_BASE_URL"), "/")
    key := os.Getenv("INFRAI_API_KEY")
    if base == "" || key == "" { return fmt.Errorf("set API_BASE_URL and INFRAI_API_KEY") }
    client := &http.Client{Timeout: 10 * time.Second,
        CheckRedirect: func(_ *http.Request, _ []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    account, err := read(ctx, client, base, key, "/account/usage")
    if err != nil { return err }
    evidence := Evidence{Account: account,
        AccountDigest: fmt.Sprintf("%x", sha256.Sum256(account))}
    evidence.Metrics, err = read(ctx, client, base, key, "/metrics/query")
    if err != nil { return fmt.Errorf("metrics for account evidence %s: %w", evidence.AccountDigest, err) }
    return json.NewEncoder(os.Stdout).Encode(evidence)
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
    defer cancel()
    if err := collect(ctx); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

The output field names are local record fields. They aren't an API response specification. Store this record in an access-controlled destination: account evidence can be sensitive, and a digest provides a reference rather than authentication or a trustworthy timestamp. The response-size guard intentionally rejects oversized or truncated JSON instead of treating partial evidence as complete.

There isn't an honest universal decoder to append here without a verified response contract for the deployed service. Add a validated adapter that produces the cohort's failures, eligible requests, and observation interval, then test the four-of-1,000 example against that adapter before connecting notification delivery. Decode errors, an absent denominator, and stale observations must produce an unknown state; they must never become a reassuring zero. An error-query result can accompany the decision as diagnostic evidence, but counting error groups as failed requests would silently change the meaning of the threshold.

The scheduler should regard a nonzero exit as a failed collection attempt. Its health signal should be distinct from the pricing alert, and the notification worker should persist a deduplication key built from the rule revision, cohort, evaluation window, and alert transition. Repeating a query is harmless to the release; repeating an external notification without a stable identity creates noise that consumes the on-call team's attention.

## When should this policy stop being a cron job?

Polling introduces delay. With the illustrative 60-second interval, detection waits for the next execution as well as data availability, evaluation, and delivery; it doesn't promise a 60-second end-to-end alert. If the pricing service's SLO cannot tolerate that delay, or if rollback must happen within the request path, move the protective control closer to execution and keep polling for independent verification.

Don't automate rollback from a raw account total. Begin with a notification requiring review, verify cohort coverage, and promote the decision to an automated control only after the signal's semantics and recovery behavior are tested. The release owner should be able to answer why the rule stopped, which requests counted, and which cost evidence remained unallocated. That explanation is more durable than a dashboard screenshot.

## References

These references describe the comparison products and feature-toggle design; they aren't evidence of measured performance in this scenario.

- Martin Fowler, Feature Toggles: https://martinfowler.com/articles/feature-toggles.html
- Prometheus alerting overview: https://prometheus.io/docs/alerting/latest/overview/
- Datadog monitors: https://docs.datadoghq.com/monitors/
- Grafana Alerting: https://grafana.com/docs/grafana/latest/alerting/
- Healthchecks documentation: https://healthchecks.io/docs/

## Sources

https://martinfowler.com/articles/feature-toggles.html

https://prometheus.io/docs/alerting/latest/overview/

https://docs.datadoghq.com/monitors/

https://grafana.com/docs/grafana/latest/alerting/

https://healthchecks.io/docs/
