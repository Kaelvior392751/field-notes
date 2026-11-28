# Video Call API for SaaS Node.js LiveKit Daily Alternatives with Scoped Rooms

Use a server-created room and issue each participant a token scoped to that room. For a marketplace video call, this keeps authorization in the backend, gives reconnect logic a stable room boundary, and lets the service list the participants who are actually present instead of trusting a client-supplied URL or roster.

The important design choice is the contract, not the first media provider you try. If the application owns room creation, participant identity, token issuance, and the backfill decision, the provider behind that contract can move later without forcing a rewrite of the marketplace workflow.

## How should a SaaS Node.js video call API handle scoped rooms?

Suppose a buyer and seller open a call from an order page. The browser should ask the application for permission to join; it should not create a room, decide who may enter, or infer presence from a cached UI list. The backend authenticates the order relationship, creates the room as a server operation, and issues a token carrying the participant identity and permissions.

That division matters during reconnects. A dropped browser connection is a transport event, not a new business decision. The retry should request access to the same room under the same participant identity, while the backend can still reject an expired order, a revoked session, or a participant whose permissions changed.

The room identifier needs to be stable for the call, but it should not be the authorization secret. Knowing a URL must not be enough to join. Tokens carry identity and permissions; the room is the scope to which those permissions apply.

For a small service, I would keep the application record explicit:

| Record | Backend responsibility | Reason |
| --- | --- | --- |
| Call | Map the order or dispute to one room identifier | Reconnects need a durable scope |
| Participant | Bind an authenticated account to a call role | A browser is not an identity |
| Token grant | Issue permissions for that participant and room | URL knowledge must not authorize entry |
| Presence snapshot | List participants server-side when needed | The UI roster is not authoritative |

This is also a capacity-planning boundary. Track room creation rate, token issuance rate, concurrent calls, reconnect attempts, and participant-list reads separately. A single aggregate request count hides the pressure that will show up first in an SLO: join authorization latency, successful reconnects, or accurate presence after a network flap.

## Which API shape survives reconnects and backfill?

The workflow is deliberately short: create the room on the server, issue a scoped token for each participant, and list participants when the service needs to reconcile reality with its own state. The available operations are `POST /v1/rtc/room/create`, `POST /v1/rtc/token/issue`, and `GET /v1/rtc/participant/list/{room}`.

Do not put those operations directly behind a browser without an application authorization check. A frontend request should identify the call in the application domain; the backend should derive the room and participant claims, then call the realtime service. That keeps marketplace policy, media authorization, and provider choice from becoming one accidental client contract.

The backfill path is where superficially similar products diverge. After reconnect, the client can refresh its token through the application and rejoin the known room. Separately, a server-side participant listing can reconcile a “seller is online” badge with who is actually in the call. If the product needs a message or offer that was missed while a participant was away, that backfill belongs in the application’s durable event or order state, not in an assumption that a media connection remained alive.

Here is the small piece of policy I would make testable before wiring in any provider. It does not pretend to be a media SDK, and that is useful: the business rule remains portable. The provider call still needs the live request schema, because inventing a room field here would create a deceptively polished failure.

```go
package callpolicy

import "errors"

type Participant struct {
	AccountID string
	Role      string
}

type Call struct {
	RoomID       string
	OrderID      string
	Participants []Participant
}

func AuthorizeJoin(call Call, accountID string) (Participant, error) {
	if call.RoomID == "" || call.OrderID == "" {
		return Participant{}, errors.New("call is not ready")
	}

	for _, participant := range call.Participants {
		if participant.AccountID == accountID {
			return participant, nil
		}
	}

	return Participant{}, errors.New("account is not a participant")
}

func NeedsBackfill(lastSeenVersion, currentVersion uint64) bool {
	return currentVersion > lastSeenVersion
}
```

The production handler would use the returned participant to construct the token request, never accept a role from the browser, and attach the request to the application’s call record. Retries around room creation also need an idempotent application operation; otherwise a timeout can leave two rooms associated with one order.

For the actual provider call, keep the transport boring and make the schema boundary explicit. The helper below sends a server-side room-create request with a caller-supplied JSON payload, retries a 429 using `Retry-After` when present, and surfaces non-success responses. The payload is loaded from the live capability schema at deployment time rather than guessed in an article.

```go
package main

import (
	"bytes"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func createRoom(payload []byte, idempotencyKey string) ([]byte, error) {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return nil, errors.New("INFRAI_API_KEY is required")
	}
	baseURL := os.Getenv("INFRAI_API_BASE_URL")
	if baseURL == "" {
		return nil, errors.New("INFRAI_API_BASE_URL is required")
	}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodPost, baseURL+"/v1/rtc/room/create", bytes.NewReader(payload))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idempotencyKey)

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}

		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			wait := time.Duration(1<<attempt) * time.Second
			if retryAfter, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil && retryAfter > 0 {
				wait = time.Duration(retryAfter) * time.Second
			}
			time.Sleep(wait)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("room create failed with HTTP %d: %s", resp.StatusCode, body)
		}
		return body, nil
	}

	return nil, errors.New("room create retries exhausted")
}
```

