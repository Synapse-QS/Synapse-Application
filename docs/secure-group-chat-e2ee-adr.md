# ADR 0001: Secure group chat and collaboration — E2EE architecture

**Status:** Phase 0 decision recorded (autonomous engineering recommendation); **NO-GO** for production implementation or activation. **Selected protocol direction:** IETF MLS (RFC 9420). **Lead library candidate for qualification:** OpenMLS. **Production library:** none selected. This is not a security audit of Synapse's integration or a claim that current E2EE paths are safe.

**Repository snapshot reviewed:** main at e5020efeddacc44a6e7a4c00964178052140e1d0. The ADR separates observed repository state from roadmap requirements and external library documentation.

## Bounded Phase 0 plan

1. Inspect repository guidance, crypto/data-flow/storage code, tests, and migration evidence.
2. Compare standards-based group-protocol candidates and record platform support evidence and blockers.
3. Classify private payloads versus operational metadata and set explicit no-go criteria.
4. Leave application crypto, database schema, authorization policies, and deployment unchanged; create no PR.

## Decision

- Select the IETF MLS standard (RFC 9420) as the group-protocol direction. Its asynchronous group key establishment, forward secrecy, and post-compromise security fit the membership/epoch problem better than the current per-recipient 1:1 approach.
- Select OpenMLS as the lead library candidate for qualification only, not as an approved production dependency. SRLabs completed an independent assessment of core OpenMLS crates, but the report explicitly excludes the cryptographic and storage providers and the proof-of-concept client/delivery service; it therefore does not audit Synapse's native bindings, provider/storage implementations, delivery service, or app integration. The OpenMLS README lists Android/iOS as unsupported-but-built, not tested supported targets. The user's no-tests constraint prevents the mobile interop/adversarial verification needed to close that gap.
- Keep mls-rs as a secondary candidate only if OpenMLS cannot meet the integration requirements. Its UniFFI wrapper is evidence of generated-language bindings, not production mobile support; its README says it has not received a full third-party security audit.
- Do not enable new E2EE group content or begin dependent Phase 1 implementation until the selected release, platform integration, backend boundaries, and security verification are qualified.
- Record this as an evidence-based blocker and stop dependent rollout. Reopen library selection only after a candidate demonstrates maintained, supported Android+iOS artifacts, storage/FFI behavior, license approval, and independent review scope for the exact shipped build.
- Evaluate OpenMLS and mls-rs as candidate implementations. Require supported, reproducible Android/iOS integration, secure platform storage, exact-version maintenance commitments, interoperability evidence, license approval, and independent security review of the actual native/FFI build before production adoption.
- Do not reuse the checked-in iOS crypto implementation, invent cryptographic primitives, or silently fall back to plaintext. Existing Android Signal code is not evidence of a supported MLS group implementation.
- Initial secure-group support is limited to Android and iOS. Web, desktop/JVM, and wasm clients must not send or receive new private group content until separately qualified. UI warnings alone do not satisfy this gate.
- Preserve existing history as readable, explicitly marked pre-E2EE data. Encrypt new private content only after supported active member devices are ready. Pause private sending on any key, verification, or rotation failure; no plaintext downgrade.

## Threat model and trust boundaries

| Threat / asset | Required property | Consequence for design |
| --- | --- | --- |
| Curious or compromised transport/backend | Backend must not learn message or attachment plaintext; alteration must be detectable | Encrypt and authenticate on-device; server handles only reviewed routing/operational fields and protocol ciphertext. |
| Malicious server or identity substitution | Server cannot silently replace a member identity or key package | Define identity verification, authenticated credential binding, key-change warnings, and any transparency/audit mechanism before rollout. |
| Removed or compromised member/device | Removed member cannot decrypt subsequent messages; copied old data cannot be remotely erased | Authenticate membership changes and rotate the group epoch before the next message. State clearly that previously copied plaintext/keys cannot be revoked. |
| Replay, reordering, duplicate/forked membership changes | Old or conflicting state must not be accepted as current | Specify epoch/state validation, replay handling, concurrent-commit behavior, and stale-client recovery; qualify cross-platform behavior. |
| Lost device, reinstall, backup | Server must never receive a user-controlled recovery/decryption key | Define device revocation, encrypted backup, recovery-key lifecycle, and user-visible loss limitations before implementation. |
| Metadata, pushes, logs, local cache, media host | Minimize leakage beyond ciphertext, including indirect plaintext copies | No plaintext/key material in logs or push payloads; review local cache at rest and media object access. |
| Vulnerable or unsupported dependency/native bridge | App must not rely on an unreviewed or silently changing crypto surface | Pin versions/providers, track advisories, license/provenance, platform slices, and upgrade ownership; review the FFI boundary. |

