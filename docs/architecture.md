# Architecture — P2P VPN

> Living document. The end-state architecture is described here, but you'll only build pieces of it phase by phase.
> When something changes, update this file before you write the code.

---

## 1. What the system does

A peer-to-peer (mesh) VPN. Each user runs the same client. Users that share a **room code** can see each other's internal IPs (e.g. `10.8.0.x`) as if they were on the same LAN. Traffic between peers is encrypted with WireGuard. There is **no central relay** that carries traffic — only a tiny coordination server for peer discovery.

```
                    ┌─────────────────────┐
                    │  coord-server       │
                    │  (small VPS)        │
                    │  peer discovery     │
                    │  + STUN-like info   │
                    └──────┬──────┬───────┘
              register     │      │      register
              peers info   │      │      peers info
                    ┌──────┘      └───────┐
                    │                     │
              ┌─────▼──────┐         ┌────▼───────┐
              │  client A  │ ◄──────►│  client B  │
              │  (laptop)  │  direct │  (laptop)  │
              │            │  WG     │            │
              │  TUN: 10.8 │  tunnel │  TUN: 10.8 │
              │   .0.1     │         │   .0.2     │
              └────────────┘         └────────────┘
              behind NAT A           behind NAT B
```

The dashed lines (peer ↔ coord-server) carry only **metadata**: public keys, public endpoints, room membership. Real user traffic never passes through coord-server.

---

## 2. Components (Cargo crates)

The repo is a single Cargo workspace under `crates/`. Each crate has one clear responsibility.

| Crate | Kind | Responsibility | First appears in |
|---|---|---|---|
| `protocol` | library | Wire types shared by client and server: `JoinRequest`, `PeerInfo`, etc. Pure data + `serde`. No I/O. | Phase 4 |
| `core` | library | Tunnel logic: TUN interface management, packet routing, WireGuard glue, peer state. The brain. | Phase 1 (minimal) → grows each phase |
| `client` | binary | The user-facing daemon. Wires `core` to the OS, reads CLI args, talks to coord-server. | Phase 1 |
| `coord-server` | binary | HTTP server that maintains the room → peer registry. Stateless from client's perspective; in-memory or sled-backed. | Phase 4 |
| `gui` | binary | Tauri app. Talks to a local daemon (the `client`) via a Unix socket / Named Pipe. | Phase 6 |

**Why split?**
- `core` as a library means `client` and `gui` can both use it directly, and we can write unit tests against it without networking.
- `protocol` as a separate crate is what keeps client/server compile-time-compatible. Change a field, both fail to compile until you fix both.
- `coord-server` doesn't depend on `core` at all (no need — it never sees traffic).

---

## 3. Dependency graph

```
                  ┌──────────────┐
                  │   protocol   │  (pure data, serde)
                  └──┬────────┬──┘
                     │        │
            ┌────────▼─┐    ┌─▼─────────────┐
            │   core   │    │ coord-server  │
            │ (tunnel) │    │   (HTTP)      │
            └────┬─────┘    └───────────────┘
                 │
        ┌────────▼─────────┐
        │      client      │  (binary)
        └────────▲─────────┘
                 │ (local IPC, Phase 6+)
        ┌────────┴─────────┐
        │       gui        │  (Tauri binary)
        └──────────────────┘
```

Rules:
- Arrows point in the direction of `depends on`.
- `protocol` has zero internal deps. Any crate may use it.
- `core` depends on `protocol` (so it knows how to talk to coord-server) but **not** on `client` or `gui`.
- `coord-server` only depends on `protocol`.
- `client` glues everything OS-related: argument parsing, signals, daemonization.
- `gui` is the only one that depends on `client` — and even that is over IPC, not source.

---

## 4. Runtime architecture (processes & threads)

### Phase 1–5 (CLI only)
- One process: `client`.
- One Tokio runtime (`#[tokio::main]`), multi-threaded.
- Concurrent tasks within that runtime:
  - **Reader task:** reads from TUN, pushes packets to outbound queue.
  - **Writer task:** reads from inbound queue, writes to TUN.
  - **UDP I/O task(s):** owns the `UdpSocket`, demuxes by peer.
  - **Coord-server poller:** periodic `/heartbeat` and peer-list refresh.
  - **Timer task(s):** WireGuard handshake/keepalive timers per peer.

### Phase 6+ (with GUI)
- Two processes: privileged `client` daemon + unprivileged `gui`.
- IPC: Unix domain socket on macOS/Linux, Named Pipe on Windows. Length-prefixed JSON or `serde_json` over a framed stream.
- The daemon is auto-started at boot (`launchd` / `systemd --user` / Windows Service).

---

## 5. Packet life-cycle (data flow)

### Outbound (something on my machine wants to reach `10.8.0.2`)
```
1. App on my laptop opens connection to 10.8.0.2 (TCP/UDP/ICMP — doesn't matter).
2. OS routes the packet to interface tun0 (because 10.8.0.0/24 is routed there).
3. client reads the raw IP packet from tun0.
4. core looks up the destination IP in its peer table → peer B.
5. core feeds the packet into peer B's WireGuard state machine → produces an encrypted UDP payload.
6. client sends that payload over its UdpSocket to peer B's known endpoint.
```

### Inbound (encrypted UDP arrives from peer B)
```
1. UDP socket receives a datagram from B's public endpoint.
2. core identifies which peer this came from (by source endpoint and/or peer index in WG header).
3. core feeds it through that peer's WireGuard state → produces a plaintext IP packet.
4. client writes the plaintext packet to tun0.
5. OS sees the packet on tun0 and delivers it to whatever local app was listening on 10.8.0.1.
```

