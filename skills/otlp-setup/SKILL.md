---
name: otlp-setup
description: Use when setting up or troubleshooting trace ingestion into LightShip through its OTLP/HTTP endpoint, including authentication, trace payloads, span identities, and ingestion results.
---

# Send traces to LightShip

LightShip helps monitor agent behaviour using agents OTEL data. 

Configure trace ingestion using the supplied LightShip endpoint and API key. Send a valid OTLP request and inspect the received trace before reporting successful ingestion.

## 1. Establish the endpoint and authentication

Use the full traces URL supplied in the setup prompt as `LIGHTSHIP_OTLP_TRACES_URL` and the API key supplied as `LIGHTSHIP_API_KEY`. Reuse supplied values and ask only for missing values; keep the key out of committed files and displayed output.

Send requests to `${LIGHTSHIP_OTLP_TRACES_URL}` exactly as supplied. Supply the API key separately in the `Authorization` header, not in the URL or trace payload:

```http
POST ${LIGHTSHIP_OTLP_TRACES_URL}
Authorization: Bearer <API key>
Content-Type: application/json
```

Use a **Lightship API key** with `ingestion:write`. The key determines the workspace; telemetry attributes cannot change it.

Image uploads also need `attachments:write`. To verify traces and images through the API, the key needs `traces:read` and `attachments:read`. If read access is missing, report ingestion acceptance separately from verification.

Configure the OTLP/HTTP trace exporter with the supplied full traces URL. Send an OTLP `ExportTraceServiceRequest` using either `application/json` or `application/x-protobuf`. Keep each request body within **4 MiB**; configure smaller batches when necessary.

## 2. Set agent and customer identities

Add the following optional **string attributes to spans**:

| Attribute | Meaning |
| --- | --- |
| `agent.id` | Stable ID chosen for the agent producing the span. |
| `customer.id` | Customer whose work the agent handled, within the LightShip organization. |
| `user.id`, then `enduser.id` | End-user identity; the first nonempty key wins. |
| `gen_ai.conversation.id`, then `session.id`, then `conversation.id` | Conversation identity; the first nonempty key wins. |

These IDs do not need prior registration. Use actual identities when available; leave optional identities unset when they are not needed. Resource attributes such as `service.name` describe the application. Setting `agent.id` or `customer.id` only on the resource does not establish agent or customer identity.

For agent and customer IDs to resolve as trace identities, values must be nonempty strings, contain no NUL, and be at most **512 UTF-8 bytes**. Invalid values remain in the original span attributes but do not populate the resolved identity.

LightShip resolves trace-level identities from spans:

- When exactly one root span exists and has a nonempty value, that value wins.
- Otherwise, all nonempty span values must agree; conflicting values leave the trace identity unset.
- An invalid value selected from the root does not fall back to a child value.
- Span queries use each span's own attributes and do not inherit root identities.

Use consistent values across spans when one trace-level identity is intended. Set the identity on each relevant span when span-level queries must match it.

## 3. Describe model and tool spans

Use the relevant attributes on the spans that recorded these operations:

| Operation | Span attributes |
| --- | --- |
| Model call | `gen_ai.operation.name=chat` |
| Model details | `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.response.id` |
| Model input/output | `gen_ai.input.messages`, `gen_ai.output.messages` |
| Token usage | `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.usage.total_tokens` |
| Finish reasons | `gen_ai.response.finish_reasons` |
| Tool execution | `gen_ai.operation.name=execute_tool` |
| Tool details | `gen_ai.tool.name`, `gen_ai.tool.call.id`, `gen_ai.tool.call.arguments`, `gen_ai.tool.call.result` |

Represent structured messages and tool arguments/results as JSON-encoded strings in OTLP attributes. Use integer values for token counts and an OTLP array for finish reasons. Populate fields from the recorded operation rather than inventing missing values. LightShip promotes supported model attributes into normalized `llm`, `input`, and `output` fields.

## 4. Construct and send the OTLP request

Preserve the tracer's trace IDs, span IDs, parent relationships, and timestamps. For a new test trace, use fresh IDs and current timestamps:

- `traceId`: nonzero, 32-character hexadecimal ID.
- `spanId`: nonzero, 16-character hexadecimal ID.
- `parentSpanId`: the actual parent's span ID, when there is a parent.
- `startTimeUnixNano` and `endTimeUnixNano`: Unix nanoseconds encoded as decimal strings.

For JSON requests, use this structure. Replace the uppercase placeholder values before sending; add the relevant span attributes from the tables above.

```json
{
  "resourceSpans": [{
    "resource": {
      "attributes": [{
        "key": "service.name",
        "value": {"stringValue": "example-service"}
      }]
    },
    "scopeSpans": [{
      "scope": {"name": "ingestion-check"},
      "spans": [{
        "traceId": "TRACE_ID",
        "spanId": "SPAN_ID",
        "name": "ingestion-check",
        "kind": 1,
        "startTimeUnixNano": "START_UNIX_NANOSECONDS",
        "endTimeUnixNano": "END_UNIX_NANOSECONDS",
        "attributes": []
      }]
    }]
  }]
}
```

Send the prepared payload from `trace.json`, using the supplied endpoint and a key available in the local environment:

```sh
curl --fail-with-body "$LIGHTSHIP_OTLP_TRACES_URL" \
  --header "Authorization: Bearer $LIGHTSHIP_API_KEY" \
  --header 'Content-Type: application/json' \
  --data-binary @trace.json
```

The OTLP/HTTP exporter can send the same request automatically. A manual request checks connectivity and credentials; verify the configured exporter's own traces when setting up ongoing ingestion.

## 5. Inspect acceptance and received traces

HTTP `200` means LightShip durably queued the accepted spans for asynchronous processing. The normal JSON success response is `{}`. Inspect `partialSuccess`, when present, for rejected spans.

Normalization, trace revision allocation, and evaluation occur later. Inspect the received trace using `GET /traces/{id}` on the LightShip API host. A trace may initially be `provisional`; later spans can change its revision. Report acceptance separately from confirmed trace visibility.

Re-sending a batch is **not an update**. Repeated spans can appear in queries and affect counts or evaluation results, even though some exact replays may later be compacted. Inspect the original trace before deciding whether another submission is needed.

## 6. Handle ingestion errors

| Response | Check |
| --- | --- |
| `401` | Valid Lightship API key and Bearer authentication. |
| `403` | API key permissions: `ingestion:write` for traces, `attachments:write` for uploads, `traces:read` / `attachments:read` for verification. |
| `400` | OTLP payload structure and field values. |
| `415` | Supported content type matching the actual encoding. |
| `413` | Request body exceeds the 4 MiB limit; reduce the batch size. |
| `502` | The body limit may have been reached while streaming through the public endpoint; check request size. |

Finish by reporting the configured endpoint with credentials redacted, the response or error observed, and whether the submitted trace was retrieved successfully. State any verification step that remains incomplete.
