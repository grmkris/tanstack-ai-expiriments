# Collaborative Chat V1 — Implementation Specification

Status: proposed architecture and implementation plan; not implemented or integration-tested.
Review date: September 11, 2026.

This document supersedes CHAT_EXPERIENCE_V1_SPEC.md where requirements conflict. In particular, real-backend V1 now includes external Durable Streams, server-owned queueing, StreamDB/TanStack DB synchronization, and multiple authenticated humans sharing one AI conversation. Do not inherit the earlier single-client queue or memory-only delivery defaults.

## 1. Product contract

Build one excellent chat experience. A conversation is a room with members and one logical assistant. A personal conversation is the same room model with one human member.

The reference demonstration must work with two separately authenticated people, ten desktop tabs, and a mobile browser accessing the application privately through Tailscale. Everyone sees the same accepted messages, shared queue, selected response attempts, and ongoing assistant output. Closing the initiating browser must not own or terminate the work.

Sending, queueing, cancellation, correction, regeneration, reconnecting, and reading streamed output must feel immediate and remain understandable under concurrent actions. No lost composer text, duplicate bubbles, accidental repeated executions, or hidden replacement of another person's messages.

In scope: authentication, room invitations/membership, attributed messages, one assistant per room, shared queue and controls, persistent history, response attempts, streaming, reconnect/catch-up, basic presence, and native durable-run wiring for the selected coding harness.

Out of scope: file explorer, editor, terminal, repository review, multi-agent orchestration, collaborative document editing, marketplace, attachments, voice, and general historical conversation trees. A harness can have a workspace internally without exposing a workbench.

## 2. Library boundaries and the architecture decision

Use these responsibilities, not a blanket installation of every overlapping transport:

| Concern | Selected role |
|---|---|
| Native AI execution and event types | TanStack AI and a supported model/harness adapter |
| React chat composition and stream processing | Native TanStack chat client/UI APIs; use the documented UI entry point and pinned types |
| Shared reactive records | TanStack DB collections through StreamDB |
| Structured synchronization | Durable State protocol through `@durable-streams/state/db` |
| Native per-run output durability | `@tanstack/ai-durable-stream` with the native response/replay lifecycle |
| Canonical server persistence | One PostgreSQL database, implementing native persistence contracts and application records |
| Agent capture/recovery | Native sandbox durable-run, journal, takeover, and reaper machinery where supported |
| Presentation | Requested shadcn components, especially Message Scroller |

StreamDB is accessed through `createStreamDB` from `@durable-streams/state/db`; it builds TanStack DB collections over a Durable State stream. It provides a synchronization/read-model implementation rather than requiring our own browser CRUD-event reducer. [S1–S3]

There are TWO different AI integrations. `@durable-streams/tanstack-ai-transport` supplies a conversation-session connection and helpers, including prompt echo and history snapshots; its documented pattern explicitly supports multiple clients. Native `@tanstack/ai-durable-stream` implements TanStack's per-run durability interface used by sandbox recovery. They are not the same interface. [S4, S5]

Decision: start with native run durability plus StreamDB for room state. Do not also mirror all token events into a second session transport by default. The session transport is an alternative delivery architecture, not a requirement to layer onto the chosen path. It may be evaluated in the integration spike, but any change must preserve native recovery, one streaming reducer, one history authority, and the acceptance tests below.

## 3. Logical topology

```text
React chat UI, independently on every device
  |-- native TanStack stream processor: active response parts
  |-- StreamDB / TanStack DB: shared room records and controls
  |-- local UI: composer draft, scroll, selections, expansion
                 |
        Authenticated HTTP commands and stream reads
                 |
             Application backend
  |-- authorization and room admission
  |-- native TanStack execution / persistence / recovery
  |-- transactional room-state publisher
                 |
       +---------+----------------------+
       |                                |
 PostgreSQL                       Durable Streams
 canonical records                room/<id>/state
 native stores                    run/<runId>/events
 outbox / command receipts
       |
 Selected agent execution environment
 native capture journal when supported
```

The names above are proposed stream namespaces, not hard-coded upstream defaults.

Keep two logical streams because their jobs differ. The room-state stream continues across turns. A run event stream describes a particular native execution. Reuse the same Durable Streams service; do not add another event bus solely for room synchronization.

The room stream advertises accepted messages, member changes, queue changes, response-attempt selection, and run lifecycle. Consequently, an idle tab learns about the NEXT response as well as the current one. Native output remains native chunks; do not invent a universal token envelope or handwritten message-part assembler.

## 4. Authority, projection, and finalization

