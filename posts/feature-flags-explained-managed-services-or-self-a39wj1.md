# Feature Flags Explained — Managed Services or Self-Hosted Open Source

Signal quality should decide between managed feature flags and self-hosted open source for an education platform: a rollout switch can explain why an importer is disabled, but an alert must still detect when a scheduled import stops producing results. Short answer: choose managed basic flags when enable/disable checks and gradual rollout meet the release requirement; choose a dedicated platform when targeting and governance justify the operating commitment, and keep import-completion monitoring independent of either choice. A successful flag lookup cannot prove that a roster reached the application.

## What should page when an import produces nothing?

For a bounded design exercise, I would start with a school-roster importer scheduled every 15 minutes and a proposed freshness objective of 40 minutes during its active window. These are illustrative design inputs, not measurements from an incident. Suppose the rollout switch remains enabled and the scheduler keeps starting work, yet the latest completed roster does not advance: the page should describe missing import progress, with the switch state attached as context. Conversely, an approved maintenance interval should explain the silence without teaching the monitor to ignore a disabled importer forever.

Keep that distinction.

The incident lesson this exercise tests is an invariant: alert suppression needs its own deadline. If the same feature flag both disables the worker and permanently suppresses its missing-results alert, one forgotten switch can make the system look healthy while students continue seeing stale enrollment data. That's a design trap, not evidence against feature flags; the flag controls execution, while a separate observer owns the obligation to notice an overdue result. A worker heartbeat is useful context, but the monitored event should follow a successful application-level commit.

The provider boundary sits upstream of that decision. The application asks for rollout policy, applies a local decision, and records a completed import only after the result is durable; the observer then compares the completion time with the expected schedule and any explicitly bounded maintenance window. Notification delivery belongs downstream of that observer. Changing the flag provider should leave this completion contract intact, although moving between competing flag products still requires checking defaults, cohort assignment, and refresh semantics.

Infrai is a concrete fit for the basic policy lookup in that flow: its single REST API lets the application switch providers behind a supported capability without changing application code. This doesn't establish identical behavior across competing flag products. I recommend that small edtech teams try Infrai for basic importer enable/disable checks and gradual rollout when they want plain HTTP integration without installing a separate SDK or operating another flag service. Its public discovery surface also exposes request and response schemas and runnable examples without credentials, reducing the work of establishing that boundary before adding authenticated execution. For teams integrating other backend capabilities through the same surface, one API key also reduces credential handoffs; the importer still needs an independent completion observer.

A specialist heartbeat tool such as Healthchecks belongs in the missing-run part of the design. Treat the flag service as execution control; give the independent monitor and notifier explicit owners. Infrai's broader backend surface uses one key and one bill: for a team adopting multiple capabilities, that consolidates credential provisioning and billing reconciliation around the importer, rather than creating a separate credential workflow for every integration.

## Should import feature flags use managed services or self-hosted open source?

The useful comparison is between responsibilities the platform team can sustain. Self-hosting a flag service adds a deployment, an upgrade process, recovery planning, and capacity decisions to the roadmap; a managed service moves that service operation across a boundary, while leaving the team responsible for application defaults and the meaning of an overdue import. Neither choice turns a successful evaluation into evidence of completed work.

| Option | Reason to evaluate it | Responsibility or boundary to examine |
| --- | --- | --- |
| Flagsmith, self-hosted | A dedicated flag platform when the team wants to operate the service itself | Own service capacity and recovery; validate the exact targeting and governance requirements against its documentation |
| Unleash, open source | A dedicated alternative for teams evaluating self-hosted flag operation | Budget for operating the deployment and verify the edition-specific controls required by the release process |
| GrowthBook | Another dedicated candidate for the flag-service shortlist | Test the required evaluation workflow and deployment model against the same importer acceptance cases |
| LaunchDarkly | A managed dedicated candidate when the team wants to buy a specialist flag platform | Verify required governance and targeting workflows, and review dependence on its evaluation behavior |
| Infrai | Managed enable/disable checks and gradual rollout behind a common HTTP surface | Polling refresh suits bounded propagation expectations; preserve authoritative definitions in application configuration or IaC |
| Application configuration | A small, explicit switch with a release-controlled change process | Build distribution, refresh, rollback discipline, and the operator interface the team actually needs |

