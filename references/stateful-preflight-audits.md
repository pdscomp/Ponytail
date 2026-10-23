# Stateful preflight audits under Ponytail

Use this checklist when reviewing a plan that changes live backend selection, persisted work, telemetry policy, or confirmation-owned UI state. The goal is the smallest correct adaptation, not a new framework.

## Trace before simplifying

- Identify the canonical policy/backend resolver and grep every caller.
- Trace constructors, serialization/deserialization, retry paths, inherited consumers, and tests that patch the old API.
- A tuple-returning helper is not automatically atomic: verify it does not resolve policy again through a nested helper.
- For multi-step operations, resolve config and identity once and derive host, token, URL, and wording from that immutable snapshot.
- Test snapshot integrity with a fake whose policy answer changes on the second lookup; static-state tests miss double resolution.

## Persisted work and process boundaries

- Persisted signed URLs, headers, or commands need provenance from the backend/session that issued them.
- Append optional fields to preserve positional constructors and legacy records.
- Treat missing provenance as untrusted, filter per item, and rewrite the sanitized cache without marking/deleting the underlying work.
- Check ownership at enqueue, dequeue, and retry boundaries.
- When snapshotting or sanitizing `queue.Queue.queue`, hold the queue mutex; release it before calling another helper that takes the same mutex.
- After mismatch, cancellation, expiry, or exhausted retry, clear the current slot and rewrite the cache immediately. On requeue, persist and clear current before retry sleep so a crash cannot lose the item or expose it as both queued and current.
- If reused daemons are isolated by process, prefer one cache class configured once per process. Test with separate process contexts instead of adding a production reset hook or independently mutable cache objects.

## Reconnect-loop placement

- Keep identity/client resolution inside the reconnect attempt's `try` block. Use the policy-only resolver for pre-loop cache sanitation so a missing identity follows normal retry/backoff instead of killing the daemon before the loop.
- In reconnect tests, return exactly one immutable `(config, client)` pair per socket attempt and assert URI, identity, and token from that pair together; assert resolver call count equals connection attempts to catch hidden extra snapshots.

## Backend identity migration and registration

- A compatibility identity mirrored from the active backend is not valid migration evidence for another backend. Before importing a legacy active ID, reject it when it matches any already-known foreign-backend ID; then fall through to that backend's durable identity or registration.
- Treat explicit unregistered sentinels exactly like a missing identity in the shared client resolver. Registration may opt in to missing identities; normal API, socket, and uploader traffic must fail closed there rather than adding guards to every caller.
- A coherent client snapshot can still become stale during IMEI waits, network backoff, or an in-flight response. Revalidate effective backend immediately before each retry's network I/O and before persisting or mirroring the result; on mismatch, restart identity resolution without silently retargeting one half of the operation.
- Test state transitions, not just isolated endpoints: register backend B, mirror it, switch to backend A with A's ID absent, and prove B's mirrored compatibility value is never adopted as A's identity. Also test that a cached unregistered sentinel is rejected by normal clients.

## Multi-step upload transitions

- Revalidate backend provenance immediately after every issuing-backend RPC, before any terminal-status early return. An old backend's `412`/"ignore" response must not mark a file complete after a flip.
- Carry the operation snapshot through storage completion and recheck before writing an irreversible completion marker such as an upload xattr or queue deletion. Fake-upload success must pass the same provenance gate.
- Exercise retry through the public orchestration path (`upload`/`step`/worker), not by manually calling the low-level request twice. Assert the failed attempt leaves no completion marker, then assert the retry uses the new backend's identity and token and finally marks success.

## Trusted session context

- Do not expose trusted provenance as a caller-controlled RPC argument.
- Reuse an existing dispatcher method map: copy it per session and bind trusted context in closures with only public wire arguments.
- Avoid keyword `partial` for security context when call-time kwargs could override it. Regression-test the real RPC dispatcher by injecting the trusted keyword; it must return invalid params without calling token or network code.
- Session isolation must cover every production producer and consumer: receive, send, response/log, and stat queues. Compatibility defaults do not isolate a daemon whose reconnect call site still omits the per-session queues.

## Session-owned transport shutdown

