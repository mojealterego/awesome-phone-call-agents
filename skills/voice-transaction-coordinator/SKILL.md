---
name: voice-transaction-coordinator
description: Reconcile a real-world phone-call result against a prepared business intent before allowing an operational state change. Use when a CALL-E workflow can create consequential side effects and the phone conversation must not be treated as the commit boundary.
license: MIT
---

# Voice Transaction Coordinator

A phone call is an unreliable participant in a real-world transaction. The caller may disagree with the prepared action, the provider may fail after the call was created, the result may be incomplete, or a retry may accidentally create a second call.

This skill separates phone execution from business authorization:

```text
intent
  ↓
PREPARE — freeze the exact constraints
  ↓
AUTHORIZE — bind participant + endpoint + scope + constraints + TTL
  ↓
CALL — execute through the phone provider
  ↓
VERIFY — require terminal provider state + structured evidence
  ↓
RECONCILE
  ├── COMMIT  — evidence matches the prepared transaction
  ├── ABORT   — terminal evidence conflicts with it
  └── RECOVER — execution/evidence is incomplete or uncertain
```

## Core rules

1. The phone conversation is never the business commit boundary.
2. Create a stable logical operation key before provider I/O.
3. Use provider idempotency for retries of the same logical operation.
4. Prepare immutable business constraints before calling.
5. Bind authorization to the participant, exact execution endpoint, scope, constraints and an expiration time.
6. Treat a phone number as an endpoint, not proof of participant identity.
7. Require terminal provider completion separately from conversational content.
8. Use explicit `unknown` states instead of guessing.
9. Reconcile observed evidence against the prepared constraints deterministically.
10. On uncertain execution, recover the existing call before considering any new outbound call. Never blindly retry an unresolved call.
11. Treat provider webhooks as notifications. Validate and deduplicate them, then obtain authoritative provider state before consequential mutation.
12. Produce an auditable receipt that binds transaction, capability, evidence and disposition.

## Evidence contract

A provider result should expose, directly or through a normalized adapter:

- terminal provider status;
- structured business fields;
- task completion state;
- completion confidence;
- evidence items or evidence summary;
- provider call identifier;
- failure information when execution did not complete.

Do not infer a business commitment merely because a call has a `completed` status. Provider completion and business acceptance are separate facts.

## Reconciliation

### COMMIT

Only when all required constraints match and the provider reports successful terminal completion with sufficient evidence and confidence.

### ABORT

When terminal evidence is complete but conflicts with the prepared transaction. Do not silently rewrite the prepared transaction from an unapproved phone answer.

### RECOVER

When execution, provider state or evidence is incomplete or uncertain. Re-fetch the existing call where possible and resume verification. Do not create a second call merely because the first response was lost.

## Secondary telephony evidence

A deployment may add an independent PBX/telephony evidence provider such as Ringostat. Normalize its call-log or webhook data into evidence and use it to corroborate facts such as disposition, duration or recording availability.

Secondary evidence must not authorize the transaction, override prepared constraints, turn a webhook into a commit signal, replace authoritative recovery, or establish participant identity from caller ID alone.

## Dry-run requirement

Provide a local preview or fixture path that exercises the same validation and reconciliation logic without placing a real call. Live execution must be explicit opt-in and require an authorized E.164 recipient and server-side credentials.

## Suggested result

```json
{
  "decision": "commit | abort | recover",
  "reasons": [],
  "operation_key": "...",
  "provider_call_id": "...",
  "evidence": {},
  "receipt_id": "..."
}
```

The decision packet is an operational control record. It is not a claim that a phone conversation is a legally binding contract.