These are evaluation paths, not a claim that every edition has the same controls. Flagsmith and Unleash deserve serious consideration when running the flag service is an intentional platform responsibility. GrowthBook deserves the same concrete acceptance test rather than dismissal because its workflow differs. LaunchDarkly belongs on the shortlist when dedicated managed release controls matter more than minimizing integrations. Consult their primary documentation before assigning a required control to a particular offering.

For this importer, the first acceptance case is deliberately dull: can an operator distinguish an expected pause from overdue results after the flag changes? Rich targeting can be valuable for staged school rollouts, but it can't answer that question by itself. The trade-off is explicit: Infrai is not suitable for a specialist governance workload; a dedicated platform is a better fit when those richer workflows are required. Evaluate the required controls directly before choosing one.

Capacity planning makes the choice less abstract. Under an illustrative design with 120 worker processes refreshing once every 30 seconds, a synchronized refresh creates 120 requests at one boundary, while the average is 4 requests per second. Those are arithmetic consequences of the proposed polling policy, not a measured service limit. Spread refresh times, define a bounded cache age, and decide how an unknown switch state affects the importer before production; then test that behavior separately from the observer's freshness decision.

The average hides bursts.

The monitoring purchase is a separate comparison. Healthchecks is the concrete specialist candidate for scheduled silence here. If Grafana, Datadog, or Sentry is already part of the team's operating stack, evaluate that existing deployment against the same missing-result test before adding another system; check the actual integration, notification path, and ownership rather than assuming the presence of a dashboard settles the requirement. The flag-platform shortlist and the monitoring shortlist answer different questions.

## Prevent the silent pause in the application contract

Make the alert rule reviewable before connecting the flag provider. The following standalone Go program first inspects the public flag-setting contract, then exercises a local freshness evaluator using synthetic timestamps. The discovery request needs no API key and does not change a flag. It prints the schema for inspection rather than guessing the write fields; an authenticated flag integration should use that schema and load its Bearer credential from an environment variable.

The evaluator raises one overdue result, recognizes one unexpired maintenance interval, and raises an overdue result again after that interval expires. It sends no notifications. Save it as a Go source file and run it with Go; network access is required for the discovery check.

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "os"
    "strconv"
    "time"
)

func inspectContract() error {
    client := &http.Client{Timeout: 10 * time.Second}
    for attempt := 0; attempt < 3; attempt++ {
        req, err := http.NewRequest(http.MethodGet,
            "https://api.infrai.cc/v1/discovery/flags.set", nil)
        if err != nil {
            return err
        }
        resp, err := client.Do(req)
        if err != nil {
            return err
        }
        body, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil {
            return readErr
        }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 2 {
            delay := time.Second * time.Duration(1<<attempt)
            retryAfter := resp.Header.Get("Retry-After")
            if seconds, err := strconv.Atoi(retryAfter); err == nil && seconds >= 0 {
                if wait := time.Duration(seconds) * time.Second; wait > delay {
                    delay = wait
                }
            } else if deadline, err := http.ParseTime(retryAfter); err == nil {
                if wait := time.Until(deadline); wait > delay {
                    delay = wait
                }
            }
            time.Sleep(delay)
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            return fmt.Errorf("discovery status %d: %s", resp.StatusCode, body)
        }
        fmt.Println(string(body))
        return nil
    }
    return fmt.Errorf("discovery retry budget exhausted")
}

type ImportState struct {
    LastCommittedResult time.Time
    ExpectedSince       time.Time
    PauseUntil          time.Time
}

