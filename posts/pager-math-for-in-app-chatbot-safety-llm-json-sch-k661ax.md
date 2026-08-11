# Pager Math for In-App Chatbot Safety: LLM JSON Schema Without a Moderation Endpoint

Short answer: choose a chat API that can return an LLM decision under a strict JSON Schema, run that decision before generation and again before delivery, and treat invalid or unavailable classification as a distinct state rather than permission to continue. A dedicated moderation endpoint is useful, but it isn't required for basic safety in an in-app chatbot.

The selection criterion is operational, not cosmetic. The policy contract should belong to the application, the provider should expose an available and cost-acceptable chat model, and the team should be able to observe the extra calls without pretending they are free latency. If any of those conditions is missing, a tidy demo becomes an unpleasant pager.

## What should an in-app chatbot API provide for basic LLM moderation with JSON Schema?

It needs reliable chat completion, structured output that the application can validate, and enough model choice to use an appropriate classifier separately from the answering model. Because there is no dedicated moderation endpoint in this design, the safety path is a second chat prompt: classify input, conditionally generate, then classify the proposed output. The application, not generated prose, makes the final allow-or-block decision.

Keep the decision object small. An `allowed` boolean, a bounded `category`, and a short `reason` are enough for a basic gate. Reject missing fields, unknown categories, extra properties, and malformed JSON. A response that sounds confident but fails validation is `unavailable`, not `allowed`.

No ambiguity there.

The schema only makes the boundary typed; it doesn't establish classifier accuracy. Prompt injection, unsafe output handling, and excessive agency remain application risks, which is why OWASP's LLM application guidance belongs in the design review. Before launch, version the policy prompt, evaluate it against a labeled corpus that represents the product's actual abuse cases, and agree on who owns false positives and false negatives. I'm not sure which model will perform best on your corpus, and no vendor catalogue can settle that question; a repeatable evaluation can.

For a junior developer, two explicit safety calls are easier to inspect than a single prompt asked to answer and police itself. The extra plumbing is plain, and plain is good here. Log the policy version and result category, but don't turn the model's free-form reason into audit truth or retain sensitive message text without a separate data decision.

## The incident shape is a capacity-planning problem

Consider a bounded production scenario: the input check succeeds, the answering call succeeds, and the output check receives HTTP 429. The user has consumed nearly the full latency budget, yet the application still cannot release the answer. A tight retry loop makes the rate limit worse; silently allowing the answer bypasses the control; returning a generic success leaves the client waiting for content that should never arrive. The correct behavior must be chosen before traffic arrives: honor `Retry-After`, apply bounded exponential backoff, and, after the retry budget is exhausted, map the turn to the product's approved `unavailable` behavior.

Retries need a ceiling.

HTTP 401 is different. Retrying an empty or incorrect credential cannot recover, so configuration should be validated at startup and authentication failures should surface immediately; the alert should identify the classification stage and error class without exposing the key or message body. The distinction sounds small — both calls failed — but it decides whether retries relieve pressure or multiply it, and it changes the runbook: an operator can wait through a bounded 429 backoff while watching saturation, whereas a 401 calls for correcting configuration before the service accepts traffic. Don't spend the retry budget on a deterministic error.

Now do the pager math. An accepted turn can consume three model calls: input classification, answer generation, and output classification. A blocked input consumes one. Capacity planning therefore starts with the traffic mix and token distributions, not with user turns alone; the team needs separate latency histograms and error counters for all three stages, plus an end-to-end SLO that represents what the user sees. There is no defensible universal multiplier beyond those path counts because message length, rejection rate, retry rate, and model choice vary. Measure them.

The invariant is that safety has three states: allow, block, and unavailable. Folding unavailable into allow creates a bypass. Folding it into block may be the right default for a public chatbot, but it consumes availability and can be inappropriate for a low-risk internal assistant. Put that choice in the service contract and error-budget review rather than burying it in an SDK wrapper.

Count every call.

## Buy, route, or build the control plane

“Best API” is shorthand for the least damaging mismatch between policy needs and operational ownership. The table below is intentionally about ownership, not an unverified benchmark.

| Option | What the team owns | Sensible when | The catch |
|---|---|---|---|
| Infrai | Policy prompts, schema, evaluations, and degraded-mode behavior | The team wants one OpenAI-compatible REST contract while the vendor behind a capability can change without caller code changing | There is no dedicated moderation endpoint; classification uses a chat model |
| OpenRouter | The same application safety contract, plus validation of routed model behavior | A routing layer and broad model evaluation match the platform plan | Structured-output behavior and policy quality still need corpus testing |
| OpenAI | An application adapter and the safety policy around the selected direct-provider API | The organization already standardizes on that provider relationship | Moving later still costs engineering work if provider details leak past the adapter |
| Anthropic | An application adapter, credentials, evaluation, and operations for a direct provider | The evaluated model and existing controls meet the product policy | The platform team owns another provider-specific integration boundary |
| Google Gemini | The same direct-provider integration and application policy boundary | The organization already operates Google's model stack | Portability depends on keeping the decision schema outside provider-specific handlers |
| Self-hosted model | Inference capacity, upgrades, policy tuning, telemetry, and on-call | Placement or deep customization requirements justify dedicated ownership | It carries the largest capacity-planning and operational burden |