The database is authoritative for membership, accepted commands, message records, selected attempts, and run control. Native stores remain authoritative for their native lifecycle fields. StreamDB is the materialized client view for this design; its optimistic writes are not authorization or distributed execution locks.

Project committed, public room-state changes to the state stream. `createStreamDB` does not automatically watch arbitrary PostgreSQL writes: the implementation must supply this publisher. Use a transactional outbox or an equivalently tested publication mechanism. Database success followed by a stream outage must not permanently hide an accepted command from other users.

Publish changes from ALL writers, including fresh execution, cancellation, recovery, and the reaper. Prefer integrating the outbox at the persistence/application-store transaction boundary instead of scattered route-level notifications. Retain a room revision and idempotent publication identity; retries must converge rather than repeatedly applying a user action.

Use native message parts for stored assistant content. For an in-progress response, the native processor's assembled message is a local rendering projection of that run's log. Overlay it onto the corresponding persisted message identity; do not render it as an unrelated second bubble. When final content is committed and observed, retire the live overlay without a visual jump or loss of partial output.

Do not update a PostgreSQL message row or a StreamDB message entity for every token. Stream the deltas through native run delivery and persist/project content at the native supported persistence boundaries. Room-state events primarily describe accepted records and structural changes.

The client facade must coordinate history hydration, room revisions, and native run attachment. Choose one hydration owner; do not concurrently drive independent full-history replacement mechanisms. Native server-authoritative persistence is a useful starting point, but live room updates still require explicit integration. [S6]

Required race handling: loading history while a message arrives, a new run starting during attachment, terminal status arriving before the client drains the final output, and final persistence arriving after run completion. Never clear the partial response merely because a summary record is terminal. Snapshot/replay cutoffs must be consistent; blind "fetch then subscribe" is insufficient.

## 5. Identities and membership

Separate model role from real-world author. Two humans can both produce native `user` messages while retaining different authenticated authors.

Proposed application metadata, not an upstream API:

```ts
interface MessageAuthorship {
  messageId: string;
  roomId: string;
  authorId: string;
  authorKind: 'human' | 'assistant';
  roomSequence: number;
  revision: number;
  replyToMessageId?: string;
}
```

Use a stable room/thread mapping and native message/run identities. Keep immutable command IDs, response-attempt IDs, and author associations where native contracts do not represent them. A logical assistant turn may contain more than one native run; do not release the room's execution slot between tool/interrupt continuations just because one segment finished.

Suggested application records: rooms, memberships, message authorship/revisions, assistant attempts, queued commands, command receipts, and publication outbox. Reuse native messages/runs/interrupts storage rather than creating competing copies of native lifecycle state.

Membership roles: owner, participant, viewer. Owner manages membership and the shared agent configuration. Participants can post, ask the assistant, and stop shared work under the default room policy. Viewers only read. Participants can edit/cancel their own eligible messages; owners may moderate explicitly. Do not silently allow one participant to overwrite another's prompt.

Include genuine application sessions and an invitation flow in the template. A development seed creates two users and a shared room through the normal authorization paths. Disable development shortcuts in production. An invitation URL is a scoped, expiring invitation, not permanent stream credentials.

Derive author identity from the authenticated session, not a request body `authorId`. Protect every room snapshot, historical stream, live stream, command, and native run lookup. A run belongs to a room; possession of a runId is not permission.

Use room-scoped streams or equivalently server-authorized projections. A browser-side TanStack DB filter is not an access-control boundary. Never send a global stream with other rooms' private records and rely on filtering them out locally.

On membership removal, reject new commands and revoke future live access through an explicit bounded revocation mechanism. Stop/revalidate existing reads as necessary. Do not promise erasure of content already received by a removed participant.

## 6. What two humans plus one AI means

Default Send requests an assistant response. Provide a small secondary "Message room" mode to post human discussion without automatically invoking the assistant. Represent this as explicit server-side intent, not an AI guess based on punctuation or a mention string.

When a human asks while the assistant is idle, admit one assistant turn. When another asks while it is busy, accept the request into the shared FIFO queue. The UI says that it is queued and not yet included in the current reply. All members see the author and the same queue position.

Assign ordering on the server; client clocks do not determine canonical order. Preserve submission identity through uncertain acknowledgements. Retries reconcile an existing receipt; they must not create another assistant turn.

Before execution, freeze the exact context message IDs and revisions, requesting actor, selected profile revision, and relevant room state. The active run does not silently change its input when another person types. Exclude future queued questions until admitted. Define whether eligible room-only comments are included when the next turn starts, and record that choice.

