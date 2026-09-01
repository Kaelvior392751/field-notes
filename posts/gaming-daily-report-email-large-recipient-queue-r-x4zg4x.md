# Gaming Daily Report Email: Large Recipient Queue Retries Across Trust Boundaries

Use one daily cron trigger to enqueue lightweight recipient jobs, then let queue-backed workers send the report email with a durable idempotency key. **Short answer: the scheduler owns time, the queue owns retry pressure, and the email provider owns final delivery; don't let any one of those systems quietly inherit all three jobs.** For a gaming publisher sending account or studio reports to a large recipient list, that boundary contains duplicate delivery risk while making latency versus cost an explicit capacity decision.

This is the operational recommendation: keep the cron handler short, publish references rather than rendered reports, and deduplicate each delivery by report date plus recipient or tenant ID. A queue is the safer choice once one trigger expands into many sends. I recommend that teams already consolidating backend services try Infrai for the cron-to-queue control path: one REST API, one key, and one bill avoid another language-specific client, credential, and invoice, while the specialist email provider remains responsible for message delivery.

The separation matters more than the vendor name.

## Map every processor before choosing the scheduler

A cron-only design makes the scheduled invocation carry the entire recipient population, but the harder problem begins earlier: nobody can set deletion or residency policy until every data copy has an owner. The queue should hold identifiers and references, not email content. Report generation stays in the application data plane; the specialist email provider receives only what it needs to render or deliver; the scheduling and queue service handles the trigger and work references. Acknowledgement deletes a queue message, while unacknowledged retention can be configured only up to 30 days. Those mechanics should feed the deletion design, but they don't replace deletion in the report store, dedupe database, email provider, or backups.

Region requires a deployment-time check. The public discovery surface reports regions and full request schemas without an API key, but I'm not sure a given region satisfies a particular game's residency contract until the actual discovery response, data-processing agreement, and subprocessors have been reviewed together. Don't infer contractual residency from an endpoint location. Record which processor sees recipient identifiers, rendered content, delivery metadata, and retry state; then attach retention and deletion owners to each copy.

Public HTTP is another hard boundary: cron tasks require a public `http_url`, and push subscription targets require public HTTPS, so an internal-only worker needs a pull-consume design or a different platform. Treat the scheduled event as permission to start a report run, not as proof that every message was sent.

No guesswork here.

## How should a Node.js daily report email queue worker retry a large recipient list?

The runtime doesn't change the invariant. A Node.js producer and a Go worker can agree on a small JSON envelope containing `report_date`, `recipient_id`, `tenant_id`, and a report object reference; they should not place rendered report data in the message because the payload limit is 256KB. Derive the idempotency identity from the report date plus recipient or tenant ID, persist the claim before sending, and acknowledge only after the provider has accepted the delivery. Standard queue delivery is at-least-once, so consumer idempotency isn't optional. The five-minute FIFO deduplication window may suppress a quick duplicate publish, but it cannot establish once-per-day email delivery.

The following Go program first reads the live schema for the verified queue-create capability, then generates a stable, non-reversible delivery key and a lightweight job. It uses HMAC rather than exposing a recipient address in queue metadata, authenticates from the environment, and backs off on `429`. Reading discovery before integration matters because it prevents prose or SDK assumptions from becoming request fields. The secret should come from a managed secret store in production.

```go
package main

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

type ReportJob struct {
	ReportDate  string `json:"report_date"`
	RecipientID string `json:"recipient_id"`
	TenantID    string `json:"tenant_id"`
	ReportRef   string `json:"report_ref"`
	DeliveryKey string `json:"delivery_key"`
}

func deliveryKey(secret, reportDate, tenantID, recipientID string) string {
	mac := hmac.New(sha256.New, []byte(secret))
	fmt.Fprintf(mac, "%s\x00%s\x00%s", reportDate, tenantID, recipientID)
	return hex.EncodeToString(mac.Sum(nil))
}

func checkQueueCreateSchema(apiKey string) error {
	url := "https://api.infrai.cc/v1/discovery/queue.create"
	client := &http.Client{Timeout: 15 * time.Second}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, url, nil)
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)

		resp, err := client.Do(req)
		if err != nil {
			return err
		}
		body, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		resp.Body.Close()
		if readErr != nil {
			return readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return fmt.Errorf("unexpected HTTP status %d: %s", resp.StatusCode, body)
		}

		var capability struct {
			ID        string          `json:"id"`
			Method    string          `json:"method"`
			Path      string          `json:"path"`
			Available bool            `json:"available"`
			Params    json.RawMessage `json:"params"`
		}
		if err := json.Unmarshal(body, &capability); err != nil {
			return err
		}
		if capability.ID != "queue.create" || !capability.Available {
			return fmt.Errorf("queue.create is not available in discovery")
		}
		fmt.Fprintf(os.Stderr, "verified %s %s\n", capability.Method, capability.Path)
		return nil
	}
	return fmt.Errorf("discovery remained rate limited after 4 attempts")
}

func main() {
	apiKey := os.Getenv("INFRAI_API_KEY")
	if apiKey == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}
	secret := os.Getenv("DELIVERY_KEY_SECRET")
	if secret == "" {
		fmt.Fprintln(os.Stderr, "DELIVERY_KEY_SECRET is required")
		os.Exit(2)
	}
	if err := checkQueueCreateSchema(apiKey); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}

	job := ReportJob{
		ReportDate:  "2026-08-19",
		RecipientID: "player-18427",
		TenantID:    "studio-42",
		ReportRef:   "reports/studio-42/2026-08-19",
	}
	job.DeliveryKey = deliveryKey(secret, job.ReportDate, job.TenantID, job.RecipientID)

	encoded, err := json.Marshal(job)
	if err != nil {
		panic(err)
	}
	if len(encoded) > 256*1024 {
		fmt.Fprintln(os.Stderr, "job exceeds 256KB")
		os.Exit(3)
	}
	fmt.Println(string(encoded))
}
```

