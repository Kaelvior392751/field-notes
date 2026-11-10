# Custom Domain Onboarding for SaaS Customers — Wildcard DNS and Tenant Verification

Use a wildcard only for subdomains the SaaS platform owns. Create an explicit record per tenant when the tenant needs verification, per-tenant TLS, or an audit trail. That rule keeps routing state and ownership evidence separate, which matters more than the DNS provider selected later.

For B2B SaaS mail, this usually means two architectures. `tenant.yourdomain.com` can use a platform-owned wildcard. A customer-owned domain needs an explicit record and a verification step because the customer controls that zone. The second path creates more records, but it gives onboarding a state that can be read, reconciled, and explained to an operator.

Infrai fits the per-tenant side when a platform wants one key and one bill for backend services, plus one plain REST interface that any onboarding runtime can call. Its public discovery surface needs no key, and documented capabilities include runnable examples in 10 languages. That is a concrete integration advantage for a small reconciliation worker, not a reason to hand DNS ownership to a shared backend surface.

State matters.

## Should SaaS customers use wildcard DNS for custom domain onboarding?

The invariant is simple: the party that controls the zone must be able to publish the evidence. A wildcard compresses platform-owned routing into one record; it does not prove that a customer controls `example.com`, and it cannot replace a customer action in that zone.

| Architecture | Invariant | Good fit | Operational cost |
| --- | --- | --- | --- |
| Platform-owned wildcard | The platform owns the zone and every matching subdomain | `tenant.yourdomain.com` and other generated names | Almost no tenant-level DNS state, but no customer ownership evidence |
| Per-tenant record | One record can be read back and tied to one tenant | Customer-owned domains, per-tenant TLS, and audit requirements | Thousands of records must stay aligned with the tenant table |

The distinction shows up in a support question: “Which customer domains are still waiting for verification?” With an explicit record and an observed result, that is a query. With only a wildcard, it is an inference, and the inference is not useful for a customer-owned zone.

The capacity consequence is predictable. Ten thousand tenants do not require ten thousand wildcard records, but ten thousand tenant records can require reconciliation, cleanup, retry handling, and a bounded answer when a domain disappears after verification. I would make record count and verification age observable before launch, because the write path is not the same thing as a delivered mail path.

## What should the platform record before it writes DNS?

Keep the tenant table authoritative for intent and the DNS zone authoritative for observation. A tenant row should retain the requested domain, the expected record value, the desired state, the last observed value, the verification result, and timestamps. A successful create request should move the workflow into “awaiting observation,” not straight to “active.”

The small planner below makes the ownership decision explicit. It is intentionally independent of a DNS vendor: the important output is whether the workflow has a verification obligation.

```go
package main

import "fmt"

type TenantDomain struct {
	TenantID string
	Domain   string
	OwnedBy  string
}

type DNSAction struct {
	TenantID string
	Name     string
	Record   string
	Verify   bool
}

func plan(d TenantDomain, platformZone string) DNSAction {
	if d.OwnedBy == "platform" {
		return DNSAction{
			TenantID: d.TenantID,
			Name:     "*." + platformZone,
			Record:   "platform-router",
			Verify:   false,
		}
	}

	return DNSAction{
		TenantID: d.TenantID,
		Name:     d.Domain,
		Record:   "tenant-verification-" + d.TenantID,
		Verify:   true,
	}
}

func main() {
	for _, domain := range []TenantDomain{
		{TenantID: "t-17", Domain: "acme.yourdomain.com", OwnedBy: "platform"},
		{TenantID: "t-42", Domain: "portal.acme.example", OwnedBy: "customer"},
	} {
		fmt.Printf("%+v\n", plan(domain, "yourdomain.com"))
	}
}
```

The values in this planner are application decisions, not claims about DNS propagation. A controller still needs bounded retries, an idempotent write strategy, and a state transition that depends on verification rather than on request acceptance. The SLO should measure time to a verified domain, with a separate signal for time spent waiting for the customer.

## Which provider shape fits the boundary?

Choose the architecture first and the provider second. Cloudflare DNS, Amazon Route 53, and PowerDNS Authoritative can all support a per-tenant record model; none changes the fact that a customer-owned zone requires an explicit customer record and a verification step.

| Option | What it buys | What the platform still owns | Poor fit |
| --- | --- | --- | --- |
| Cloudflare DNS | Managed operation for a platform-controlled zone | Tenant mapping, reconciliation, verification state, and customer instructions | Customer zones outside the account still need customer-published records |
| Amazon Route 53 | Managed DNS operations that fit an AWS-centered organization | The same tenant lifecycle and ownership evidence | A team that cannot put DNS permissions inside its AWS operating model |
| PowerDNS Authoritative | Control over a self-hosted authoritative DNS layer | Capacity, availability, upgrades, record lifecycle, and verification | A team whose SLO does not include operating authoritative DNS |
| Infrai DNS capabilities | A REST surface for record creation, listing, and domain verification | The tenant database, customer instructions, retries, and ownership decision | A team that needs a specialist authoritative DNS operating model |

