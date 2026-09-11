# Media Training Retention: Large App Backup File Multipart Upload in Node.js

Short answer: for large media-training artifacts, create a reproducible archive first, upload it with multipart object storage, and make retention and deletion decisions in a durable ledger rather than in the upload worker. Complete only after every part is verified; abort an upload when its source, policy version, or ownership cannot be proven.

That constraint changes the implementation. A resumable transfer is useful only when the thing being resumed is still the same artifact and the same retention decision. Otherwise, “resume” is just a convenient way to publish a file whose contents or deletion date no longer match the training run.

## Retention evidence for a large app backup file multipart upload

Media teams tend to call several different things a backup: a source video, a transcoded derivative, a label manifest, and the model-training metadata that ties them together. Put those inputs into one immutable archive, or record a manifest with cryptographic digests for each object, before starting the transfer. The archive key should include an application or dataset identifier, a run identifier, and a policy revision; it should not depend on a mutable display name.

The ledger needs more than an upload ID. Store the source digest, byte length, object key, retention class, deletion deadline, policy revision, current state, and the part number plus returned checksum or entity tag for every accepted part. A worker restart can then distinguish an interrupted upload from a new training run. A second worker must acquire a lease on the ledger row before it acts, because object storage is a data plane, not the coordinator for your backup schedule.

Keep the lifecycle states boring: prepared, uploading, ready, delete-pending, deleted, and aborted. A completed multipart upload becomes a candidate for restore only after the ledger transaction records completion and the final object metadata matches the archive manifest. The delete path should make its own durable transition, so a retry does not turn a timeout into an invented assumption that deletion happened.

The retention policy is the product requirement here. “Keep the latest artifact” is not reproducible if latest is calculated from a changing query; store the selection result and the policy version that produced it. For every delete, retain an audit record containing the actor, reason, policy revision, object key, and timestamp. You may delete bytes, but you shouldn't delete the evidence that explains why.

Keep that ledger append-only.

## How can a Node.js worker implement resume for a large app backup file multipart upload?

The application can be written in Node.js while the transfer boundary follows the S3 multipart model: initiate an upload, send numbered parts, complete with the accepted part list, or abort the upload. The durable state machine is the important part. A process-local array of successful parts is not a resume mechanism.

Use a fixed source file and bounded concurrency. Each part should be independently retryable, and the worker should persist a part result before treating that part as done. The completion request must be assembled from the ledger, sorted by part number, and checked against the expected part count. Never infer completion from a 200 response returned by an unrelated operation, from a worker exit, or from a log line that arrived before the response body was read. The failure case that deserves the longest explanation is a worker that loses its lease after uploading a part but before committing the part result: a replacement worker must first renew ownership, read the source digest and policy revision, compare the existing part record with the upload session, and either continue from a verified ledger state or abort and prepare a new session; retrying blindly can duplicate work, while completing from a partial in-memory list can create an artifact that has no defensible retention deadline, and both outcomes look deceptively healthy until a restore or deletion review much later.

Here is a small Go transport example for a Node.js-owned backup workflow. It intentionally uses a generic S3-compatible endpoint and standard multipart calls; the surrounding service can expose the same operation through a Node.js queue, while this example makes the retry and response rules explicit. The endpoint and credentials are configuration, not article-level assumptions.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

type Part struct {
	Number int
	ETag   string
}

func request(ctx context.Context, method, endpoint, uploadID string, part *Part, body io.Reader) error {
	req, err := http.NewRequestWithContext(ctx, method, endpoint, body)
	if err != nil {
		return err
	}
	if uploadID != "" {
		query := req.URL.Query()
		query.Set("uploadId", uploadID)
		if part != nil {
			query.Set("partNumber", strconv.Itoa(part.Number))
		}
		req.URL.RawQuery = query.Encode()
	}

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		return fmt.Errorf("multipart %s returned %s", method, resp.Status)
	}
	return nil
}

func uploadWithRetry(ctx context.Context, endpoint, uploadID string, part Part, body io.Reader) error {
	for attempt := 0; attempt < 4; attempt++ {
		err := request(ctx, http.MethodPut, endpoint, uploadID, &part, body)
		if err == nil {
			return nil
		}
		if attempt == 3 {
			return err
		}
		select {
		case <-time.After(time.Duration(1<<attempt) * time.Second):
		case <-ctx.Done():
			return ctx.Err()
		}
	}
	return fmt.Errorf("unreachable retry state")
}

