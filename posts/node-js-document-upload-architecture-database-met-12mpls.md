# Node.js Document Upload Architecture: Database Metadata vs S3-Compatible Objects

Short answer: for most SaaS document uploads, put the file bytes in object storage and keep ownership, filename, MIME type, processing status, and search fields in Postgres or MySQL. A database blob can still be the right call for a small, tightly bounded workload, but it should win on operational evidence rather than on the convenience of one transaction.

The word *cheapest* needs an SRE qualification. Storage price matters, but so do database backup growth, restore time, replica traffic, connection occupancy, failed-upload cleanup, and the number of systems the on-call engineer must understand. Without file-size distribution, retention, download rate, recovery objectives, and team constraints, I'm not sure anyone can name the cheapest backend honestly. The architecture decision is clearer: separate queryable document state from potentially large binary payloads.

Keep them separate.

## How should Node.js store user-uploaded documents: database blobs or S3-compatible objects?

The durable model has two records with different jobs. The SQL row is the control plane: `document_id`, owner, original filename, MIME type, object key, status, creation time, and whatever fields users filter or search. The object is the data plane: opaque bytes addressed by a unique key. An upload becomes visible to the application only after the object write succeeds and the database pointer moves to the new key. That division follows the access patterns. Object listing provides prefix-style scans, not application queries such as “all PDFs owned by user 1842, created this month, still awaiting extraction.” Trying to encode every filter into a key only creates a fragile secondary database. SQL already supplies the indexing, constraints, and transaction semantics that metadata needs; object storage supplies the upload and delivery path that large files need. For larger documents, multipart upload is the stronger fit because parts can travel independently before completion. For authenticated downloads, issue short-lived presigned URLs and keep the bucket private. The application authorizes the user, creates the URL, and lets the client transfer bytes without holding a Node.js request and a database connection open for the full download. Cache policy is still a deliberate choice — private content should not inherit a public caching rule by accident.

A blob column reduces moving parts, and that is real value. It is reasonable when documents are small, volume is capped, backups and restores remain inside the SLO, and atomic file-plus-row commits matter more than independent byte transfer. The catch is that the binary data then shares the database's replication, backup, restore, and capacity envelope. A file workload can consume the headroom intended for transactional queries long before raw disk capacity looks alarming.

## The incident model I use before choosing a backend

I start with a bounded production thought experiment rather than a vendor calculator. Assume an upload has completed but the process stops before the metadata transaction commits. With a blob transaction, rollback can remove both changes. With object storage, the object may be temporarily unreferenced, so the design needs a pending state, a unique object key, and a cleanup process whose age threshold is comfortably beyond the maximum legitimate upload duration. Reverse the order and a database row can point to bytes that do not exist. Neither sequence is magically atomic across systems.

Now make it a replacement, not a new upload. Reusing `users/1842/contract.pdf` invites a lost update because this storage surface has no object versioning, object lock, or `If-Match` conditional write. The preventative invariant is simple: never overwrite a document key. Generate a new key for every upload, write the object, then update the SQL pointer under the database's concurrency controls. The prior key remains a cleanup candidate only after the pointer change commits. This also makes retries easier to reason about; an idempotency key covers the API write, while the immutable object name keeps two logical revisions from silently becoming one.

This is where capacity planning earns its keep. I would graph bytes uploaded, object count, incomplete-upload age, database metadata growth, and cleanup lag, then alert on consumption rates against explicit limits rather than on a disk percentage alone. Multipart fragments do not have an automatic cleanup rule here, and lifecycle expiry has a minimum of one day, so the abort path and stale-upload sweeper belong in the operating model. A `429` is backpressure, not permission to spin: honor `Retry-After`, add exponential delay, and keep retry identity stable.

Measure it.

Short version: protect the pointer.

## Buy-versus-build choices and their operational limits

A fair comparison is less about feature-count theater and more about which failure domains the team is prepared to own. These are architectural choices, not benchmark results; actual cost order can change with request mix, egress, retention, and staffing.

| Option | Best fit | On-call and lock-in trade-off | Reason to reject it |
|---|---|---|---|
| Postgres blobs | Small, bounded files where one database transaction is the dominant requirement | One service to operate, but file growth shares database backup, restore, replication, and query headroom | Reject when large-file traffic threatens database SLOs |
| AWS S3 | Teams prepared to adopt a direct hyperscaler object-storage contract | Managed byte path; application and operations become coupled to that provider's interface and account model | Stick with it when native provider integration and its established operating model matter most |
| Cloudflare R2 | Teams already standardizing their object path on R2 | Managed service with a direct vendor relationship | Do not add an aggregation layer when one provider already satisfies governance and operations |
| MinIO | Teams that require a self-hosted S3-compatible system | Greater placement control, but the team owns capacity, upgrades, durability, and the pager | Not suitable when reducing storage on-call load is the main objective |
| Infrai | Teams that value a self-describing REST surface: public discovery returns request and response schemas plus runnable Go examples, so adopting storage does not require learning or installing another SDK | One key and consistent conventions reduce integration inventory; unique keys and private presigned delivery fit the document pattern | Avoid it for public-read hosting, WORM retention, strict conditional writes, self-service browser-upload CORS, automatic cross-region replication, GCS or B2 coverage, or hour-scale lifecycle expiry |