- `threading.Event.set()` does not interrupt `select()`, socket reads, or other blocking I/O. Close the session websocket before joining readers, or use bounded waits that re-check the event.
- A bounded `select()` does not bound the subsequent `recv()`: one readable byte or a partial frame can enter an unbounded blocking read. Give child sockets finite read/connect timeouts or abort them from the session-end path, then re-check the event after `recv()` before forwarding data.
- Audit synchronous child connection setup inside joined RPC workers. An unbounded DNS/TCP/TLS connect can prevent control-session teardown even when established proxy loops are event-aware.
- Do not use one tiny websocket timeout for both setup and steady-state I/O. In `websocket-client`, `create_connection(timeout=...)` keeps that timeout on the established socket: 100 ms may bound teardown but is not a viable cellular DNS/TCP/TLS budget and can also make proxy I/O brittle. Split a realistic bounded connect timeout from the established-socket timeout, retain explicit close-on-session-end, and re-check the session immediately after connect so a late socket is closed before forwarding.
- Audit every sibling connector and wrapper, not only the primary backend path. A bounded shared proxy shim does not help a secondary wrapper that can still block indefinitely before entering it.
- Audit subordinate transports created by RPCs (local proxies, tunnels, streams), not only the control websocket. Every direction must observe the session event; otherwise an old backend can keep receiving traffic after the control socket closes.
- Test shutdown with both an established idle child transport and a readable-but-incomplete frame; also cover stalled connection setup. Separately assert that the connect timeout is operationally realistic rather than merely forwarded. Assert each transport stops within the policy bound; idle-socket, mocked-connect, and happy-path token tests do not prove quiescence.

## Checked irreversible transitions

- Inspect the production persistence wrapper's failure contract before trusting failure tests. A fake that raises is counterfeit evidence when production discards return codes or reports no status.
- Advance or clear an irreversible staged request only after checked persistence. When the API has no success result, read back the exact value; test silent failed writes at every stage, not only injected exceptions.
- Keep the durable fail-closed marker first, but do not clear the retry request until every required state write is verified. Otherwise a partial failure can make the lock permanent without enabling the requested feature.

## Live guards and UI dismissals

- An initialization-only telemetry guard is insufficient if the SDK may already exist. Guard every public emitting helper before its first SDK call; preserve required local crash writes first and never flush/close on a blocked transition.
- Confirmation-owned settings require unbound controls. Bound controls may persist after callbacks.
- Restore optimistic UI state from the actual Param before pushing a dialog. This covers Cancel, Back, Escape, and external dismissal even when only explicit dialog buttons run callbacks.
- Revalidate safety predicates (offroad, disengaged, permissions) inside the confirm callback. Dialogs outlive the control that opened them, so initiation-time gating does not protect the committing action.
- Put actual-state restoration in `finally`. If staging succeeds but the follow-up reboot/action cannot be armed, roll back the staged request or surface an explicit recoverable state instead of leaving an invisible future trigger.
- Backend-aware pairing preflights must snapshot the effective backend, validate that snapshot's backend-specific identity field, and derive error wording from the same snapshot. A compatibility `DongleId` mirror can hold the other backend's identity after a reversible switch; trusting it can admit pairing only to generate a normal-looking URL with an empty token.
- Token-generation failure must surface the pairing error state, not encode a scannable `?pair=` URL. A plausible QR with no token converts an explicit failure into a silent dead end.

## Verification risks

- Update every inherited/shared call site when signatures change; thread target mismatches may only log.
- Update tests that patch replaced symbols (`Api` versus a snapshot helper) to prevent accidental real network calls.
- For confirmation-owned UI, test the production callback without constructing the full renderer: create a bare owner, replace the dialog with a fake that captures its callback, and collect `push_widget` calls. Keep the real checked-write helper in the path with a fake Params that can both raise and silently drop writes.
- Platform skip hooks may classify tests from parametrization IDs. Use neutral IDs for host-runnable variants (for example, `standard`, not `tici`), run with skip reasons enabled, and require every intended variant to execute.
- A texture stand-in used with UI destructors must expose the fields the destructor reads (for raylib textures, at least `id=0`); a bare `object()` can turn a useful regression failure into an unraisable destructor error.
- An unconditional skip is no evidence, especially if imports are inside the skipped body.
- Under a strict read-only audit, do not run commands that create caches/generated files. If tests are explicitly required, run pytest cachelessly with `PYTHONDONTWRITEBYTECODE=1 pytest -p no:cacheprovider ...` (and wrap UI tests in `xvfb-run -a` when needed), then compare `git status --short` before and after. If the suite itself writes fixtures or generated artifacts, do not run it; state the verification gap and provide the exact later command.

## Ponytail stop rule

Reuse the existing resolver, dispatcher mapping, queue/cache class, retry behavior, control, and dialog. Add a framework only when those concrete extension points cannot satisfy a stated requirement.
