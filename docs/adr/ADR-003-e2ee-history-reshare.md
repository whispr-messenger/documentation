# ADR-003 — E2EE history re-share for newly-added devices

**Status:** Accepted (initial implementation on `deploy/preprod2` for the
devzeyu self-host, 2026-05-28).

**Stakeholders:** auth-service, messaging-service, mobile-app.

---

## Context

Whispr encrypts every message end-to-end with a Signal-style envelope:

```
{
  v: 1,
  t: "whispr_e2ee_v1",
  conversation_id: <uuid>,
  sender: { user_id, device_id, identity_key },
  cipher: { nonce, box(plaintext, msgKey) },
  key_packets: [
    { user_id, device_id, nonce, box(msgKey → recipient_pub) },
    …
  ]
}
```

The per-message key `msgKey` is encrypted once for each recipient device
the sender knows about at send time. The server stores the envelope as
opaque ciphertext; only a device whose private identity key matches one
of the `key_packets` entries can recover `msgKey` and decrypt the cipher.

This works perfectly while the recipient set is stable. It breaks the
moment a user installs Whispr on a brand-new browser or phone:

1. The new device generates a fresh Curve25519 identity keypair and
   uploads the public key to auth-service.
2. The user opens an existing conversation.
3. Every historical message was encrypted *before* this device existed,
   so none of their `key_packets` targets it.
4. The new device has no way to recover `msgKey` and every old message
   shows up as **Message non disponible**.

The user can read messages sent *after* their new device registered
(senders enumerate the recipient's device list at encrypt time), but
historical visibility on the new device is broken.

This ADR documents the chosen recovery mechanism — **key-packet
re-share initiated by an existing device of the same user** — and the
endpoints / data model that implement it.

---

## Decision

**Add a per-recipient-device key_packet side-channel** the user's
existing devices can populate without the server ever learning the
underlying plaintext.

Three components:

1. **`message_reshares` table** (messaging-service Postgres).
2. **`POST /messaging/api/v1/messages/reshares`** + read-path JOIN.
3. **Client `ReshareService`** that runs on cold-start and back-fills
   packets for any of the user's other devices it hasn't yet covered.

Re-shares are initiated by devices that already hold `msgKey` (because
they were in the original `key_packets`). They decrypt `msgKey` locally,
re-encrypt it for the new device with their own identity secret + the
new device's identity public, and POST the opaque packet to the server.

### Why this approach

| Property | This design | Alternatives |
|---|---|---|
| Server never sees plaintext | ✓ | A naive server-side rewrite would have to know `msgKey` |
| Forward secrecy of the cipher | ✓ | Per-message `msgKey` stays unchanged; only the wrapping is new |
| Idempotent | ✓ via unique `(message_id, recipient_device_id)` | A queue/stream design would be order-sensitive |
| Works with the existing envelope schema | ✓ — adds a sibling `reshare_key_packets` array on read | Modifying `key_packets` in place would require re-parsing the stored ciphertext on the server |
| Recoverable history without a passphrase | ✓ — only requires another device of the user to come online | Server-side encrypted backup requires the user to remember a recovery code |
| Recoverable history when all old devices are lost | ✗ — fundamental limit | A future passphrase-derived backup (ADR-004 once written) covers this case |

---

## Threat model

- **Server compromise** does not yield plaintext. Re-share packets are
  Curve25519 ECDH boxes; the server holds neither party's secret.
- **A compromised device cannot mint reshares it isn't allowed to**.
  The endpoint pins `resharer_device_id` to the device extracted from
  the caller's JWT (`deviceId` claim), so a stolen access token cannot
  impersonate a different device.
- **Replay / double-publication** are absorbed by the unique
  `(message_id, recipient_device_id)` index — the endpoint uses
  `INSERT … ON CONFLICT DO NOTHING`.
- **Adding a malicious new device** still requires logging in as the
  user (handled by auth-service). Re-share is then governed by the
  existing trust boundary: any existing device, on next cold-start,
  will re-share to it. Users who want stricter control can deactivate
  a device via the existing `/auth/devices` flow.
- **The only thing leaked to the server** is which device re-shared
  for which message — i.e. activity metadata, not content.

---

## Data model

### `message_reshares` (Postgres / messaging-service)

```text
id                   uuid PK
message_id           uuid FK -> messages.id ON DELETE CASCADE
recipient_user_id    uuid (denormalised; lets the read path filter
                          without joining auth)
recipient_device_id  uuid
nonce                text  -- base64 (24 bytes Curve25519 nonce)
box                  text  -- base64 (NaCl box of msgKey, 32 bytes)
resharer_device_id   uuid  -- audit: which device produced this packet
inserted_at          timestamptz

UNIQUE (message_id, recipient_device_id)
INDEX  (recipient_device_id, message_id)
```

Migration: `priv/repo/migrations/20260528170000_create_message_reshares.exs`.

---

## API contracts

### `POST /messaging/api/v1/messages/reshares` (auth required)

Bulk-publish reshare packets. The caller's `device_id` (from the JWT
`deviceId` claim) is the resharer; the body never overrides it.

```json
{
  "reshares": [
    {
      "message_id": "<uuid>",
      "recipient_user_id": "<uuid>",
      "recipient_device_id": "<uuid>",
      "nonce": "<base64>",
      "box": "<base64>"
    },
    …
  ]
}
```

Responses:
- `201 { inserted: N }` on success (`N` may be < input size because of
  conflict skipping).
- `400 { error: "Missing device identity" }` when the JWT didn't carry a
  `deviceId` claim.
- `401` on bad auth.

### `GET /messaging/api/v1/conversations/:id/messages` (existing)

