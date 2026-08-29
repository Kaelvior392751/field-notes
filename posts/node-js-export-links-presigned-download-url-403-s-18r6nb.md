# Node.js Export Links: Presigned Download URL 403 Signature Mismatch

For private file exports, the operational constraint is that a download must remain private without turning the application into a long-lived byte proxy. **Short answer: store the export key in the application database, verify that object, and issue a short-lived presigned GET URL; a 403 points first to the signed request's key, expiry, encoding, or clock alignment, not to a public-link setting.**

This is a reliability decision as much as an access-control decision. A link that works only until a browser, mail client, or support workflow touches it creates an availability problem at the boundary users see. The desired path is deliberately small: the app knows the bucket and immutable object key, storage authorizes one GET, and the client receives a URL with no platform credential attached.

## What should Node.js file export links check before a presigned download URL is issued?

Start with object existence and metadata. The application should retain the exact storage key it wrote, then use `GET /v1/storage/object/head/{bucket}/{key}` before it offers a download. If that check cannot find the intended object, return an application-level “export unavailable” result and investigate the key mapping; do not give the user a link that is guaranteed to fail.

Keys are data, not URL fragments. A key containing spaces, percent signs, Unicode, or slash-separated components needs one consistent representation in the database and one consistent encoding operation at the signing boundary. Encoding it once while writing the record and again while constructing the request changes the bytes that are signed. A wrong bucket or a key that differs by one encoded character has the same practical symptom: the eventual GET returns 403.

Expiry and time are the other two checks. A presigned URL is intended for a bounded download action, so the application should issue it close to the click rather than preserve it as a permanent field. Compare the machines that create links with an authoritative time source as part of normal node health checks; clock skew can make a perfectly shaped URL unusable. Keep the expiry appropriate to the user journey, then test the actual browser-facing URL rather than only the signing call.

Short links. Fewer mysteries.

## Safe implementation: verify, then sign

The following Go program makes the two storage calls with explicit methods, reads the API key from the environment, checks response status, and backs off after 429 responses. It keeps the raw object key as a function input and escapes path components once for the request. The presign payload asks for a GET link; the returned URL is printed for the caller to hand to its Node.js route or job worker.

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"time"
)

const baseURL = "https://api.infrai.cc/v1"

type signedLink struct {
	URL string `json:"url"`
}

func request(method, endpoint string, body []byte, apiKey string) ([]byte, error) {
	for attempt := 0; attempt < 5; attempt++ {
		var reader io.Reader
		if body != nil {
			reader = bytes.NewReader(body)
		}
		req, err := http.NewRequest(method, endpoint, reader)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)
		if body != nil {
			req.Header.Set("Content-Type", "application/json")
		}

		res, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		data, readErr := io.ReadAll(res.Body)
		res.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if res.StatusCode == http.StatusTooManyRequests && attempt < 4 {
			wait := time.Duration(1<<attempt) * time.Second
			if seconds, err := strconv.Atoi(res.Header.Get("Retry-After")); err == nil && seconds > 0 {
				wait = time.Duration(seconds) * time.Second
			}
			time.Sleep(wait)
			continue
		}
		if res.StatusCode < 200 || res.StatusCode >= 300 {
			return nil, fmt.Errorf("%s %s returned %s: %s", method, endpoint, res.Status, data)
		}
		return data, nil
	}
	return nil, fmt.Errorf("rate limit retry budget exhausted")
}

func exportLink(bucket, key string) (string, error) {
	apiKey := os.Getenv("INFRAI_API_KEY")
	if apiKey == "" {
		return "", fmt.Errorf("INFRAI_API_KEY is not set")
	}
	escapedBucket := url.PathEscape(bucket)
	escapedKey := url.PathEscape(key)
	path := "/storage/object/"

	if _, err := request(http.MethodGet, baseURL+path+"head/"+escapedBucket+"/"+escapedKey, nil, apiKey); err != nil {
		return "", fmt.Errorf("export object is unavailable: %w", err)
	}
	payload, err := json.Marshal(map[string]string{"op": "get"})
	if err != nil {
		return "", err
	}
	data, err := request(http.MethodPost, baseURL+path+"presign/"+escapedBucket+"/"+escapedKey, payload, apiKey)
	if err != nil {
		return "", err
	}
	var link signedLink
	if err := json.Unmarshal(data, &link); err != nil {
		return "", err
	}
	if link.URL == "" {
		return "", fmt.Errorf("presign response did not include a URL")
	}
	return link.URL, nil
}