func main() {
	ctx := context.Background()
	endpoint := os.Getenv("OBJECT_STORAGE_ENDPOINT")
	uploadID := os.Getenv("MULTIPART_UPLOAD_ID")
	if endpoint == "" || uploadID == "" {
		panic("storage endpoint and upload ID are required")
	}
	part := Part{Number: 1}
	if err := uploadWithRetry(ctx, endpoint, uploadID, part, os.Stdin); err != nil {
		panic(err)
	}
}
```

This is transport code, not a complete protocol implementation: a production worker must use a seekable source or reopen the part after a retry, capture the response checksum, and send the ledger's complete-part list to the completion operation. The retry loop above must not reuse a consumed stream. That is the sort of detail that passes a unit test with a tiny buffer and fails at backup scale.

Resume is a decision.

For the Node.js implementation, the same rules apply around the SDK or HTTP client: use a file handle rather than buffering the archive, cap simultaneous part reads, and make the job lease expire only after the worker has stopped renewing it. Keep retry budgets inside the backup window. A rate-limit response deserves backoff; a policy mismatch deserves a state transition and operator review, not another upload attempt.

## How should restore and deletion tests evaluate the policy?

Before completion, compare the ledger with the immutable source manifest: expected parts exist, no part number is duplicated, every recorded checksum came from a successful response, and the intended object key is unchanged. After completion, issue a metadata read and compare size and digest information that the storage system makes available. Then mark the artifact ready in the ledger. The order matters because a successful assembly call is not the same as a verified restore artifact.

Before deletion, evaluate the stored policy revision against the current legal or operational hold. If a hold exists, move the record back to a retained state and preserve the reason. If the deadline is valid, delete the object and record the result; a retry must be idempotent at the ledger level. A deletion timeout should leave the record delete-pending until a later verification pass establishes the actual state.

Abort is the rollback for an unsafe in-progress upload. Use it when the source digest changed, the policy revision cannot be resolved, the lease was lost, or the worker cannot establish that the recorded parts belong to this archive. Do not “complete what is available” merely to clear storage. Incomplete multipart fragments also need an explicit cleanup policy; a bucket lifecycle rule should be treated as a backstop, not as proof that the application has made a retention decision.

Watch the signals that map to recovery objectives: upload age, bytes pending, part retry count, completion latency, abort count, delete-pending age, and restore verification rate. Set an SLO around the time from backup preparation to a verified ready artifact, then set a separate objective for deletion completion. One blended “backup succeeded” metric hides the exact failure that matters during a recovery exercise.

## When should a team migrate the retention control plane?

The storage choice should follow the boundary you are prepared to operate. Building the ledger and worker gives the platform team explicit policy history, testable state transitions, and a clear place to enforce holds. It also creates on-call work: lease recovery, multipart cleanup, checksum verification, restore drills, and schema changes are now yours.

Buying a managed object-storage service reduces the amount of storage infrastructure to run, but it does not transfer responsibility for application retention semantics. Confirm multipart behavior, minimum part constraints, lifecycle cleanup timing, metadata and checksum support, region placement, deletion guarantees, audit export, and the shape of any object-lock or WORM control before committing. A familiar S3-compatible API can reduce client changes while still leaving meaningful differences in policy controls.

| Decision | Build around a generic S3-compatible interface | Use a provider-specific control plane |
| --- | --- | --- |
| Retention logic | Application ledger makes policy revision and holds visible | Native controls may reduce custom deletion code |
| On-call load | Team owns workers, cleanup, and restore verification | Team still owns policy evidence and restore tests |
| Portability | More portable request and object model | Deeper controls can increase migration cost |
| Capacity planning | Size worker concurrency, disk staging, and retry budget | Size quotas, request rates, and egress for recovery peaks |

The catch is that this approach is not suitable when the workload needs provider-native immutability, legal holds, automatic cross-region replication, or a browser-first upload product and those controls are non-negotiable. Choose a service with those guarantees, or place a dedicated compliance layer in front of storage. Your mileage may vary on checksum and retention semantics across S3-compatible systems; resolve that uncertainty with a restore-and-delete test against the exact endpoint, not with an API compatibility claim.

Make the decision from the recovery exercise. If the team cannot explain which policy revision deleted an artifact, the missing feature is not a faster upload; it is a control-plane record.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://docs.aws.amazon.com/AmazonS3/latest/API/API_CompleteMultipartUpload.html
- https://docs.aws.amazon.com/AmazonS3/latest/API/API_AbortMultipartUpload.html
- https://developers.cloudflare.com/r2/
- https://developers.cloudflare.com/workers/