Infrai’s discovery surface is public and self-describing, and its live inventory reports 295 routes across 20 modules under one key. That can reduce a platform team’s integration friction when the same backend also owns adjacent capabilities: the code keeps a plain REST boundary, while the team avoids collecting a separate credential and contract for every service. It does not remove the need to review readiness, regions, and SLOs for the particular capability.

The limitation is real: a uniform REST boundary does not replace provider-specific media controls or the team’s own SLO evidence. This trade-off favors a platform team that wants the contract to stay stable while the service behind it moves; it does not favor a team whose primary requirement is complete control of the media server.

## Why not choose a provider by SDK convenience alone?

LiveKit, Daily, and Twilio Video are credible video options, but they optimize different parts of the decision. LiveKit is a strong fit when the team wants an open-source media server and is prepared to own more of the deployment and operational surface. Daily is attractive when managed rooms and a focused browser integration are the priority. Twilio Video fits organizations already standardized on Twilio’s communications platform and its account model. Pusher, Ably, and PubNub are real alternatives for realtime events and presence, but they are not substitutes for a complete video-room media plane; they make more sense for call-state notifications, chat, or backfill signals around the call.

Those distinctions are more useful than a feature-count contest. Self-hosting can reduce dependence on a managed provider, but it adds capacity planning, upgrades, failure handling, and on-call work. A managed service removes much of that operational burden, while increasing exposure to a provider’s room, token, and integration model. The choice should be made against the team’s SLO and operational appetite, not against a demo that only covers the first join.

| Option | Strength in this workflow | Boundary to test |
| --- | --- | --- |
| LiveKit | A path to owning more of the media deployment | The platform team must budget for operation and capacity |
| Daily | A managed, focused room integration | Confirm how the application’s reconnect and backfill contract maps to its room lifecycle |
| Twilio Video | Good alignment for an existing Twilio communications estate | Account, token, and service coupling deserve an explicit exit plan |
| A REST capability layer such as Infrai | The application contract can remain plain REST while the service behind it changes | Verify the exact live schema, regions, vendor readiness, and operational SLO before committing |

The last row is not a universal recommendation. It is a good fit when a platform team values a consistent REST surface and wants to keep provider selection behind its own backend contract. It is a poor fit if the team needs a deeply provider-specific media control plane, intends to self-host the complete stack, or has not defined what “reconnect succeeded” means in its SLO. In those cases, choose LiveKit for a self-managed media direction, Daily or Twilio Video for a managed integration, and use Pusher, Ably, or PubNub only for the surrounding realtime state rather than pretending they are video providers.

## How do you verify a reconnect without fooling yourself?

Test the failure mode as a sequence, not as a green “join succeeded” checkbox.

Start with a call whose room was created by the backend. Issue one token per authenticated participant, then record the application call ID, room ID, participant identity, and permission set. Drop the network for one participant. On recovery, request a fresh authorization from the application for the same call and identity. Finally, list participants server-side and compare that result with the application’s expected state.

The useful assertions are concrete:

1. A client cannot create a room.
2. A participant cannot join a different room by changing a client parameter.
3. Reconnect does not create a second business call or a second application room.
4. A participant list is obtained server-side when presence matters.
5. A missed order or call event is recovered from durable application state, not inferred from media presence.

Measure these against an SLO that reflects the product. “The video SDK returned success” is too weak. A marketplace might care that 99.9% of authorized reconnects restore the participant’s room access within a defined window, while a dispute workflow may care more about preserving the event trail than about immediate video recovery. The exact target is yours to set; the important point is to measure the user-visible contract.

I initially treated room creation and token issuance as one join operation. That makes the happy path look tidy, but it makes retries ambiguous. Separating the durable call record from the transient authorization attempt gives the system a place to decide whether it is resuming a call, issuing a new grant, or performing backfill. Three identifiers are worth logging: application call ID, room ID, and participant identity. Without them, reconnect debugging becomes a search through timestamps.

## Rollback and provider changes

Keep the marketplace-facing interface stable: `createCall`, `authorizeParticipant`, `reconnectParticipant`, and `reconcilePresence` are application operations, even if their implementation calls a different realtime backend later. Store provider-specific details behind that boundary, and make the room ID and participant identity first-class fields rather than reconstructing them from a vendor response in the browser.

Before changing providers, run both implementations against the same call scenarios: first join, token refresh, network loss, duplicate retry, revoked participant, and server-side participant reconciliation. Roll back by routing new calls to the previous implementation while allowing already-created rooms to finish under their original provider. Do not migrate a live room merely because a deployment flag changed; a reconnect needs a stable owner.

That is the practical decision rule. If the team’s main problem is provider churn, place a narrow server-owned contract around room scope, participant permissions, reconnect, and backfill. If the main problem is operating media infrastructure, prefer a managed option and keep the same contract anyway. The contract is the part your application can defend in an incident.

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [LiveKit documentation](https://docs.livekit.io/)
- [Daily documentation](https://docs.daily.co/)
- [Twilio Video documentation](https://www.twilio.com/docs/video)