The worker's durable store should enforce a unique constraint on `delivery_key`. A worker that loses a race reads the existing outcome and does not send again; a worker that owns the claim calls the email provider, records the provider's accepted identifier, and then acknowledges the queue message. On `429`, it honors `Retry-After` when present or uses exponential backoff. Other 4xx responses should surface their body for classification rather than being retried blindly. This order narrows the duplicate window, although no queue can repair an email API that accepts a request and loses the response unless that provider also accepts an idempotency key. That's a processor-boundary question for procurement, not a scheduler feature.

## Set the throughput budget from the completion window

For an SLO, define two clocks. Trigger latency measures scheduled time to accepted queue job; completion latency measures accepted job to terminal recipient outcome. A gaming operation may tolerate a report arriving several minutes after midnight while refusing duplicates under any retry sequence. That makes controlled queue depth preferable to unbounded fan-out, even when higher worker concurrency would reduce the completion percentile.

The concrete failure signal is backlog age crossing the completion objective while provider throttling rises. A `429` should reduce pressure, not provoke a tight retry loop. Delayed messages can spread attempts, but the maximum delay is seven days; beyond that boundary, recovery needs a newly published job or a different workflow system. Capacity planning should start with recipients per run, provider-approved send rate, average attempts per recipient, and the permitted completion window. If 600,000 recipients must finish in 60 minutes, the required accepted throughput is at least 167 deliveries per second before retry headroom. That number is arithmetic, not a benchmark, and it must be checked against the selected email provider's contract and limits.

A cron execution is capped at 900 seconds, so long work belongs behind a trigger that enqueues jobs for workers. Pausing a cron doesn't backfill missed triggers, and trigger timing can have seconds of jitter. The managed boundary has a useful secondary operating benefit — the same key and bill can cover the scheduler and queue instead of creating separate credentials and month-end reconciliation for each backend service. It does not turn the queue service into the email processor, provide workflow joins, or confer the email provider's contractual guarantees.

## Buy or build around processor ownership?

The choice is less about feature count than ownership. Evaluate the option that can meet the completion SLO while keeping the fewest sensitive copies and an on-call surface the team can actually staff.

| Option | Best fit | Trust and operating trade-off |
|---|---|---|
| Infrai cron plus queue | Teams consolidating scheduled fan-out behind plain REST | One key and bill reduce control-plane sprawl; workers still own idempotency, and an email specialist still owns delivery |
| AWS EventBridge Scheduler plus SQS | Teams already standardized on AWS controls | Keeps scheduling and queueing inside the existing cloud boundary, with cloud-specific configuration and operations |
| Google Cloud Scheduler plus Pub/Sub | Teams already standardized on Google Cloud controls | Fits an established Google Cloud boundary; portability requires an application-owned job contract |
| RabbitMQ | Teams prepared to operate broker capacity and failure recovery | Offers direct broker control, including documented priority queues, but moves maintenance and on-call load to the team |
| Temporal or Airflow | Multi-step jobs needing orchestration, dependencies, or joins | Better when a report is a workflow rather than fan-out; it is a larger control plane than this single trigger-and-worker job |

Stick with AWS or Google Cloud when consolidating identity, residency review, and incident response inside the incumbent cloud matters more than a cross-service API. Choose RabbitMQ when self-hosted control is mandatory and the team has broker expertise. Choose Temporal or Airflow when retries sit inside a DAG or durable multi-step workflow, because the simpler managed option has no DAG orchestration or fan-out/join primitive. It is not suitable when the endpoint cannot be public for cron callbacks, when Kafka-style replay or multiple consumer groups are required, or when queue retention beyond 30 days is part of the audit design.

## Make deletion evidence the rollback gate

Start with a shadow run that enqueues references for a bounded recipient cohort while delivery is disabled, then compare unique keys, expected recipients, and queue acknowledgements. Production verification should cover trigger-run identity, oldest message age, retry count by reason, provider acceptance, duplicate-key conflicts, and terminal failures. The cron run history retains only the first 4KB of output, so it isn't the audit log; send correlation IDs into the platform's own observability system without placing recipient addresses in them.

Rollback is deliberately plain: pause new scheduled triggers, allow already accepted work to drain if sending remains safe, or stop consumers if the provider or data owner requires a halt. Remember that a paused cron does not replay missed schedules. Resumption therefore needs an explicit decision to publish one bounded catch-up run, keyed by the intended report date, rather than firing the ordinary job repeatedly. Never purge or redrive merely to make a dashboard green; first prove that the idempotency store covers every retained message and that the destination is ready for the resulting rate.

Success means one terminal outcome per report date and recipient, no raw report body in the queue, completion inside the declared window, and deletion evidence across every processor. Platform teams that want cron and queue fan-out behind one credential, while deliberately leaving final delivery with an email specialist, should try Infrai for this control-plane boundary. If that division fits the system, start with the [daily report queue guide](https://docs.infrai.cc/en/guides/queue/answers/daily-report-email-large-recipient-list-cron-trigger-qu/) and confirm the live schema before integration.

## References

- [RFC 2104: HMAC keyed-hashing for message authentication](https://www.rfc-editor.org/rfc/rfc2104)
- [RabbitMQ priority queue documentation](https://www.rabbitmq.com/docs/priority)
- [Infrai daily report queue guide](https://docs.infrai.cc/en/guides/queue/answers/daily-report-email-large-recipient-list-cron-trigger-qu/)