Out of scope for cryptographic guarantees: a compromised/unlocked endpoint, screenshots or deliberate exports by a recipient, and all traffic-analysis metadata. Communicate these residual risks to users.

## Data classification (initial proposal; validate against the actual backend)

The current DTOs and a read-only inspection of live public-schema metadata informed the fields below; no application rows were queried. This is a target classification, not proof that the policies enforce every property listed.

| Current field/category | Target handling for new secure groups | Server visibility / notes |
| --- | --- | --- |
| Message content and structured metadata (content, metadata) | Encrypt/authenticate as application data | Ciphertext only. Include replies and rich-message semantics unless a reviewed feature requires otherwise. |
| Reply relation, message type, captions, filenames, attachment descriptors | Encrypt where not strictly required for delivery | Do not expose semantic type, reply target, filename, or captions by default. Justify any retained routing hint as leakage. |
| Attachment bytes, media URL, attachment URL/type/size/duration/group ID | Encrypt bytes client-side before upload; protect keys and descriptors in the message envelope | Storage/CDN receives ciphertext and minimum opaque object-routing data. Public third-party URLs are not an acceptable encrypted-storage substitute. |
| Group name, description, avatar/photo, roster, and user-facing membership state | Encrypt where feasible; define a minimal authorization/routing projection | Backend may need a minimal membership/ACL projection for authorization. Approve its fields and leakage; UI-only role checks are insufficient. |
| Opaque conversation/message IDs, sender/device routing, protocol version, ciphertext, delivery/expiry/edit/delete state, delivery/read timestamps, and content state | Server-readable only where operationally necessary; minimize and document | Can reveal social graph, activity, timing, group size, and user behavior. Review access and retention. |
| Edit-history content and revisions | Encrypt like message content; historical plaintext remains explicitly pre-E2EE | The live schema includes a message_edit_history content field. |
| Reactions, forwards, per-user deletion markers, and poll-vote records | Treat semantic choices and recipient/user associations as private; keep new poll/vote behavior out of Phase 1 until its separate encrypted design | The live schema includes reaction, forward, deletion-marker, and poll-vote tables; Phase 2 polls/voting remain out of scope. |
| Public identity credentials / MLS key packages | Public material may be stored for routing, but must be authenticated and bound to a verified device identity | Private keys and recovery keys never go to the server. Key substitution and device revocation remain critical. |
| Existing legacy messages | Keep readable and visibly marked pre-E2EE; do not silently rewrite history | Server may continue to hold historical plaintext. Cutover and history-sharing policy must be explicit to joining members. |
| Push notifications, local decrypted cache, crash/log data | Generic/private-safe push only; define protected local caching and deletion | Current SQLDelight cache stores message content; the iOS crypto class uses multiplatform Settings. At-rest protections need resolution. |

The classification is based on MessageDto, ChatDto, ChatParticipantDto, and MessageAttachmentDto. Confirm every persisted, realtime, RPC, storage, and notification field against the actual Supabase schema before changing it.

## Candidate evaluation (evidence reviewed; no production choice)

| Candidate | Evidence and fit | Decision / unresolved evidence |
| --- | --- | --- |
| MLS, RFC 9420 | IETF Proposed Standard for asynchronous group key establishment with forward secrecy and post-compromise security. It fits group/epoch/removal better than treating group chat as unrelated 1:1 sessions. | Lead protocol for evaluation only. Define credential binding, delivery service, storage, interop profile, metadata policy, and migration before approval. |
| OpenMLS | Maintained Rust RFC 9420 implementation (MIT). SRLabs published a public assessment (report v1.2, March 11, 2026) of the openmls, traits, and basic_credential crates, reviewing repository commit a3402f2 from October 22, 2025. It found eight issues, including one High; the report excludes crypto providers, storage providers, and the proof-of-concept client/delivery service. The maintainer's May 27, 2026 summary says fixes for seven findings were published in OpenMLS 0.8.1 and 0.7.3, while one Low issue was still being addressed. Android/iOS targets remain listed as unsupported-but-built, not tested. | Preferred candidate for qualification. Confirm the exact release/fixes, open finding status, maintained mobile packaging, and app-level provider/storage/native bridge behavior; the library audit is not an audit of Synapse's integration. |
| mls-rs | Rust RFC 9420 implementation (repository metadata lists Apache-2.0 and MIT licenses); README describes interoperability/security-focused tests. An mls-rs UniFFI wrapper exists and describes Kotlin/Swift binding generation. Its README explicitly says it has not received a full third-party security audit. | Candidate for qualification only, not production selection. Confirm supported mobile releases/artifacts, provider/storage behavior, version stability, audit scope, and Android↔iOS interoperability for the exact build. A wrapper's existence alone is not a support commitment. |
| Current Signal stack | Repository pins org.whispersystems:signal-protocol-android:2.8.1; its API models per-recipient sessions. Current libsignal repository metadata lists AGPL-3.0. Its README says it is used by official Signal clients, external use is unsupported, and APIs may change; legal review is required. The older signal-protocol-java repository is archived. | Not an assumed cross-platform group/MLS solution. Reuse for legacy 1:1 compatibility needs a separate support, license, maintenance, and migration decision. |
| Current iOS implementation | Checked-in code returns the literal MOCK_SIGNATURE, stores key material through Settings, and uses a custom shared-secret/AES path instead of the Android Signal implementation. | Rejected for private content. Replace only with an approved vetted protocol/library; do not patch or extend bespoke crypto. |