The response shape gains one optional field per message:

```jsonc
{
  "id": "<uuid>",
  …,
  "reshare_key_packets": [
    {
      "user_id": "<uuid>",
      "device_id": "<uuid>",
      "nonce": "<base64>",
      "box": "<base64>"
    }
  ]
}
```

The array contains only packets addressed to the **calling device** (we
filter by `recipient_device_id = conn.assigns.device_id`). For devices
that were a recipient on the original envelope, this is always empty.

---

## Client logic

### Decryption (E2EEService)

`decryptTextMessage` now takes an optional `reshareKeyPackets` argument.
The decrypt loop:

1. Look up self in `envelope.key_packets`. If found → decrypt with it.
2. Otherwise look up self in `reshareKeyPackets`. If found → decrypt
   with it.
3. Otherwise return `null` (UI surfaces "Message non disponible").

Both branches use the same `nacl.box.open(box, nonce, senderPub,
ownSecret)`; the only difference is which packet supplies `box` and
`nonce`.

### Re-share orchestration (ReshareService)

On cold-start (after the existing identity-key self-heal in
`AuthContext`):

1. Fetch the user's device list from
   `auth-service GET /signal/keys/:userId/devices`.
2. Diff against a locally persisted "already re-shared to" set
   (`whispr.signal.reshared.to` in storage).
3. For each target device, fetch its `identity_key` from auth-service.
4. Walk all conversations + messages we can already decrypt. For each
   message:
   1. Find our own `key_packet`.
   2. `nacl.box.open` to recover `msgKey`.
   3. For each target device, `nacl.box(msgKey, nonce,
      targetPub, ownSecret)` and queue an entry.
5. POST batches of 25 to `/messages/reshares`. The server
   `INSERT ... ON CONFLICT DO NOTHING` makes this idempotent.
6. Persist the target device ids to the "already re-shared to" set so
   we don't repeat the walk next time.

### UI affordance

The conversations list surfaces a one-line banner while
`ReshareService.hasUnresharedDevices` is true:

> **Pour récupérer l'historique chiffré sur cet appareil, connectez-vous
> au moins une fois sur l'un de vos autres appareils Whispr. Il
> republiera automatiquement les clés ici.**

The banner polls every 30s and disappears once the other device runs
its own cold-start pass.

---

## Limitations

- **All prior devices lost.** If every device that ever held `msgKey`
  is gone, the ciphertext cannot be recovered. This is a hard
  cryptographic property of the envelope design — the server has no
  ability to help. ADR-004 (TBD) will introduce passphrase-derived
  encrypted backup to cover this case.
- **The other device must come online at least once.** Re-share runs in
  the device's cold-start hook. Until that happens, the new device sees
  the banner.
- **No throttling yet.** A user with hundreds of conversations and many
  devices could publish O(conversations × messages × new_devices)
  packets at startup. Acceptable for the self-host target; production
  deployments should add per-device-per-conversation backoff and a
  server-side rate limit.
- **Bulk re-share is unaware of moderation deletions.** Reshares for
  tombstoned messages are still produced. They're harmless (decrypt
  succeeds, the bubble still shows the deletion tombstone) but they're
  wasted I/O.

---

## File index

### messaging-service
- `priv/repo/migrations/20260528170000_create_message_reshares.exs`
- `lib/whispr_messaging/messages/message_reshare.ex`
- `lib/whispr_messaging/messages.ex` — `insert_reshares/2`,
  `list_reshares_for_device/2`
- `lib/whispr_messaging_web/controllers/reshare_controller.ex`
- `lib/whispr_messaging_web/controllers/message_controller.ex` —
  `render_message/2` now takes a `reshare_packets` arg; `index/2`
  pre-loads them via `Messages.list_reshares_for_device/2`
- `lib/whispr_messaging_web/plugs/authenticate.ex` — now extracts
  `deviceId` from the JWT and assigns it to `conn.assigns[:device_id]`
- `lib/whispr_messaging_web/router.ex` — `POST /messages/reshares` route

### mobile-app
- `src/services/E2EEService.ts` — `decryptTextMessage` accepts
  `reshareKeyPackets`; `loadOwnIdentityKeypair` exposed for ReshareService
- `src/services/messaging/api.ts` — `postReshares` client
- `src/services/ReshareService.ts` — cold-start orchestrator
- `src/services/storage.ts` / `.web.ts` — uses existing secure storage
  for the "reshared to" set (`whispr.signal.reshared.to`)
- `src/context/AuthContext.tsx` — kicks off
  `ReshareService.runForCurrentUser` on cold-start
- `src/screens/Chat/ConversationsListScreen.tsx` — banner

### deploy
- `overlays/devzeyu-selfhost/messaging-service/deployment.yaml` — points
  to the locally-built image with the new endpoint
  (`docker.io/whispr-messenger/messaging-service:devzeyu-reshare`,
  `imagePullPolicy: Never`).

---

## Test plan

- **DB migration** runs cleanly on a fresh `messaging_service_db`.
- **Idempotency:** posting the same reshare twice in a row leaves a
  single row.
- **Authn:** request without a Bearer token → 401; request with token
  but no `deviceId` claim → 400.
- **End-to-end:** with two devices A (existing) and B (new),
  - B logs in, opens a conversation → all historical messages show
    "Message non disponible" + the banner is up.
  - A is opened (cold-start). ReshareService publishes packets.
  - B reloads the conversation. Messages now decrypt successfully and
    the banner disappears within 30s.
- **Lost-device fallback:** with B as the only remaining device, every
  historical message stays undecryptable (expected). Documented as a
  limitation; ADR-004 will lift it.