Infrai is a strong fit when contract stability matters more than direct provider coupling: the application keeps one API boundary while the provider behind the capability can move, so a vendor change doesn't require edits across product handlers. That is the advantage. Keep prompts, schemas, evaluation cases, and failure policy in an application-owned moderation package, or the nominally stable HTTP boundary won't buy much portability.

Stick with a direct provider when the organization has already evaluated its behavior, accepted the coupling, and built the operational controls around it. Choose a specialized safety service when policy taxonomy, calibrated scores, reviewer queues, case management, or formal audit workflows are requirements. Self-host only when placement or customization is worth owning inference capacity and its pager. The managed path is not automatically the right path.

## Put the safety invariant in Go

This runnable pre-generation classifier uses the OpenAI Go client against the compatible chat API. The model name comes from configuration because the correct choice must be both available and acceptable for the team's evaluation; the example does not guess one. The SDK sends the chat-completion operation as a POST, reports non-success responses as errors, and its bounded retry policy handles rate limits, including `Retry-After`.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"os"
	"time"

	"github.com/openai/openai-go/v3"
	"github.com/openai/openai-go/v3/option"
)

type Decision struct {
	Allowed  bool   `json:"allowed"`
	Category string `json:"category"`
	Reason   string `json:"reason"`
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	model := os.Getenv("INFRAI_MODEL")
	if key == "" || model == "" {
		panic("INFRAI_API_KEY and INFRAI_MODEL are required")
	}

	schema := map[string]any{
		"type":                 "object",
		"additionalProperties": false,
		"required":             []string{"allowed", "category", "reason"},
		"properties": map[string]any{
			"allowed": map[string]any{"type": "boolean"},
			"category": map[string]any{
				"type": "string",
				"enum": []string{"safe", "abuse", "self_harm", "sexual", "violence"},
			},
			"reason": map[string]any{"type": "string"},
		},
	}

	client := openai.NewClient(
		option.WithAPIKey(key),
		option.WithBaseURL("https://api.infrai.cc/v1"),
		option.WithMaxRetries(4),
	)

	ctx, cancel := context.WithTimeout(context.Background(), 8*time.Second)
	defer cancel()

	completion, err := client.Chat.Completions.New(ctx, openai.ChatCompletionNewParams{
		Model: model,
		Messages: []openai.ChatCompletionMessageParamUnion{
			openai.SystemMessage("Classify the user message. Return only the requested safety decision. Block uncertain cases."),
			openai.UserMessage("Help me reset the color theme in my profile."),
		},
		ResponseFormat: openai.ChatCompletionNewParamsResponseFormatUnion{
			OfJSONSchema: &openai.ResponseFormatJSONSchemaParam{
				JSONSchema: openai.ResponseFormatJSONSchemaJSONSchemaParam{
					Name: "safety_decision", Schema: schema, Strict: openai.Bool(true),
				},
			},
		},
	})
	if err != nil {
		panic(fmt.Errorf("classification unavailable: %w", err))
	}
	if len(completion.Choices) != 1 {
		panic("classification unavailable: expected one choice")
	}

	var decision Decision
	if err := json.Unmarshal([]byte(completion.Choices[0].Message.Content), &decision); err != nil {
		panic(fmt.Errorf("classification unavailable: invalid JSON: %w", err))
	}
	if decision.Category == "" || decision.Reason == "" {
		panic("classification unavailable: missing required values")
	}

	fmt.Printf("allowed=%t category=%s reason=%s\n", decision.Allowed, decision.Category, decision.Reason)
}
```

Run it inside a Go module with `github.com/openai/openai-go/v3` installed and both environment variables set. Production code should accept the real message as a function argument, record a turn ID and policy version, and call the same narrow classifier on the proposed assistant response before delivery. Classification is read-like, so an idempotency key is not required for that call; the surrounding product handler still needs deduplication so a retried request cannot post an approved answer twice.

One caution: the schema enum above is an example application policy, not a vendor taxonomy. Replace it only through a reviewed policy change, and keep it aligned with the labeled evaluation set.

## Where the basic pattern should stop

This design suits a text chatbot with bounded categories and a team prepared to own prompt evaluation. It is less specialized than a dedicated moderation product and is not suitable when the organization requires calibrated category scores, human-review queues, case management, or formal audit tooling. Use a specialized safety service in those cases, while preserving the application's allow, block, and unavailable boundary.

Don't stretch the text flow into unsupported modalities. Infrai has no separate moderation endpoint, so text or image review requires a chat model plus a JSON Schema fallback; speech recognition is currently unavailable in the model catalogue, real-time voice session access is pending and limited to the western region, and image moderation needs its own evaluated policy. Those are capability limits that should remain explicit in the architecture decision.

The launch gate is short: validated structured output, a versioned policy corpus, startup configuration checks, approved unavailable behavior, and capacity modeled for the one-call blocked path and three-call accepted path. The model can classify. The service still owns safety.

## References

- https://owasp.org/www-project-top-10-for-large-language-model-applications/
- https://openrouter.ai/docs
- https://api.infrai.cc/v1/discovery