An independent OpenMLS library assessment is evidenced below; it is not an audit of this app or of its provider, storage, FFI, persistence, backend, or mobile integration. The mls-rs README states that mls-rs has not received a full third-party security audit. Upstream library reviews do not replace app-level security qualification.

## Repository findings and blockers

- SignalProtocolManager exposes recipient-oriented operations and no MLS group/epoch API. The existing group send path encrypts per participant; that is not proof of secure group membership evolution or key rotation.
- Android DI supplies AndroidSignalProtocolManager, but the iOS KMPHelper constructs SupabaseChatRepository with a null manager. Desktop DI also supplies null. SendMessageUseCase explicitly sends plaintext when its manager is null. The web inbox identifies itself as not E2EE and still has a send handler. These paths must be gated before secure-group rollout.
- The iOS class contains a mock signed-pre-key signature and home-grown crypto; it is not interoperable with the Android Signal implementation. The Android dependency is the older archived Signal Protocol Android artifact, not the current libsignal package.
- Current message/group DTOs show plaintext-capable fields for content, metadata, media URLs, attachments, names, descriptions, avatars, participants, roles, and delivery/read state. The message decryption helper treats non-envelope content as plaintext, so the new format must explicitly distinguish legacy plaintext from authenticated versioned ciphertext.
- Decrypted content is persisted in the SQLDelight message cache. The active Android Signal store uses EncryptedSharedPreferences, but its isTrustedIdentity implementation replaces a changed identity key and returns true; this is not verified identity-change handling. The iOS manager uses multiplatform Settings. Define secure key/state storage, identity-change verification, local-cache policy, logout/device-removal deletion, and backup behavior.
- No E2EE-focused tests were found in the inspected common-test paths. Per the task instruction, no tests will be added or run for this documentation-only Phase 0 change. This does not satisfy the roadmap's later security verification gates and is not approval to enable E2EE.
- Read-only backend metadata snapshot (no table rows queried, no DDL): the live public schema has chats, chat_participants, and messages with RLS enabled (not FORCE); observed policies restrict reads to participants and writes by membership/sender conditions. user_public_keys exists and is publicly readable for key distribution; signal_identity_keys and signal_one_time_pre_keys do not exist despite references in the legacy SignalRepository. The database reports 35 recorded migrations, latest chat_message_poll_votes. The repo's supabase/ directory contains only a deployment note; its README says previous remote migration/RLS test files were removed and the SQL RLS harness was not run against production. Remote migration history is not a substitute for tracked, reproducible SQL. Policy behavior was inspected statically, not validated adversarially. No schema or production data changes were made.
- CONTRIBUTING.md requests branches from develop, but the connected repository currently lists only main; this work is isolated on a feature branch from main. No issue assignment, PR, merge, deployment, or release is performed.

## Migration and lifecycle requirements

1. Introduce an explicit versioned ciphertext envelope with authenticated protocol/version and content-kind handling. Never infer legacy plaintext merely because parsing/decryption failed.
2. Preserve and label existing messages as pre-E2EE. Do not automatically re-encrypt historical records.
3. For existing-group cutover, pause private sends until all supported active devices establish and verify required state. Encryption, identity verification, membership commit, or key rotation failure must fail closed.
4. A removal must be committed and keys rotated before any later group message. Added members receive only history permitted by the approved none/7-day/30-day policy, shared client-side under encryption.
5. Encrypt attachment bytes before upload and keep decryption keys client-side. Specify encrypted object deletion/expiry and how URLs, sizes, and MIME/type hints leak metadata.
6. Keep recovery keys user-controlled; server-side backup stores ciphertext only. Document that revocation cannot erase content already copied by a lost device.
7. Enforce roles and membership operations at trusted backend/data boundaries. Before any schema change, inspect the actual migration workflow, live schema, RLS policies, RPCs, and relevant tests; add versioned migrations and authorization tests only after that workflow is verified.

## Go / no-go gate before any Phase 1 implementation is enabled