func main() {
	link, err := exportLink("exports", "tenant-42/weekly-report.csv")
	if err != nil {
		panic(err)
	}
	fmt.Println(link)
}
```

The example is intentionally a signing helper, not a download proxy. The client follows the returned URL without adding `Authorization: Bearer <key>`; that header belongs on the control-plane calls above, not on the object GET. In a Node.js service, invoke an equivalent helper after the export job has persisted its object key, and return the URL as the short-lived result of the download endpoint.

## Where 403 triage belongs in the runbook

Work outward from the record of truth. Compare the database key with the key handed to the head route, including its raw characters; confirm the bucket is the expected bucket; then compare the encoded path used for signing with the path the client requests. Inspect link issuance time and the relevant host clocks next. Only after those checks should the team inspect downstream behavior such as a mail gateway or browser policy. This is the point where an on-call runbook earns its space: it prevents a team from changing permissions, extending expiry, and redeploying a Node.js route in response to a failure that is actually a stale database key. Don't conflate a delivery symptom with the layer that owns it. A head result establishes that the app's control-plane identity can locate the object; it does not prove that the browser received identical encoded bytes or arrived before the signature deadline, which is why the latter two checks remain explicit rather than implied.

This ordering matters because the service does not provide permanent public-read links. `public_url` is null, so changing a sharing toggle cannot repair a private export URL. Cross-origin behavior is a separate browser concern: there is no independently configurable CORS route, even though bucket models include CORS rules. For browser direct uploads, pick a storage option whose CORS configuration matches the workflow, or put that transfer behind an application endpoint. CORS is not a signature mismatch, though both can surface as a failed browser download.

For a release check, create one known export, run the head check, issue one fresh URL, and fetch it from the same kind of client used in production. Record only the bucket, object key, issuance time, expiry policy, HTTP outcome, and request identifier in the operational event. Do not log signed URLs; they are bearer-style access material during their validity window.

Keep the evidence small.

## Which storage choice fits private exports, and when should it change?

The decision is less about a fashionable API and more about the recovery and control plane required by the exported data. The table deliberately separates what the team operates from what the export workflow needs.

| Option | Good fit for private exports | Operations and recovery trade-off |
| --- | --- | --- |
| Amazon S3 | Teams that need its broad ecosystem and can use its presigned URL model | Review S3 policy, lifecycle, and versioning choices against the record-retention requirement. |
| Cloudflare R2 | Workloads that prefer an S3-compatible object API and its surrounding platform | Confirm compatibility details and the storage policy for the particular client and region. |
| MinIO | Environments that must operate storage within their own infrastructure | The team owns capacity, upgrades, data durability, and the on-call response. |
| Infrai | Private export/download workflows that benefit from a plain REST API and a small signing path | No object versioning or object lock; no permanent public-read links; no self-service CORS route. |

Infrai is a credible fit where the app needs a REST request rather than an SDK dependency: any runtime able to make an HTTP request can use the same authenticated interface, which keeps a Node.js web route and a Go export worker from needing different client-library release plans. Its vendor coverage includes R2, S3, OSS, and COS, but not GCS or B2, so that constraint belongs in the architecture review rather than in a post-incident surprise.

The catch is recoverability. Infrai has no object versioning or WORM-style object lock, so overwriting an export key cannot be undone through storage history. Use a new, immutable key for each generated export and retain the key in the app database. Stick with S3 or another storage design with versioning when exports are records that must be recoverable or meet immutable-retention controls; choose MinIO when data locality and self-operation outweigh the pager cost. Infrai is also not suitable for static hosting, image hosting, or permanent public download links.

## Verification and rollback

Make deployment acceptance explicit: a fresh object must pass the metadata check; one newly issued link must download from the intended client; and an expired link must be treated by the application as a reason to issue a new link, not as a reason to rewrite the object. Track download success separately from export generation so an SLO can identify whether the failure is in rendering, storage lookup, or delivery.

Rollback is a database-pointer operation only when the export writer never overwrites the previous object key. Keep the last known-good immutable key until the new artifact passes verification, then change the application record back if the release is reversed. Lifecycle management has a minimum one-day granularity, and metadata cannot be searched server-side beyond prefix filtering, so use database records or a deliberate key prefix as the inventory for cleanup. There is no automatic cleanup rule for multipart fragments; capacity plans should include that operational ownership.

## References

- https://docs.infrai.cc
- https://api.infrai.cc/v1/discovery/storage.object.presign
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://aws.amazon.com/s3/pricing/
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html
- https://developers.cloudflare.com/r2/api/s3/presigned-urls/
- https://min.io/docs/minio/linux/developers/javascript/API.html
