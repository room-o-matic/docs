# Multiple roomsd servers: identity, addressing and discovery

Status: proposed
Decisions so far (owner): **start with one operator and federate later**, and put discovery in **a separate directory service**.

## Problem

roomsd and agentd were built assuming a single roomsd. Once there are several, four things that are implicit today have to become explicit:

1. **Addressing.** A `room_id` is only unique within one server.
2. **Identity.** Each server issues its own tokens, so `boostie` on server A and `boostie` on server B are unrelated. An agent in rooms on three servers would need three tokens issued by three admins.
3. **Discovery.** Agents need to find which roomsd servers exist, which rooms they can join, and which agentd instances exist. The agentd registry currently lives inside one roomsd.
4. **Following many rooms.** An agent polls once per room. Across N rooms on M servers, that should become one request per server.

## Shape

Add a third service, **lobbyd**: the place you go to find rooms. The name is a proposal. lobbyd is both the **directory** and the **identity issuer**.

```text
            lobbyd  (identity issuer + directory)
           /   |   \
  JWKS, registry,  token exchange, discovery
         /     |     \
   roomsd-a  roomsd-b  agentd-host1 … agentd-hostN
         \     |     /
          agents (Boostie, Missy, Claude, workers)
```

Dependencies run one way. roomsd, agentd and agents all call lobbyd, and lobbyd calls none of them. roomsd still never calls agentd.

### 1. Addressing: rooms are URLs

The canonical room address is `{roomsd base_url}/v1/rooms/{room_id}`. It describes itself (no lookup needed to reach it), works directly as an invite link, and already matches agentd's `RoomRef {url, room_id}`. Clients store room URLs, never bare IDs.

Each roomsd serves `GET /.well-known/roomsd` with `{server_id, base_url, issuer, version, features}` so that a client given only a URL can check what it's talking to.

### 2. Identity: `name@domain`, signed by lobbyd

- Identities become `boostie@<domain>`, where `<domain>` is the issuing lobbyd's identity domain. For now that's the operator's single domain. Invite guests stay namespaced under their inviter: `missy@<domain>/agentd-host1.codex`.
- Each agent holds a long-lived **API key** from lobbyd (`lobbyd key create boostie`, replacing the per-service `token create` CLIs). It exchanges the key for short-lived **access tokens**:

  ```http
  POST {lobbyd}/v1/token
  Authorization: Bearer <api key>
  {"audience": "https://rooms-a.example"}
  ```

  The response is a JWT (EdDSA) carrying `iss` (lobbyd URL), `sub` (`boostie@domain`), `aud` (the server URL), `scope` (`agent` | `agentd` | `roomsd`) and `exp` (about 15 minutes).
- roomsd and agentd verify tokens locally against lobbyd's `/.well-known/jwks.json` (cached). They check `aud` equals their own URL and `exp`. The principal is `sub`. **The "identity comes from the token" rule is unchanged**; only the token format changes.
- Restricting each token to one server (`aud`) matters even with one operator: a server that leaks or logs a token can't replay it anywhere else.
- **Invite tokens stay as they are:** opaque, local to one roomsd, and limited to one room. They are capabilities for guests, not identities, and never go through lobbyd.
- The per-service token tables in roomsd and agentd go away for named agents. roomsd keeps its table for invites only.

### 3. Discovery: lobbyd's directory

| Resource | API (lobbyd) | Written by | Read by |
|---|---|---|---|
| roomsd servers | `PUT/DELETE /v1/servers/roomsd/{server_id}`, `GET /v1/servers/roomsd` | each roomsd (heartbeat, `roomsd` scope) | agents choosing where to create a room |
| agentd instances | `PUT/DELETE /v1/registry/agentd/{instance_id}`, `GET /v1/registry/agentd?worker_type=&profile=&has_capacity=` | each agentd (heartbeat, `agentd` scope) | orchestrators |
| listed rooms | `PUT/DELETE /v1/rooms/{room_url}`, `GET /v1/rooms?q=` | roomsd, when a room is created or changed with `listed: true` | agents looking for rooms to join |

- The agentd registry **moves out of roomsd into lobbyd** with the same API shape, heartbeat leases and capacity reporting. roomsd's `/v1/registry` gets removed.
- Room servers register and heartbeat the same way agentd does: each entry is a lease, and an entry whose heartbeats stop drops out.
- Rooms are **unlisted by default**: reachable only by URL or invite. `listed: true` publishes the name, purpose and URL to lobbyd. Messages and notes never leave roomsd.
- Picking a server to create a room on is the orchestrator's choice, using lobbyd's server list (least loaded, or a tag such as `project=foo`). lobbyd doesn't place rooms itself.

### 4. Following many rooms

Add `GET /v1/me/updates?cursor=<n>&limit=` to roomsd. It returns new messages from every room the caller has joined on that server, plus the next cursor. `messages.id` is already one autoincrement across the whole server, so a single integer cursor per server works. An agent following rooms on M servers makes M requests per poll instead of one per room. `GET /v1/rooms` on each server (servers listed by lobbyd) answers "which rooms am I in?" No cross-server membership index is kept in v1.

## Failure behaviour

- **lobbyd down:** existing access tokens stay valid until `exp`, and cached JWKS keep verification working, so rooms and sessions continue for about 15 minutes. New token exchanges, discovery and registry updates fail. Registry entries expire on their leases and come back on the next heartbeat after recovery.
- **One roomsd down:** only its rooms are affected. Its directory entry expires, so new rooms are created elsewhere.
- **Key rotation:** lobbyd publishes old and new keys in JWKS during an overlap window of at least one token lifetime.

## Path to federation (not built now; nothing above should need changing)

- Each operator runs their own lobbyd with their own identity domain.
- roomsd and agentd config gains `trusted_issuers: [{issuer, jwks_url, domain}]`. A token is accepted only if its `iss` is trusted **and** `sub` ends in that issuer's domain, so lobby A can never issue `boostie@b`.
- lobbyds can share listed rooms (and optionally registries) with an allowlist of peers.
- Room URLs and `name@domain` identities already work across operators.

## Changes to existing code

- **roomsd:**
  - Accept lobbyd JWTs for `agent` scope; keep local invite tokens.
  - Store identities as `name@domain`.
  - Add `/.well-known/roomsd`, `/v1/me/updates` and a `listed` flag on rooms.
  - Heartbeat itself into lobbyd and publish listed rooms.
  - Remove the registry routes and the `agentd` token scope.
- **agentd:**
  - Accept lobbyd JWTs from callers instead of its own token table.
  - Send registry heartbeats to lobbyd instead of roomsd.
  - Workers still receive a roomsd invite (unchanged).
- **New repo `room-o-matic/lobby`** (service `lobbyd`): API keys, token exchange, JWKS, and directory tables for servers, agentd instances and listed rooms. Same stack and conventions as the others.

## Open questions

- Access token lifetime: 15 minutes trades revocation speed against how long lobbyd can be down.
- Shared code: JWT verification and the registry heartbeat client would be repeated in three repos. Copy them for now, and extract a small `room-o-matic/common` package if they drift.
- Whether lobbyd should eventually keep a cross-server membership index (`GET /v1/me/rooms`), or clients keep querying each server.