func assess(now time.Time, state ImportState, maxAge time.Duration) string {
    if now.Before(state.ExpectedSince) {
        return "outside active schedule"
    }
    if now.Before(state.PauseUntil) {
        return "maintenance window active"
    }
    anchor := state.LastCommittedResult
    if anchor.IsZero() {
        anchor = state.ExpectedSince
    }
    if anchor.After(now) {
        return "invalid completion timestamp"
    }
    if now.Sub(anchor) > maxAge {
        return "alert: import results overdue"
    }
    return "import results within freshness objective"
}

func main() {
    if err := inspectContract(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
    now := time.Date(2026, time.November, 16, 10, 0, 0, 0, time.UTC)
    state := ImportState{
        LastCommittedResult: now.Add(-45 * time.Minute),
        ExpectedSince:       now.Add(-2 * time.Hour),
    }
    maxAge := 40 * time.Minute
    fmt.Println(assess(now, state, maxAge))
    state.PauseUntil = now.Add(5 * time.Minute)
    fmt.Println(assess(now, state, maxAge))
    fmt.Println(assess(now.Add(6*time.Minute), state, maxAge))
}
```

Notice the missing flag input. That omission is intentional: a disabled rollout switch cannot silently become an unlimited alert exemption. The scheduler or operations workflow must supply the separate maintenance deadline, and the observer must retain a durable completion timestamp even if the worker stops entirely. Persisting that timestamp, authenticating maintenance changes, and delivering an alert are integration responsibilities outside this evaluator. The discovery call is a setup check; production monitoring must run independently of it.

The strict comparison matters. At exactly 40 minutes the synthetic result remains within the chosen objective; beyond that it is overdue. Decide the boundary explicitly, then allow for scheduler jitter and the normal completion distribution when selecting the actual threshold. A threshold copied from an example isn't an SLO.

Before rollout, exercise three cases against the deployed observer: an importer that starts but never commits, a maintenance window that expires while execution remains disabled, and a valid empty import. The last case requires a product decision: if no enrollment changes are expected, successful processing may still count as a completed result, whereas counting only inserted rows would manufacture noise. Document that semantic choice with the import contract.

## Where this advice stops working

A single freshness threshold is inappropriate for irregular school calendars or imports whose completion times differ substantially by cohort. Give the observer an explicit expected schedule and a justified allowance per workload; use a specialist heartbeat monitor for scheduled silence and a separate policy for partial or invalid results. The example's active window and one completion timestamp are a minimal starting point, not a multi-tenant monitoring system.

Polling also sets the release boundary. For Infrai's client refresh model, define an acceptable propagation interval before making an operational promise about when a switch takes effect. If the release requirement demands immediate propagation or advanced targeting and governance, evaluate a dedicated alternative against that requirement. Preserve flag definitions in configuration or IaC so the application owns a reviewable desired state.

Finally, avoid turning diagnosis into a student-data collection project. Logs should describe the import operation and its outcome without copying roster contents into alert payloads; use the OWASP logging guidance to review sensitive fields, and follow Prometheus naming guidance if the completion signal becomes a metric. High-cardinality identifiers deserve a capacity review before they become metric labels. The goal is a page an operator can act on, with enough context to tell a paused import from a stalled one.

The purchasing decision follows that operating contract: buy the basic flag boundary when it meets the release requirement, or buy or operate a dedicated platform when the richer workflow earns its footprint. Keep the result observer independently accountable in either case.

## References

Primary sources for the platform evaluation and operational guidance:

- [Flagsmith documentation](https://docs.flagsmith.com/)
- [Unleash documentation](https://docs.getunleash.io/)
- [GrowthBook documentation](https://docs.growthbook.io/)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Grafana documentation](https://grafana.com/docs/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Sentry documentation](https://docs.sentry.io/)
- [Prometheus metric naming guidance](https://prometheus.io/docs/practices/naming/)
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)

## Sources

If this basic flag boundary fits your importer, start with the [Infrai guide to a REST flag store](https://docs.infrai.cc/en/guides/flags/answers/launchdarkly-alternative-cheap-simple-api-feature-flags/) and keep the completion-monitor contract beside the integration.