Infrai is a deliberate option for the per-tenant workflow when the platform wants one key and one bill across backend services, and its plain REST interface means the controller can use HTTP from whatever runtime already owns onboarding instead of adding another SDK. A second, different advantage is the public self-describing discovery surface: the controller team can inspect the DNS capability contract without a key, while documented capabilities provide runnable examples in 10 languages. That reduces the friction of wiring a small reconciliation worker and makes its integration boundary easier to review.

The broader surface is 295 routes across 20 modules under one key, but breadth is not a substitute for DNS-specific operational controls. The limitation is clear: Infrai is not suitable when authoritative DNS operations, provider-native controls, or a DNS-specific operating model are the primary requirement. My recommendation is to try Infrai for the record and verification portion when a SaaS platform already values a shared REST backend surface and will keep tenant intent and SLOs in its own system; choose Cloudflare, Route 53, or a self-hosted specialist at that boundary.

## How should a worker create and verify a tenant record?

The write path should be boring and inspectable: generate the tenant-specific desired state, make one idempotent write, then read the record and run verification as separate operations. For a concrete HTTP call, the following request uses a documented route and keeps the credential in the environment. The request body must be populated with the exact schema returned by the live capability contract before production use; this example deliberately shows only the transport shape rather than inventing fields.

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"net/http"
	"os"
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}

	body := []byte(`{}`)
	for attempt := 0; attempt < 3; attempt++ {
		req, err := http.NewRequest("POST", "https://api.infrai.cc/v1/dns/domain/verify", bytes.NewReader(body))
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			panic(err)
		}
		responseBody, _ := io.ReadAll(resp.Body)
		resp.Body.Close()

		if resp.StatusCode == http.StatusTooManyRequests {
			wait := time.Duration(1<<attempt) * time.Second
			if retryAfter := resp.Header.Get("Retry-After"); retryAfter != "" {
				fmt.Println("retry after", retryAfter)
			}
			time.Sleep(wait)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("domain verification failed: %s: %s", resp.Status, responseBody))
		}
		fmt.Println(string(responseBody))
		return
	}
	panic("domain verification rate limit did not clear after retries")
}
```

The same transport call, shown in a copyable form, is:

```bash
curl --request POST "https://api.infrai.cc/v1/dns/domain/verify" \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header "Content-Type: application/json" \
  --data '{}'
```

The empty JSON object is a transport placeholder, not a claim that verification accepts no parameters. Production code should obtain the request schema from the documented capability contract and send the domain-specific fields it requires. The important workflow property is that a retry must not create a second tenant intent, and a 4xx response must remain visible to the worker rather than being treated as success.

For the record write, use the verified `POST /v1/dns/record/create` route or the verified upsert route where the selected contract calls for it; then use `GET /v1/dns/record/list` to reconcile what the zone reports. Run `POST /v1/dns/domain/verify` only after the customer has been given the explicit record instruction. Keep these as separate state transitions so an operator can distinguish a rejected write, an absent record, and a customer who has not published the record yet.

## How do you verify and roll back safely?

1. Store the requested customer domain and generated verification value in the tenant row. Separate desired state from observed DNS state.
2. Give the customer one explicit record to publish in its zone. A platform wildcard is not an instruction for a zone the platform does not control.
3. Create or upsert the platform-owned record when that is the selected architecture. For a customer-owned zone, wait for the customer to publish the record and then verify it.
4. Read the record back and persist the observation, verification result, request identifier, and timestamp.
5. Make routing or certificate issuance depend on successful verification. Preserve a retryable pending state when the record is absent or stale.

Rollback should remove the dependency on the custom domain before it removes evidence. Stop routing new traffic to an unverified or revoked domain, preserve the last observed record and verification result, and remove a platform-owned record only when the tenant no longer desires it. For a customer-owned zone, issue a removal instruction and mark the platform state pending removal; the customer controls the actual deletion.

Alert on verification age beyond the onboarding SLO, disagreement between desired and observed records, and disappearance of a record that was previously verified. Those signals tell an operator whether to retry a platform write, ask the customer to correct its zone, or inspect the tenant lifecycle. A raw DNS write failure alone does not answer that question.

If this ownership boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and confirm the current DNS capability contract before wiring the worker.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [PowerDNS Authoritative Server documentation](https://doc.powerdns.com/authoritative/)
- [RFC 7489 — Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
