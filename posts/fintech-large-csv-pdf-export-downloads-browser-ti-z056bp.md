# Fintech Large CSV/PDF Export Downloads: Browser Timeout and Storage Throughput

Short answer: for private CSV and PDF exports, make the completed object the source of truth, send the browser a time-bounded signed URL, and design retries around byte ranges and a measured large-file throughput budget. Keep the application in the control plane; do not make Node.js carry every byte unless the product requirement genuinely needs an application proxy.

This is a delivery problem, not one timeout. An export worker can finish successfully while a browser later fails because the file is large, the connection is slow, the link expires during transfer, or a retry starts at byte zero. In a fintech system, that distinction matters: a user may be downloading a regulated report, while the platform team is looking at an export-job metric that says everything is healthy.

## The incident lesson: “ready” is not the same as “downloadable”

I use a bounded scenario for this design: a private user requests a large CSV or PDF, a background worker writes it to object storage, and the UI offers an expiring download link. The invariant is simple. The link is not visible until the upload is complete and the stored object has been checked for the expected size and content type.

That check prevents a particularly expensive class of support ticket. The job page can say “ready” only after the object is available for a read, its size matches the export record, and its media type is the one the download response is meant to describe. The filename should also be intentional. `Content-Disposition` controls whether a response is treated as an attachment and provides filename behavior that browsers understand; the MDN reference is the useful starting point for the header details.

The second invariant is operational: export completion and download completion are separate SLOs. Track the time from request to object-ready independently from the time to deliver the object. Record object size, link issuance time, expiry time, first-byte time, completion time, response status, and the number of refreshes or resumed ranges. A single “download timeout” counter cannot tell an on-call engineer whether the pressure is in export capacity, network delivery, authorization lifetime, or the browser path.

Short path.

The failure I want to catch is a valid object behind an invalid delivery window. If a download begins near expiry, a slow connection can lose authorization before the final byte arrives. The recovery action is to mint a fresh link for the same completed object, then resume only when the HTTP response proves that the requested range is being honored.

## Why do large CSV/PDF export downloads time out when browser retries meet object storage?

The user sees one symptom, but several boundaries are involved. A large file makes every fixed timeout more suspicious: the relevant budget is object size divided by the lowest throughput the product promises to support, with room for connection setup and a retry. There is no universal expiry value. It should come from observed delivery behavior, a stated download SLO, and the largest export allowed by the product.

Presigned URLs are bearer credentials, so their lifetime is part of the access design, not merely a transport setting. A link that has not expired but fails immediately points to a connection, browser, or intermediary problem. A link requested long after expiry points to a refresh-flow problem. A response that returns a different size from the export record points to publication validation. These cases need different owners and different alerts.

Range requests add another contract. After writing bytes through offset `N`, a client may ask for `bytes=N-`. A successful partial response is status `206`, and the client must verify that the response describes the requested range before appending. A server may instead return the complete object with status `200`; that is a replacement download, so the client must truncate the destination rather than append it. A range that cannot be satisfied is not permission to guess: surface the failure, refresh the metadata or URL, and make a deliberate restart decision.

Do not hide this behind a Node.js proxy by default. A proxy can consume application bandwidth, buffer large bodies, and lose range-related headers unless its forwarding behavior is carefully specified. Direct object delivery removes those bytes from the application data path while leaving the application responsible for authorization, object readiness, audit events, and a current URL. That separation is usually the cleaner capacity boundary.

## What should the export pipeline validate before a browser download?

The pipeline needs a small state machine rather than a single boolean:

1. `requested`: the user has asked for an export and the request is authorized.
2. `writing`: the worker is producing the CSV or PDF and uploading it.
3. `ready`: the upload is complete, the object metadata has been checked, and the object key is recorded.
4. `link_issued`: a current signed URL and expiry are associated with that object.
5. `delivering`: the browser has started a read; progress and range attempts are observable.
6. `complete` or `retryable`: the client has verified the final byte count, or a new link and resume policy are required.

The UI should not manufacture a retry by replaying the original URL. It should ask the application for a current link tied to the same immutable export object. That keeps authorization bounded and avoids turning an expired private link into a public one. It also gives the audit trail a clear event: this user received access to this object during this window.