Build model-visible speaker labels from authenticated author metadata at the server serialization boundary. Do not assume arbitrary message metadata survives every harness adapter. Test that the AI can distinguish participants. Display names and quoted content never grant tool permissions; all human content remains untrusted user-level content.

Shared configuration and tool access must be intentional. Joining a room must not silently expose an owner's unrelated private accounts, credentials, memory, or broader tools. V1 uses a limited shared execution profile and server-side tools only.

## 7. Shared interaction contract

### Send and queue

Capture text into recoverable local pending state immediately, render feedback, clear the composer, and preserve focus. The network acknowledgment and first model event are distinct statuses. Keep typing responsive throughout.

Accepted queue entries are server-owned. Do not also dispatch them through TanStack's client FIFO. The native queue's documented stop/error/reload semantics differ from a shared persistent queue. [S7]

Proposed initial limits: five outstanding AI requests per participant and twenty per room, configurable and enforced on the server. Rejected submissions remain recoverable in the originating client.

On normal completion, drain the next queued request through one server admission path. After a shared Stop or execution failure, preserve and pause remaining entries. Make the paused reason visible and require an authorized Resume. Closing a tab never pauses or deletes the server queue.

### Cancel queued input

A participant can cancel their own not-yet-started entry. The owner can moderate the queue. Undo restores an unsent local draft, not an automatic new request.

Use a conditional server transition so cancellation racing with dispatch yields one truthful result. If the request has started, show that fact and offer Stop; do not pretend it was never accepted.

### Stop

Target the exact native run/assistant attempt or pending submission. Do not implement Stop as "look up whichever run is active when the delayed request arrives".

React locally with `Stopping…`, retain output, and record cancellation intent through the supported backend path. Confirmation of intent is not confirmation of termination. Other views learn the same control status and actor attribution.

For durable harness runs, local `chat.stop()` is not remote cancellation. Use native cancellation intent plus the appropriate execution stop path. A terminal outcome must come from execution, not from the HTTP handler deciding success. [S5]

Do not let a late cancellation for run A stop replacement B. A response that finished before cancellation should retain its real completed state. On uncertain cancellation, retain the replacement/draft and show uncertainty rather than run both.

### Stop and send

Make this explicit. Atomically claim the replacement operation against the expected current attempt, preserve/pause existing queue entries, request cancellation, and wait for confirmed termination before launching the replacement. Two concurrent replacements must not both win.

### Regenerate / retry

Transport retry reuses command identity and attaches to existing work. Deliberate regeneration creates a new response attempt and does not duplicate the user message.

Support the latest eligible response only. Require an expected message/attempt revision; preserve prior attempts and publish the shared selected attempt. Allow users to inspect an older attempt locally without changing shared context selection.

Do not regenerate the same answer twice because two tabs clicked. The losing stale action should reconcile and explain which newer attempt is active.

For a coding harness, a rerun is new execution, not undo of tools or native session state. Keep its semantics honest even though V1 exposes only chat.

### Edit

Allow editing one's own latest eligible prompt, with revision preservation. Do not silently rewrite context already consumed by later turns or change another participant's message. V1 can reject historical edits that require branching, explaining why.

## 8. Two different concurrency boundaries

Room admission prevents two distinct human commands from starting overlapping assistant turns. Implement it transactionally on server-owned room/queue state. Native run recovery does not replace this admission rule: two different run IDs are still two jobs.

Native run ownership coordinates hosts attempting to drive the SAME execution. Reuse TanStack's lifecycle and ownership mechanisms, with a tested shared LockStore for a multi-host deployment. `withLocks` exposes a capability; it does not automatically serialize an entire conversation. [S8]

Use one native RunStore shared between persistence and sandbox machinery. Preserve all required native durable fields rather than mapping only a simplified status enum. Private recovery fields belong on the server, not in a broadly published room projection.

No browser subscription, newly observed user message, or restored client state may itself start a new model invocation. The backend dispatches accepted intents. Use server-side tools for V1; spectators must never execute a tool once per tab.

## 9. Native durable-run requirements

For the coding-harness profile, select the external-log delivery tier. The native capture journal remains inside a surviving sandbox; the external run log is the client-delivery source. A delivery log cannot capture output lost upstream of its writer. [S9]

Use the same native durability adapter identity/configuration and native run binding across producing, joining, and recovery paths. Configure the native RunStore and durability together. Build fresh execution, recovery, cancellation, and reaping from one server-side resolved profile definition so they cannot select different agents or credentials.

Schedule the native reaper/finalizer and document runtime, abandonment, and output-retention policies. Do not count visible browsers as the lifecycle authority. A run can finish with zero readers and still needs history finalization and cleanup. [S10]

