# Cascade Context Token Usage — `GetCascadeTrajectoryGeneratorMetadata`

This document covers a separate RPC that the existing
[CASCADE_PROTOCOL.md](CASCADE_PROTOCOL.md) and
[API spec](WINDSURF_API_SPEC.md) don't mention:
**`GetCascadeTrajectoryGeneratorMetadata`**. This is the endpoint the IDE
uses to populate the context-usage indicator (e.g. `63% (125K / 200K)`)
shown next to the Cascade chat input.

If you need real token counts — not a character-based estimate — this is
where they live.

---

## 1. It's a separate RPC, not part of `GetCascadeTrajectorySteps`

`GetCascadeTrajectorySteps` returns the trajectory *content* (planner
responses, tool calls, thinking). It does **not** include token usage.

Token usage lives in a parallel metadata stream that you poll alongside
the trajectory steps:

```
InitializeCascadePanelState(metadata)
 → StartCascade(metadata, source)          → cascade_id
 → SendUserCascadeMessage(cascade_id, …)
 → poll GetCascadeTrajectorySteps(cascade_id, step_offset)        ← content
 → poll GetCascadeTrajectoryGeneratorMetadata(cascade_id, offset) ← token counts
```

Both endpoints use offset-based pagination. Track the offsets independently
per cascade session.

---

## 2. Request format

```
GetCascadeTrajectoryGeneratorMetadataRequest:
  field 1: cascade_id                    (string)
  field 2: generator_metadata_offset     (uint32)
```

Same `cascade_id` you get from `StartCascade`. The offset starts at 0 and
advances by the number of metadata entries returned each poll.

---

## 3. Response format

```
GetCascadeTrajectoryGeneratorMetadataResponse:
  field 1: generator_metadata (repeated CortexStepGeneratorMetadata)
```

Each `CortexStepGeneratorMetadata` entry corresponds to one LLM invocation
within the turn. A single user message can produce multiple entries (e.g.
if the planner makes multiple LLM calls for tool selection + response
generation).

---

## 4. The token-usage chain

```
CortexStepGeneratorMetadata
  └─ field 1: chat_model (oneof "metadata", ChatModelMetadata)
       └─ field 4: usage (ModelUsageStats)
            ├─ field 2: input_tokens         (uint64)  ← context tokens used
            ├─ field 3: output_tokens        (uint64)  ← tokens generated
            ├─ field 4: cache_write_tokens   (uint64)
            ├─ field 5: cache_read_tokens    (uint64)
            ├─ field 6: api_provider         (enum)
            ├─ field 7: message_id           (string)
            ├─ field 8: response_header      (map<string, int64>)
            ├─ field 9: model_uid            (string)
            ├─ field 10: billing_model_uid   (string)
            └─ field 11: requested_model_uid (string)
```

**`ModelUsageStats.input_tokens` (field 2) is the value the IDE displays
as the numerator in its context-usage indicator.** It represents the total
prompt tokens sent to the LLM for that invocation — system prompt +
conversation history + tool results + the current user message.

The **last** entry with `input_tokens > 0` in a poll response represents
the most recent context usage. Use it for the live meter.

---

## 5. `CortexStepGeneratorMetadata` full schema

| Field | Name                                  | Type      | Notes                              |
|-------|---------------------------------------|-----------|------------------------------------|
| 1     | chat_model                            | message   | oneof "metadata" → ChatModelMetadata |
| 2     | step_indices                          | uint32[]  | Which steps this metadata covers   |
| 3     | planner_config                        | message   | Planner configuration              |
| 4     | execution_id                          | string    |                                    |
| 5     | error                                 | string    |                                    |
| 6     | parallel_rollout_generator_metadata   | message   |                                    |
| 7     | arena_cap_reached                     | bool      |                                    |

---

## 6. `ChatModelMetadata` full schema