Here is the core of a range-aware reader. It is deliberately a generic HTTP client, because the important contract is the response status and byte position, not a storage SDK. Production code should add cancellation, a bounded retry policy for transport failures and `429`, integrity verification appropriate to the export format, and telemetry around each attempt.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
)

func fetchPart(url, path string, offset int64) error {
	req, err := http.NewRequest(http.MethodGet, url, nil)
	if err != nil {
		return err
	}
	if offset > 0 {
		req.Header.Set("Range", "bytes="+strconv.FormatInt(offset, 10)+"-")
	}

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK && resp.StatusCode != http.StatusPartialContent {
		return fmt.Errorf("download request returned %s", resp.Status)
	}

	flags := os.O_CREATE | os.O_WRONLY
	if resp.StatusCode == http.StatusPartialContent && offset > 0 {
		flags |= os.O_APPEND
	} else {
		flags |= os.O_TRUNC
	}

	file, err := os.OpenFile(path, flags, 0600)
	if err != nil {
		return err
	}
	defer file.Close()

	_, err = io.Copy(file, resp.Body)
	return err
}
```

The important detail is not the function name. It is the refusal to append to a `200` response when the caller expected a continuation. The caller should compare the resulting file size with the stored object size before marking the export complete. It should also avoid unbounded retries: a slow connection is not fixed by opening dozens of simultaneous reads, and a rate-limited response is a capacity signal, not a cue for a tight loop.

## How should a platform team compare build and managed delivery options?

The comparison should be made against the failure modes and the on-call budget, not a feature-count spreadsheet. The data plane, control plane, recovery behavior, and evidence needed during an incident are separate concerns.

| Approach | Useful property | Cost or boundary to accept |
| --- | --- | --- |
| Direct object download | Large bodies bypass application buffering and use standard HTTP behavior | The browser and storage endpoint must agree on range, disposition, and authorization semantics |
| Application proxy | One domain can centralize policy, logging, and response transformation | The application owns bandwidth, connection handling, buffering limits, and backpressure |
| Self-hosted object service | Storage behavior and deployment boundaries are under the team's control | The team also owns capacity planning, upgrades, replication, and recovery testing |
| Managed object storage | The storage data path and operational primitives are supplied as a service | Provider-specific policies, egress terms, and integration surface create a dependency to review |

The catch is that direct delivery is not suitable when every byte must be transformed, inspected synchronously, or delivered through an application-controlled policy boundary. In those cases, keep the proxy, but budget its egress and memory explicitly and test interrupted transfers through it. Stick with a self-hosted service when jurisdiction, deployment control, or an existing storage operations team outweighs the reduction in on-call work from a managed service. Choose the simpler boundary when the requirement is only private download of a completed export.

Price can enter this decision once, after the reliability and ownership requirements are clear: compare storage, request, and egress policies for the expected object-size distribution, including retries. A lower nominal storage rate does not answer whether a slow browser causes repeated egress or whether the team can operate the recovery path. I'm not sure any static price table will stay representative for a workload whose export sizes and retry rates are still changing; measure those distributions first.

## The runbook I would put beside the SLO

Test the whole path with a large synthetic CSV and PDF, a constrained connection, an interrupted transfer, an expired link, a retry at a nonzero offset, and a response that declines the range. Verify that a partial response appends, a full response replaces, and a mismatched final size remains visibly incomplete. Test the same cases through any proxy that the product actually uses.

During an incident, start with the object key and expected size. Then inspect readiness time, issued expiry, response status, range offset, bytes written, and whether the client asked for a fresh link. This sequence keeps the investigation grounded in evidence and makes a capacity decision possible: increase worker capacity for generation, adjust delivery limits and expiry policy, remove unnecessary proxying, or improve the browser recovery path.

Three words matter: measure the path.

The durable design is boring in the right way. Validate the completed object, issue a bounded private URL, preserve HTTP range semantics, verify the final size, and expose separate SLOs for generation and delivery. That method applies to CSV and PDF exports without tying the system to a particular provider or pretending that one timeout value can solve a throughput problem.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- https://www.rfc-editor.org/rfc/rfc9110
- https://www.backblaze.com/cloud-storage/pricing