Distinguish browser/network recovery, backend-driver recovery with a surviving sandbox, and loss of the sandbox itself. Do not promise the last case is recoverable through the delivery log. Direct model adapters also do not acquire the sandbox journal guarantees merely by using resumable delivery.

Known upstream qualifications at review time:

- The detailed takeover reference acknowledges a residual stale-append race because the append interface lacks an atomic compare-and-set. Use the documented locking/lease controls and test ownership loss; do not advertise unconditional exactly-once execution or airtight fencing. [S5]
- The simplified durable-runs explanation overstates completion-sentinel secrecy. The journal reference and inspected source derive the nonce from runId and acknowledge deliberate reproduction is possible. Treat it as accidental-output disambiguation, not a cryptographic authentication boundary against a malicious agent. Do not invent another sentinel implementation. [S11]

Pin package versions and verify these properties in the installed source during implementation. No integration or failure-recovery tests were run in preparing this plan.

## 10. UI quality and collaboration feedback

Keep one central conversation with a compact member header, activity line, queued-request area, and sticky composer. Do not add a workbench around it.

Use author identity, not native message role alone, for attribution and alignment. Show readable names/avatars for other humans and a distinct assistant identity. Group consecutive messages carefully without hiding who asked a question or stopped the run.

Show controls in context: "Replying to Alex", "Your follow-up is queued", "Stopped by Kristjan", "Queue paused", "Reconnecting", and "Another participant started a new response". Only derive model/tool activity from actual events; never fabricate reasoning or percentages.

Keep connection, execution, and submission status separate. A disconnected phone is not proof the assistant stopped. On return from mobile suspension, restore and catch up instead of promising continuous background rendering.

Keep drafts private to the current user/device or tab according to an explicit local-draft policy. Never sync partially typed content as a shared room message. Clear private caches on sign-out/account change and isolate cache keys by identity and room. Synchronize only a typing signal, not draft text.

Basic presence should track connection/session leases and aggregate by human identity. Ten tabs for one person must not display ten members, and closing one tab must not mark the person offline while others remain. Expire stale typing/presence; do not replay an old typing event as if it were current. Avoid durable high-frequency heartbeat writes into the permanent transcript.

Use shadcn Message Scroller as the scrolling owner. Follow streamed output only while the viewer is at the live edge; respect deliberate upward scrolling, show a jump-to-latest control, and preserve anchors during history load or expansion. [S12]

Keep composer input responsive during long code streams. Render received output without a decorative typewriter backlog. Memoize completed messages, isolate active-message rendering, retain partial output on stop/error, and sanitize untrusted rendering. Support Enter/Shift+Enter, IME composition, accessible labels/focus, reduced motion, and mobile keyboard layout.

Local Send/Stop feedback target: within 100 ms on the reference device, measured separately from remote acknowledgment and actual termination. This is a proposed acceptance target, not a claim about library benchmarks.

## 11. Deployment and operational boundaries

Reference deployment: one long-lived application service, one PostgreSQL database, one persistent Durable Streams service, and the selected agent runtime. Keep implementation compatible with a second app replica through shared state and tested locking, but do not introduce a general workflow platform.

Require browser-facing HTTP/2 for the ten-tab deployment test. SSE over HTTP/1.x has a small browser/domain connection limit; test the negotiated protocol, proxy buffering, and actual behavior through the deployment origin. [S13]

Tailscale Serve may expose the app privately over HTTPS. [S14] Application authorization remains required. Keep durable-stream write/admin credentials server-side. Readers access authorized same-origin routes or equivalently scoped read capabilities. Derive upstream stream paths server-side from authorized IDs; do not accept arbitrary proxy destinations.

Bound queue size, message size, concurrent rooms per actor, run duration/cost, idle readers, and storage retention. Define deletion and retention separately from append-only delivery so old content is not accidentally retained forever. Snapshot/cursor handling must be implemented before claiming unbounded-history scalability; StreamDB's documented preload begins with stream materialization. [S2]

## 12. Acceptance suite

Use two genuinely separate authenticated browser contexts plus additional tabs. A mock username switch alone is not an authorization test.

