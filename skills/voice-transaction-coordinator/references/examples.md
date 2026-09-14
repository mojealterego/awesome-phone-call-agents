# Voice Transaction Coordinator — Examples

These examples use fictional values and are intended for offline reasoning only. They do not place calls or authorize a real-world action.

## Example 1: COMMIT

Prepared transaction:

```json
{
  "operation_key": "af-demo-001",
  "participant_id": "TRUCK-42",
  "endpoint": "+15550100",
  "route": "B",
  "max_eta": "19:00",
  "scope": "route_change"
}
```

Observed terminal evidence:

```json
{
  "provider_status": "completed",
  "route": "B",
  "acceptance": "yes",
  "eta": "18:40",
  "confidence": "high",
  "task_completed": true,
  "evidence_summary": "Recipient accepted Route B and stated an ETA of 18:40."
}
```

Decision: `commit` because the provider is terminal-completed, acceptance is positive, the route matches, the ETA satisfies the prepared maximum, and the evidence is sufficient.

## Example 2: ABORT on conflict

Prepared route: `B`.

Observed terminal evidence:

```json
{
  "provider_status": "completed",
  "route": "C",
  "acceptance": "yes",
  "eta": "18:30",
  "confidence": "high",
  "task_completed": true
}
```

Decision: `abort` because the participant accepted a route different from the prepared transaction. Do not silently replace Route B with Route C.

## Example 3: RECOVER on unknown execution

The client loses its response after creating a CALL-E task. The local record contains a call identifier, but the terminal provider state is not known.

Decision: `recover`.

Recovery should re-fetch the existing provider call and reconcile its authoritative state. It must not place a second outbound call merely because the original response was lost.

## Example 4: RECOVER on incomplete evidence

The provider reports terminal completion, but the structured result does not establish route acceptance or contains `unknown` for a required business field.

Decision: `recover` until authoritative evidence is sufficient. Provider completion alone is not business acceptance.

## Example 5: ABORT on complete rejection

Prepared route: `B`.

Observed terminal evidence:

```json
{
  "provider_status": "completed",
  "route": "B",
  "acceptance": "no",
  "confidence": "high",
  "task_completed": true
}
```

Decision: `abort`. The conversation completed, but the prepared action was not accepted.

## Endpoint binding

Two otherwise identical prepared transactions authorized for different endpoints must produce different authorization/receipt material. A phone endpoint is therefore part of the authorization boundary, while the phone number itself is not treated as proof of identity.