The "plaintext" packet on the inside is a complete IP packet (header + payload). It's just hidden from the rest of the internet by the encryption + UDP wrap.

---

## 6. Network architecture & connectivity

### Coord-server endpoints (Phase 4)
| Method | Path | Purpose |
|---|---|---|
| `POST` | `/rooms/:room/join` | Register `(pubkey, public_endpoint)`. Returns current peer list. |
| `GET`  | `/rooms/:room/peers` | List active peers (for refresh). |
| `POST` | `/rooms/:room/heartbeat` | Keep me alive (every 30s). |
| `DELETE` | `/rooms/:room/peers/me` | Leave. |

A peer is "active" if its last heartbeat was within 60s.

### NAT traversal (Phase 5)
- Each client runs a STUN query at startup to discover its public `(ip, port)`.
- That tuple is sent to coord-server via `/join`.
- When client A learns about client B's public endpoint, A starts sending WireGuard handshake packets to B's endpoint and listens on its own port. B does the same simultaneously.
- If both NATs allow it, the WG handshake completes within ~1–3 seconds and the tunnel is up.
- If hole punching fails (Symmetric NAT, CGNAT), the peer is marked unreachable. **No TURN relay** in the initial design — that is an explicit non-goal of v1.

### Encryption
- WireGuard via `boringtun`.
- Each client owns one long-lived X25519 key pair, persisted in `~/.config/p2p-vpn/key`.
- The pubkey is the peer's identity for the coord-server.
- Pre-shared room codes (PSK) are not used for encryption — they only authenticate **who can see the peer list of room X** at the coord-server.

---

## 7. Key design decisions

| Decision | Choice | Why |
|---|---|---|
| Async runtime | `tokio` (multi-thread) | Standard, well-supported by every networking crate we'll use. |
| VPN protocol | WireGuard via `boringtun` | Modern, audited, simple. Cloudflare's Rust impl saves us from writing crypto. |
| Discovery transport | HTTP/JSON (not gRPC, not custom) | Easy to debug with `curl`. No code-gen step. |
| Persistence on coord-server | `sled` (embedded KV) | No external DB. Survives restarts. SQLite is also fine if you prefer SQL. |
| Persistence on client | Plain TOML config + binary key file | Human-readable; key file is `chmod 600`. |
| Logging | `tracing` + `tracing-subscriber` | Structured, async-friendly, levels and spans. Replace `println!` from Phase 4 onward. |
| Error strategy | `anyhow` in binaries, `thiserror` in libraries | Standard Rust convention. Library users want typed errors; main fns just want context. |
| TUN crate | `tun` (Linux/macOS), `wintun` (Windows) | The most maintained options. |
| GUI | `tauri` + Svelte/Solid | Small bundle, Rust backend, web frontend. |
| Privilege model | Daemon = privileged, GUI = unprivileged, IPC over local socket | Same model as Tailscale. Smaller attack surface. |
| Repo layout | Cargo workspace, one binary per process | Standard Rust convention. Shared `Cargo.lock` and `target/`. |

---

## 8. How the architecture evolves through phases

| Phase | What's added | What changes |
|---|---|---|
| 1 | `client` reads from TUN | Skeleton workspace + `client` binary; `core` may not even exist yet — start with logic in `main.rs`, extract later. |
| 2 | UDP socket + outbound/inbound tasks | Switch from sync to `tokio` runtime. Static peer config (CLI args). |
| 3 | `boringtun` integration | First time `core` becomes a real library. WireGuard timers added. |
| 4 | `coord-server` + `protocol` | Two more crates. Static peer config replaced by dynamic discovery. |
| 5 | STUN + hole punching | Adds `stun` dependency. Peer endpoint becomes dynamic and may change at runtime. |
| 6 | `gui` + IPC | Splits the binary in two. Daemon survives independently of GUI. |
| 7 | Tracing, metrics, tests, CI | No new components — only quality. |

You don't need to design for the final architecture today. Each phase asks for a refactor; that refactor **is** part of the learning.

---

## 9. Threat model (v1, intentional limits)

In scope:
- Eavesdropping on user traffic between peers — prevented by WireGuard.
- Tampering with traffic between peers — prevented by WireGuard's AEAD.
- Coord-server compromise — attacker can see metadata (who is in what room, public endpoints, public keys) but **cannot read user traffic**.

Out of scope (for now — flagged for later):
- Forward secrecy beyond what WireGuard provides natively.
- Hiding metadata from coord-server (would require an anonymous overlay; way out of scope).
- Sybil attacks on rooms (anyone with the code joins). Mitigated by long random codes; full membership control is post-v1.
- Malicious peer rooms (peer claims `10.8.0.1` and intercepts). Mitigated by pinning peer-IP-to-pubkey on the client side.

---

## 10. Open questions

These are decisions worth making once you reach the relevant phase. Park them here so you don't forget.

- [ ] How does `client` get its TUN address inside the room? Static (config) or assigned by coord-server?
- [ ] What happens when two peers claim the same internal IP?
- [ ] Should peer keys rotate? Manual or automatic?
- [ ] Should we support IPv6 inside the tunnel? (Probably yes — WireGuard supports it natively.)
- [ ] Pricing model for the coord-server when scaling? (Free Tier should hold thousands of peers — measure first.)
- [ ] Rate-limit `/join` to prevent abuse?

Update this section as you decide.