| Field | Name                      | Type      | Notes                              |
|-------|---------------------------|-----------|------------------------------------|
| 1     | system_prompt             | string    |                                    |
| 2     | message_prompts           | message[] | ChatMessagePrompt (repeated)       |
| 3     | model (deprecated)        | enum      | Model enum                         |
| 4     | usage                     | message   | **ModelUsageStats** ← token counts |
| 5     | model_cost                | double    |                                    |
| 6     | last_cache_index          | uint32    |                                    |
| 7     | tool_choice               | message   |                                    |
| 8     | tools                     | message[] | ChatToolDefinition (repeated)      |
| 9     | chat_start_metadata       | message   |                                    |
| 10    | message_metadata          | message[] | MessagePromptMetadata              |
| 11    | time_to_first_token       | Duration  |                                    |
| 12    | streaming_duration        | Duration  |                                    |
| 13    | credit_cost               | int32     |                                    |
| 14    | retries                   | uint32    |                                    |
| 15    | model_uid                 | string    |                                    |
| 16    | acu_cost                  | double    |                                    |
| 18    | quota_cost_basis_points   | sint32    | optional                           |
| 19    | overage_cost_cents        | sint32    | optional                           |
| 20    | response_dimension_groups | message[] |                                    |

---

## 7. Where the denominator (max context window) comes from

The percentage denominator is **not** in the generator metadata. It comes
from `IntentToolConfig`:

```
exa.cortex_pb.IntentToolConfig:
  field 1: intent_model (deprecated)  (enum)
  field 2: max_context_tokens         (uint32)  ← max context window
  field 3: intent_model_uid           (string)
```

This is per-model, not per-turn. To compute the percentage shown in the UI:

```
percentage = input_tokens / max_context_tokens * 100
```

You can discover `max_context_tokens` for each model at runtime via
`GetUserStatus` → `cascade_model_config_data.client_model_configs[]`,
as noted in [CASCADE_PROTOCOL.md](CASCADE_PROTOCOL.md) §3.

---

## 8. Protocol notes

The existing docs describe `application/grpc-web+proto` framing. This
endpoint also works with **Buf Connect-RPC** (`application/proto`,
`Connect-Protocol-Version: 1`), which is simpler — no gRPC framing
(length-prefixed messages + trailers), just raw protobuf in the request
and response body. The IDE's bundled extension uses Protobuf-ES
(`@bufbuild/protobuf`), and the language server accepts both framing
styles.

Headers required (same as other Cascade RPCs):

```
Content-Type: application/proto
Connect-Protocol-Version: 1
x-codeium-csrf-token: {csrf_token}
Authorization: Bearer {api_key}
```

URL pattern:

```
POST http://127.0.0.1:{port}/exa.language_server_pb.LanguageServerService/GetCascadeTrajectoryGeneratorMetadata
```

---

## 9. Verification

A test message ("Hello, what is 2+2?") with an injected system prompt
returned:

```
GeneratorMetadata #1: input_tokens=1719 output_tokens=25
```

1,719 input tokens for a trivial question is consistent with the system
prompt + context overhead that gets prepended before the user message
reaches the LLM.

---

## 10. How to discover this in `extension.js`

The bundled extension at `resources/app/extensions/windsurf/dist/extension.js`
uses Protobuf-ES, which registers every message type with a `static typeName`
and a `static fields` array. To find the schema:

```bash
# List all protobuf type names
grep -oE 'typeName="exa\.[^"]+"' extension.js | sort -u

# Get field definitions for a specific type
# (search for the typeName, then read the newFieldList that follows)
```

The `ModelUsageStats` type is under `exa.codeium_common_pb`, not
`exa.cortex_pb` — it's shared between autocomplete and Cascade. The chain
from `CortexStepGeneratorMetadata` down to `ModelUsageStats` is:

```
exa.cortex_pb.CortexStepGeneratorMetadata
  → exa.cortex_pb.ChatModelMetadata (field 1, oneof "metadata")
    → exa.codeium_common_pb.ModelUsageStats (field 4, "usage")
```

Search for `input_tokens` or `ModelUsageStats` in the extension to find
the field definitions directly.