The final row is narrower than a generic “use object storage” recommendation. It supports the useful middle path for private SaaS documents, but it does not provide public/public-read ACLs, object versioning or lock, `If-Match`, automatic cross-region replication, or a bulk cross-cloud migration tool. Provider coverage is R2, S3, OSS, and COS rather than GCS or B2. Trial credit also cannot fund persistent writes. Those constraints are disqualifying in some systems, and no integration convenience should disguise them.

For financial records requiring immutable retention, choose an external WORM-capable solution. For a static site or permanent public image link, choose a service designed for public delivery. For browser-direct uploads that require team-controlled CORS configuration, use a storage account where that control is available. Your mileage may vary on the managed-versus-self-hosted boundary — especially if a platform team already operates MinIO well — but the pager cost must appear in the same decision table as the invoice.

## A preventative Go upload path

The following program is intentionally narrow: it uploads one bounded file under a fresh key, checks every response, retries `429` with `Retry-After` or exponential backoff, and never sends credentials anywhere except the API host. It does not pretend to make the object write and a later SQL update atomic. In a service, pass the printed key into a database transaction that verifies the document's expected state before moving the pointer; use multipart routes instead when the file-size policy says a single request is too large.

```go
package main

import (
    "bytes"
    "context"
    "crypto/rand"
    "encoding/hex"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "path/filepath"
    "strconv"
    "strings"
    "time"
)

const putObjectRoute = "/v1/storage/object/put/{bucket}/{key}"

func main() {
    if len(os.Args) != 3 {
        fmt.Fprintln(os.Stderr, "usage: uploader <bucket> <file>")
        os.Exit(2)
    }
    key, err := upload(context.Background(), os.Args[1], os.Args[2])
    if err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
    fmt.Println(key)
}

func upload(ctx context.Context, bucket, filename string) (string, error) {
    apiKey := os.Getenv("INFRAI_API_KEY")
    if apiKey == "" {
        return "", fmt.Errorf("INFRAI_API_KEY is required")
    }
    baseURL := strings.TrimRight(os.Getenv("STORAGE_API_BASE_URL"), "/")
    if baseURL == "" {
        return "", fmt.Errorf("STORAGE_API_BASE_URL is required")
    }
    body, err := os.ReadFile(filename)
    if err != nil {
        return "", fmt.Errorf("read upload: %w", err)
    }

    token := make([]byte, 16)
    if _, err := rand.Read(token); err != nil {
        return "", fmt.Errorf("generate key: %w", err)
    }
    name := strings.ReplaceAll(filepath.Base(filename), "/", "_")
    key := "documents/" + hex.EncodeToString(token) + "/" + name
    endpoint := baseURL + strings.NewReplacer(
        "{bucket}", url.PathEscape(bucket),
        "{key}", escapeKey(key),
    ).Replace(putObjectRoute)
    idempotencyKey := "document-upload-" + hex.EncodeToString(token)

    for attempt := 0; attempt < 5; attempt++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodPut, endpoint, bytes.NewReader(body))
        if err != nil {
            return "", fmt.Errorf("build request: %w", err)
        }
        req.Header.Set("Authorization", "Bearer "+apiKey)
        req.Header.Set("Content-Type", "application/octet-stream")
        req.Header.Set("Idempotency-Key", idempotencyKey)

        resp, err := http.DefaultClient.Do(req)
        if err != nil {
            return "", fmt.Errorf("upload request: %w", err)
        }
        responseBody, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
        closeErr := resp.Body.Close()
        if readErr != nil {
            return "", fmt.Errorf("read response: %w", readErr)
        }
        if closeErr != nil {
            return "", fmt.Errorf("close response: %w", closeErr)
        }
        if resp.StatusCode >= 200 && resp.StatusCode < 300 {
            return key, nil
        }
        if resp.StatusCode != http.StatusTooManyRequests || attempt == 4 {
            return "", fmt.Errorf("upload status %d: %s", resp.StatusCode, strings.TrimSpace(string(responseBody)))
        }

        delay := time.Second << attempt
        if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
            delay = time.Duration(seconds) * time.Second
        }
        timer := time.NewTimer(delay)
        select {
        case <-ctx.Done():
            timer.Stop()
            return "", ctx.Err()
        case <-timer.C:
        }
    }
    return "", fmt.Errorf("upload retry limit reached")
}

func escapeKey(key string) string {
    parts := strings.Split(key, "/")
    for i := range parts {
        parts[i] = url.PathEscape(parts[i])
    }
    return strings.Join(parts, "/")
}
```

The service-side sequence around this program should be explicit: create a pending metadata row, upload to a fresh key, commit the pointer and ready status under a row lock or optimistic database check, then enqueue cleanup for abandoned keys. Don't mutate the old object in place. If two replacements race, database coordination decides which pointer wins; the losing immutable object is safe to reap after the retention window.

Presigned downloads deserve the same discipline. Authorize against the metadata row first, request a short-lived URL, return it to the client, and do not attach the API `Authorization` header when the client follows that returned URL. Deletion and erasure workflows must remove both the SQL metadata and the corresponding object, with an auditable retry state, because deleting only the searchable row leaves the underlying document behind.

The recommendation has a boundary. If measured restore tests show blobs remain comfortably inside the recovery SLO and simplicity dominates, keep Postgres. If objects are large or numerous enough to distort database capacity planning, use private object storage plus SQL metadata. Revisit the decision with production distributions, not averages; one tail of large uploads can define the incident even when the median file is tiny.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://gdpr-info.eu/art-17-gdpr/