| Scenario | Required outcome |
|---|---|
| Two users, ten tabs, one mobile | Same accepted history, authors, queue, and assistant attempt |
| Second user joins mid-response | Correct history followed by remaining live output, no duplicate text |
| Both users send at once | Stable server order and no overlapping assistant turns |
| Retransmit one timed-out command | Existing command is reconciled, not executed again |
| Idle tabs observe a later prompt | They discover the next turn without reloading |
| Participant cancels their queued item from mobile | Shared queue converges; someone else's input remains |
| Two viewers regenerate the same attempt | One transition wins; the other reconciles |
| Delayed Stop for A arrives after B starts | B is unaffected |
| Stop and replace during startup | No orphan process, duplicate run, or lost replacement |
| Initiating tab closes; all tabs close | Accepted work follows server policy and can finalize unseen |
| Mobile sleeps and returns | Catch-up succeeds and composer draft remains private |
| Driver restarts with sandbox surviving | Supported native recovery is exercised without launching another agent |
| Driver loses its claim | Native ownership-loss behavior is tested and limitations documented |
| DB commit succeeds while state stream is unavailable | Outbox eventually publishes without duplicate commands |
| Final status precedes final message projection | Partial answer remains visible until reconciled |
| Member is removed | Future commands/reads fail and active reads are revoked per the documented bound |
| Unauthorized stream/run ID is supplied | No history, live output, or status is disclosed |
| Two people have identical display names | Stable author identities and permissions remain distinct |
| Tool event arrives in ten clients | One server tool execution, no spectator side effects |
| User scrolls up while another sends | Reading position remains stable |
| Two users have different private drafts | No draft leakage or cross-account cache recovery |

Count model/harness starts and tool side effects in tests, not just final UI bubbles. A visually deduplicated transcript can conceal duplicate paid execution.

## 13. Implementation order

1. Integration proof: pin compatible packages; compile the native run stream, persistence, UI composition, and StreamDB room projection together. Demonstrate two authenticated readers and one real backend. Prove one history/stream ownership path before polish. Do not infer compatibility solely from similarly named packages.
2. Collaboration core: membership/invites, authorship, server command receipts, serial room admission, shared FIFO, and reliable publication. Seed the two-user demo through normal auth.
3. Interaction polish: fast composer, cancel/stop/replace/regenerate/edit semantics, activity feedback, attempts, scroller behavior, private drafts, and basic presence.
4. Durability and failure tests: reconnect, unseen completion, native reaper, driver restart where supported, cancellation races, outbox recovery, authorization/revocation, and real HTTP/2 ten-tab/mobile exercise.

Use deterministic native-event fixtures for UI/race tests and real-agent smoke tests for execution claims. Delivery of the template requires a verification report with pinned versions, passed tests, known limitations, and any unsupported selected-profile capability. Do not describe documentation inspection as runtime verification.

## 14. Sources and API verification references

These sources establish library behavior. The room model, policies, schema extensions, publication design, deployment defaults, and test criteria above are proposed application decisions.

- [S1] TanStack DB: https://tanstack.com/db/latest
- [S2] StreamDB: https://durablestreams.com/stream-db
- [S3] Durable State protocol and materialization: https://durablestreams.com/durable-state
- [S4] Durable Streams session integration: https://durablestreams.com/tanstack-ai
- [S5] Native takeover, cancellation, and ownership qualifications: https://tanstack.com/ai/latest/docs/sandbox/takeover
- [S6] Server-authoritative client persistence: https://tanstack.com/ai/latest/docs/persistence/client-persistence
- [S7] Native client queue: https://tanstack.com/ai/latest/docs/chat/queueing
- [S8] Lock capability and implementation contract: https://tanstack.com/ai/latest/docs/advanced/locks
- [S9] Durable run tiers: https://tanstack.com/ai/latest/docs/sandbox/durable-runs
- [S10] Reaping and retention: https://tanstack.com/ai/latest/docs/sandbox/reaping
- [S11] Run journal and nonce qualifications: https://tanstack.com/ai/latest/docs/sandbox/journal
- [S12] Message Scroller: https://ui.shadcn.com/docs/react/message-scroller
- [S13] SSE connection considerations: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- [S14] Tailscale Serve: https://tailscale.com/docs/features/tailscale-serve
- [S15] Native React UI composition: https://tanstack.com/ai/latest/docs/ui/react
- [S16] Native store reference: https://tanstack.com/ai/latest/docs/persistence/store-reference
- [S17] Native resumable streams: https://tanstack.com/ai/latest/docs/resumable-streams/overview
- [S18] Session transport README reviewed: durable-streams/durable-streams, packages/tanstack-ai-transport/README.md, blob 4c59c4f8dd08812138c627d798904ec8f09987e6.
- [S19] Journal source reviewed: TanStack/ai, packages/ai-sandbox/src/journal.ts, commit 44a73e0e8790f478d853bf3d843c44f1f501762e. The source explicitly qualifies the derived nonce's threat model.