- [ ] Product/security owners approve the threat model, data classification, protocol profile, device identity and membership-authentication model, metadata leakage, and recovery policy.
- [ ] Select a maintained library/version/provider with documented Android/iOS support, license approval, artifact provenance, upgrade ownership, and independent security review covering the exact build and FFI surface. No custom cryptography.
- [ ] Android↔iOS interoperability and adversarial protocol verification cover tampering, identity substitution, replay/reordering, stale/forked membership changes, add/remove rotation, multi-device state, restore/recovery, and failure handling. The current no-tests constraint means this gate cannot be reported as passed.
- [ ] No iOS, web, desktop/JVM, or wasm path can access new private group content without a supported verified implementation. No null-manager plaintext fallback remains on the private-content path.
- [ ] Attachments, metadata, group profile fields, roster, notifications, caches, logs, and local secret storage are reviewed against the classification above.
- [ ] Before any schema change, reconcile tracked migration source with the live schema, review relevant RLS/RPC behavior, and complete focused authorization verification. The current no-tests constraint leaves this gate unpassed.
- [ ] Legacy history is visibly marked pre-E2EE; cutover readiness and added-member history policy are enforced; complete diff and review evidence are recorded.

**Current gate result: NO-GO.** MLS is the protocol direction and OpenMLS is the lead qualification candidate, but no production library is approved. The audit scope, untested Android/iOS targets, missing app integration evidence, live authorization findings, and no-tests constraint stop dependent Phase 1 rollout. Do not substitute plaintext or custom cryptography.

## Primary references

### Repository evidence

- [Roadmap issue #11](https://github.com/synapseSRC/synapseApp/issues/11) and [Phase 0 issue #12](https://github.com/synapseSRC/synapseApp/issues/12)
- [AGENTS.md](../AGENTS.md), [REVIEW.md](../REVIEW.md), [CONTRIBUTING.md](../CONTRIBUTING.md)
- [Shared crypto dependency declarations](../shared/build.gradle.kts), [SignalProtocolManager](../shared/src/commonMain/kotlin/com/synapse/social/studioasinc/shared/data/crypto/SignalProtocolManager.kt)
- [Android manager](../shared/src/androidMain/kotlin/com/synapse/social/studioasinc/shared/data/crypto/AndroidSignalProtocolManager.kt), [active Android Signal store](../shared/src/androidMain/kotlin/com/synapse/social/studioasinc/shared/data/crypto/store/AndroidSignalStore.kt), [iOS manager](../shared/src/iosMain/kotlin/com/synapse/social/studioasinc/shared/data/crypto/IosSignalProtocolManager.kt)
- [Shared send use case](../shared/src/commonMain/kotlin/com/synapse/social/studioasinc/shared/domain/usecase/chat/SendMessageUseCase.kt), [iOS construction path](../iosApp/iosApp/Chat/Models/KMPHelper.swift), [desktop DI](../desktop/src/jvmMain/kotlin/com/synapse/social/studioasinc/desktop/di/DesktopModule.kt)
- [Chat DTOs](../shared/src/commonMain/kotlin/com/synapse/social/studioasinc/shared/data/dto/chat/ChatDtos.kt), [local cached-message schema](../shared/src/commonMain/sqldelight/com/synapse/social/studioasinc/shared/data/database/CachedMessage.sq), [Supabase deployment note](../supabase/README.md)
- [Web chat warning](../web/src/features/home/ChatThread.tsx), [web inbox](../web/src/features/inbox/InboxScreen.tsx)

### External primary evidence

- [RFC 9420 — The Messaging Layer Security (MLS) Protocol](https://www.rfc-editor.org/rfc/rfc9420)
- [OpenMLS README](https://github.com/openmls/openmls#readme) and [OpenMLS project site](https://openmls.tech/)
- [OpenMLS SRLabs assessment announcement](https://blog.openmls.tech/posts/2026-05-27-independent-audit/), [full SRLabs report (PDF)](https://blog.openmls.tech/SRL-OpenMLS_security_assurance_assessment.pdf), and [OpenMLS v0.8.1 release](https://github.com/openmls/openmls/releases/tag/openmls-v0.8.1)
- [mls-rs repository/README](https://github.com/awslabs/mls-rs) and [mls-rs UniFFI wrapper source](https://github.com/awslabs/mls-rs/blob/main/mls-rs-uniffi/src/lib.rs)
- [Signal libsignal README](https://github.com/signalapp/libsignal#readme)
- [Archived Signal Protocol Java repository](https://github.com/signalapp/libsignal-protocol-java)

## Verification record

Documentation-only change. No application source, tests, dependency versions, database schema, authorization policy, or deployment was changed. No tests were added or run, as instructed. This draft records current blockers and is not a completed independent security review.
