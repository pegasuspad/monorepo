Appendix A: Threat Vector Justifying HMAC Enforcement

Title: Authenticated Payload Forgery via Compromised Producer Identity

Summary:

This appendix describes a realistic and concrete threat vector demonstrating how a valid IAM-authenticated service can emit forged, high-privilege commands without detection or blocking, unless message-level integrity (e.g., HMAC) is enforced. The scenario bypasses IAM, EventBridge source binding, and centralized ACL enforcement, and is applicable within the trust boundaries of the existing Snavio architecture.

Threat Scenario: Payload Forgery Using a Compromised IAM Principal

Assumptions:

    analytics-service is a legitimate, IAM-authenticated service authorized to emit snavio.command-sent events with events:source = analytics-service

    Dispatcher enforces ACLs on the triplet (source, command.name, target)

    analytics-service is not allowed to emit certain sensitive commands (e.g., delete-user, revoke-session) to high-privilege targets like auth-service or user-service

Attack Steps:

    Compromise or impersonate analytics-service: An attacker gains access to the IAM role or Lambda environment used by analytics-service via SSRF, dependency injection, or credential leak.

    Construct a forged message payload: The attacker emits a syntactically valid payload with a high-privilege command: { "command": { "name": "delete-user", "target": "user-service", "payload": { "userId": "1234" } }, "metadata": { "id": "<uuid>", "timestamp": "<ISO8601>", "hmac": null_or_garbage } }

    Emit to EventBridge using valid IAM credentials: The attacker sets events:source = analytics-service. EventBridge accepts the message.

    ACL enforcement triggers rejection (if the command is unauthorized): The dispatcher evaluates (analytics-service, delete-user, user-service) and rejects if not allowed. However, the attacker observes snavio.command.failed responses.

    Enumeration and escalation: The attacker sends forged commands across many combinations. When they discover a valid ACL (e.g., analytics-service → metrics-service → queue-metrics-upload), they craft malicious payloads under that command and dispatch successfully.

Why IAM and ACLs Alone Fail:

IAM + EventBridge source:

    Only verifies who sent the message

    Does not verify whether the payload was generated by authorized application logic

ACL enforcement:

    Filters known bad command routes

    Cannot detect forged-but-valid messages over allowed command paths

No HMAC:

    Allows arbitrary command construction and emission using valid credentials

    Provides no payload integrity check

What HMAC Enforcement Provides:

    Cryptographic integrity of payload

    Tamper detection

    Replay protection (with timestamp-based validation)

    Insider threat mitigation

    Payload provenance tied to generating logic, not just the emitter's identity

Without HMAC, You Allow:

    Valid IAM-authenticated services to forge command types they were never intended to emit

    Enumeration of valid ACL entries via observable dispatcher rejections

    Malicious payload injection via weakly validated command paths

    No ability to distinguish legitimate from synthetic messages after IAM compromise

Secret Hygiene Mitigation:

    Store HMAC keys in scoped SSM Parameter Store entries, restricted by IAM

    Enforce TTL validation on timestamps in the HMAC input

    Rotate keys on schedule or incident

    Accept timestamp windows (e.g., ±2 minutes) to tolerate clock drift

Recommendation:

Maintain mandatory HMAC enforcement for all snavio.command-sent messages. IAM enforces identity. HMAC proves message integrity and origin authenticity. Alternatively, allow opt-out only for explicitly low-sensitivity commands, with enforcement required for all critical and moderate ACL-tagged messages.
