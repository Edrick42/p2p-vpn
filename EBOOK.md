# Building a P2P VPN in Rust — From Zero to Working

> A project-driven learning guide.
> You'll write every line of code. I give you the map.

---

## Preface

### Who this guide is for
You can program in some language (any will do), but you're new to Rust. You want to learn Rust **for real** — not through "fizzbuzz" tutorials, but by building something concrete, with everything that comes with it: cryptic compiler errors, painful refactors, and "aha!" moments.

### The project
A **P2P (mesh) VPN** in Rust. Each user downloads the app, generates a key pair, joins a "room", and connects directly to other users in the same room — no central server paying for bandwidth.

### Why this project is great for learning Rust
Rust shines at **systems programming**: low-level networking, cryptography, performance. Building a VPN forces you to:
- Work with raw bytes (you'll feel ownership in your bones)
- Work async (you'll master `tokio`)
- Integrate complex libraries (`boringtun`, `axum`, `tauri`)
- Think about security (impossible to ignore)
- Package across platforms (Cargo shines here)

### How to use this guide
1. Each **Phase** is an independent milestone — you run it, see it work, celebrate.
2. Each phase has: **Goal · Rust concepts · Crates · Tasks · Done criteria · Common pitfalls**.
3. **Don't skip phases.** Each one builds vocabulary for the next.
4. When you get stuck, come back here. If you're still stuck, ping me (Claude) with the specific phase.

### What you'll have at the end
- A desktop app (macOS/Linux/Windows) that creates a private network among your friends.
- Real Rust knowledge — enough to pick up any systems-programming project.
- A portfolio piece that impresses (seriously, this is senior-level work).

---

## Chapter 0 — Preparation

### 0.1 — Mindset
- **The compiler is your teacher, not your enemy.** Every error is a lesson. Read until you understand.
- **Refactoring the same module 5 times is normal.** In Rust, the first version is rarely the final one.
- **Don't copy code you don't understand.** The goal here is to learn, not to ship.

### 0.2 — Tools
Install in this order:

1. **rustup** — Rust version manager (`https://rustup.rs`).
2. **rust-analyzer** — LSP, install the extension in your editor.
3. **Editor:** VS Code, Helix, Neovim, Zed — anything with good rust-analyzer support.
4. **cargo-watch** — `cargo install cargo-watch` (auto-recompile on save).
5. **bacon** — `cargo install bacon` (prettier alternative to cargo-watch).
6. **Wireshark** — for packet inspection. Indispensable.

### 0.3 — Required pre-reading
Before writing a single line of code:
- Read chapters 1–10 of **The Rust Book** (`https://doc.rust-lang.org/book/`). It's dry. Do it anyway.
- Do the first 30 exercises of **Rustlings** (`https://github.com/rust-lang/rustlings`).

You don't need to memorize anything — just get exposure. You'll come back to this material throughout the project.

### 0.4 — Repository layout you'll build
We'll use a **Cargo workspace** — one repo, multiple crates:

```
p2p-vpn/
├── Cargo.toml          (workspace)
├── crates/
│   ├── core/           (tunnel logic, crypto)
│   ├── client/         (CLI/daemon that runs on the user's machine)
│   ├── coord-server/   (coordination server)
│   ├── protocol/       (messages exchanged between client and coord)
│   └── gui/            (Tauri, comes only in Phase 6)
└── EBOOK.md            (this file)
```

You'll create this **in Phase 1**, not now.

---

## Chapter 1 — Theoretical foundations

> Read once. Come back when you get lost.

### 1.1 — How IP packets work
The internet is a packet-pushing machine. Each packet has a header (source, destination, protocol) and a payload (the data). Routers read the header and forward to the next hop.

**What you need to know:** an IP packet is just bytes. You can read, modify, and craft packets from scratch. That's what a VPN does.

### 1.2 — UDP vs TCP
- **TCP:** reliable, ordered, slow. Great for HTTP.
- **UDP:** unreliable, fast, simple. Great for VPN, games, voice.

VPNs use **UDP** because the traffic running **inside** the tunnel already has TCP when it needs it. Putting TCP inside TCP causes "TCP meltdown" (performance disaster).

### 1.3 — TUN interface
Imagine a fake network card created by software. Read from it = you receive packets. Write to it = you inject packets into the OS. That's the heart of a VPN.

```
User's app ──> OS ──> tun0 (your interface) ──> your Rust program
                                                       │
                                                       ↓
                                                 encrypt,
                                                 send over UDP
```

### 1.4 — NAT — the P2P villain
Your home router doesn't give a public IP to every device. It shares **one** public IP across all of them. That's NAT.

Problem: if I'm A and I want to talk directly to B, but both are behind NAT, neither accepts inbound connections.

Solution: **UDP hole punching**. Both sides send packets "outward" simultaneously, opening holes in their NATs. If the holes line up, communication works. It doesn't work for all NAT types (especially "Symmetric NAT").

### 1.5 — Cryptography in 5 minutes
- **Symmetric (AES, ChaCha20):** same key encrypts and decrypts. Fast.
- **Asymmetric (RSA, X25519):** key pair (public/private). Slower.
- **Real-world pattern:** asymmetric to **exchange a symmetric key**, then symmetric for everything else.

That pattern is called a **handshake**. WireGuard uses the **Noise Protocol Framework** for the handshake.

### 1.6 — WireGuard, in one sentence
> A modern, minimalist VPN protocol based on UDP + Noise + ChaCha20-Poly1305.

Cloudflare rewrote it in Rust and open-sourced it: **`boringtun`**. We'll use it from Phase 3 onward.

---

## Chapter 2 — Essential Rust

You don't need to master everything upfront — just have a map of the territory.

### 2.1 — Ownership / Borrowing
The core rule: **a value has one owner. When the owner goes out of scope, the value is dropped.** You borrow (`&` or `&mut`) instead of copying.

This will hit you in the face during Phases 1–2. Push through.

### 2.2 — `Result<T, E>` and `Option<T>`
Rust has no exceptions. Functions that can fail return `Result`. The `?` operator propagates errors up.

### 2.3 — Traits
Like interfaces, but more powerful. You'll consume traits (`Read`, `Write`, `Future`) long before writing your own.

### 2.4 — Async/await with tokio
Rust async is unusual. Key points:
- `async fn` returns a `Future`. It does nothing until `await`ed.
- You need a **runtime** to run futures. We'll use `tokio`.
- `tokio::spawn` creates a "task" — concurrent, not necessarily parallel.

### 2.5 — Error handling
Use `thiserror` for errors in libraries (`crates/core`, `crates/protocol`).
Use `anyhow` in binaries (`crates/client`, `crates/coord-server`) where you just want "error with context".

### 2.6 — Cargo workspace
A root `Cargo.toml` declares `members = [...]`. Each subfolder is a crate. They can depend on each other via `path = "../core"`. This keeps the project modular.

---

## Phase 1 — "Hello, Packet!"

### Goal
Create a TUN interface, read raw IP packets, print source/destination to the terminal.

### Rust concepts you'll learn
- Cargo project structure
- Basic ownership
- `Result` and the `?` operator
- `loop` and `match`
- Reading bytes (`&[u8]`)

### Crates
- `tun` (Linux/macOS) or `wintun` (Windows) — TUN interface
- `etherparse` — IP header parser
- `anyhow` — error with context

### Tasks
1. Create the workspace and the `client` crate.
2. Add the dependencies.
3. Create a TUN interface named `vpn0` with address `10.8.0.1/24`.
4. In a loop: read a packet, parse it, print `src -> dst (proto, len)`.
5. Handle the "permission denied" error with a friendly message.

### Done criteria
In another terminal, run `ping 10.8.0.2` and see in your program:
```
10.8.0.1 -> 10.8.0.2 (ICMP, 84 bytes)
10.8.0.1 -> 10.8.0.2 (ICMP, 84 bytes)
```

### Common pitfalls
- **Permissions:** creating a TUN requires `sudo`/admin. Don't fight it yet — later you can solve it with setuid or `cap_net_admin`.
- **MTU:** if you create the TUN with MTU 1500 but your UDP link only handles 1400, large packets get dropped. Use MTU 1380 to be safe.
- **macOS is different:** the interface is called `utun0`, not `tun0`. The `tun` crate handles this.

### Estimate
3–6 hours, depending on how much Rust reading you've done.

---

## Phase 2 — Bare UDP tunnel (no encryption yet)

### Goal
Two processes on different machines (or two VMs) exchange packets via TUN, encapsulated in UDP.

### Rust concepts you'll learn
- `tokio` runtime basics
- async `UdpSocket`
- `tokio::select!` (waiting on multiple events)
- Channels (`tokio::sync::mpsc`) optional
- Borrowing in async code (will hurt a bit)

### New crates
- `tokio` with `["full"]` features

### Tasks
1. Refactor `client` to run two parallel "loops":
   - **Inbound:** read from the `UdpSocket`, write to the TUN.
   - **Outbound:** read from the TUN, write to the `UdpSocket`.
2. Configure via CLI args: my TUN's local IP, peer IP+port, my UDP port.
3. Test on two VMs (or Docker, or Tailscale between you and a friend just to get IP connectivity).

### Done criteria
- Machine A with `vpn0 = 10.8.0.1`, peer = `B:51820`.
- Machine B with `vpn0 = 10.8.0.2`, peer = `A:51820`.
- `ping 10.8.0.2` from A works. `ssh 10.8.0.2` works.

### Common pitfalls
- **Fanout:** still only 1 peer. Don't generalize to N peers yet — that's Phase 4.
- **Async + closure capturing `self`:** when you spawn tasks that touch `self`, you'll need `Arc<Self>`. Don't avoid it, learn it.
- **Small buffers:** allocate `[0u8; 1500]` per read. Don't use `Vec::new()`.

### Estimate
8–12 hours.

---

## Phase 3 — Add encryption (WireGuard via boringtun)

### Goal
Replace the bare tunnel with real WireGuard. Encrypted traffic.

### Rust concepts you'll learn
- Working with libraries that have "weird" APIs (boringtun uses a state-machine model)
- Type-state and enums with associated data
- Reading docs and third-party source code (underrated skill)

### New crates
- `boringtun` — WireGuard implementation
- `x25519-dalek` — key generation (already comes via boringtun, but good to know)
- `base64` — for serializing keys as text

### Tasks
1. Generate a key pair on startup. Save it in `~/.config/p2p-vpn/key`.
2. Configure a static peer for now: peer's pubkey + UDP endpoint, in a TOML config file.
3. Replace the "UDP encapsulation" from Phase 2 with `Tunn::new()` from boringtun.
4. The flow becomes:
   ```
   TUN ─> boringtun.encapsulate() ─> UDP socket
   UDP socket ─> boringtun.decapsulate() ─> TUN
   ```
5. Implement boringtun's timers (handshake, keepalive).

### Done criteria
- Same scenario as Phase 2, but:
- Capture UDP traffic between A and B in Wireshark.
- You see only scrambled bytes — no internal IPs, no ICMP.
- Ping still works.

### Common pitfalls
- **Boringtun is low-level.** It doesn't manage sockets, TUN, or scheduling — only crypto/protocol. You're the glue.
- **Timers:** WireGuard has 4 timers. Don't skip them. If you do, the connection silently drops in 2 minutes.
- **Endianness:** WireGuard uses little-endian in several places. Be careful when serializing.

### Estimate
15–25 hours. This is the phase that teaches you the most Rust.

---

## Phase 4 — Coordination server

### Goal
End the static configuration. Peers discover each other via a lightweight server.

### Rust concepts you'll learn
- HTTP server with `axum`
- Serialization/deserialization with `serde`
- Shared state (`Arc<Mutex<...>>` or `Arc<RwLock<...>>`)
- Simple persistence (sled or sqlite)
- Defining a "protocol" crate shared between client and server

### New crates
- `axum` — HTTP framework
- `serde` + `serde_json`
- `sled` (embedded KV) or `sqlx` + `sqlite`
- `tracing` — structured logging (replace `println!` everywhere)

### Architecture
```
crates/protocol/    ── shared types (RegisterRequest, PeerInfo, etc.)
crates/coord-server/── uses axum + protocol, exposes REST
crates/client/      ── uses reqwest to call the server
```

### Suggested endpoints
- `POST /rooms/:room/join` — body: `{ pubkey, endpoint }` → response: `{ peers: [...] }`
- `GET /rooms/:room/peers` — list active peers
- `POST /rooms/:room/heartbeat` — keep peer alive
- `DELETE /rooms/:room/peers/me` — leave

### Tasks
1. Create the `protocol` crate with `serde`-serializable structs.
2. Build `coord-server` with axum, in-memory state first.
3. Have `client` call `/join` on startup and receive the peer list.
4. Reconfigure boringtun dynamically when peers join/leave.
5. Add persistence (sled) so the server survives restarts.
6. Host on a cheap VPS (Hetzner, Oracle Free Tier, Fly.io free).

### Done criteria
- 3 friends download the client, all join the "home" room.
- Each one sees the other two in the list, direct connections work (still within the same network for now).

### Common pitfalls
- **Not authenticating rooms = anyone can join.** Use long random codes. Don't roll your own crypto.
- **`Mutex` in async code:** use `tokio::sync::Mutex`, not `std::sync::Mutex`, when the lock crosses an `await`.
- **Heartbeat:** if a peer disappears without `DELETE`, expire it after 60s.

### Estimate
20–30 hours.

---

## Phase 5 — NAT traversal (the hardest phase)

### Goal
Connect peers in real home networks, behind different NATs.

### Rust concepts you'll learn
- Heavy networking (STUN, ICE)
- Network debugging with `tcpdump`/Wireshark
- Handling timing and races
- Patient async (timeouts, retries)

### New crates
- `stun-rs` or `stun_codec` — STUN client
- (optional) `str0m` or pieces of `webrtc-rs` if you want full ICE

### New concepts
- **STUN:** a public server (e.g., `stun.l.google.com:19302`) that tells you your externally-visible IP/port.
- **Hole punching:** A and B send UDP packets to each other simultaneously. NATs open holes. If holes line up, it connects.
- **NAT types:** Full Cone (easy), Restricted, Port Restricted, Symmetric (almost impossible).

### Tasks
1. On client startup, run a STUN query. Discover your `(public_ip, port)`.
2. Send it to coord-server as part of `/join`.
3. When you learn a peer's public endpoint, start sending packets to them and listening for theirs simultaneously.
4. Detect when it works (WireGuard handshake completed) and mark the peer as "connected".
5. Implement fallback: if hole punching fails after 10s, mark the peer as "unreachable".

### Done criteria
- You're at home (behind your ISP's router).
- A friend is in another city (behind their router).
- No relay server in the path.
- You ping each other directly.

### Common pitfalls
- **Symmetric NAT:** will fail ~10–20% of the time on corporate/4G networks. **Accept it.** Note "TODO: implement TURN relay" and move on.
- **CGNAT:** your ISP may put you behind carrier-grade NAT. Without TURN, you're stuck.
- **OS firewalls:** Windows and macOS block UDP by default. Build a nice "allow connection" prompt.

### Estimate
30–50 hours. Budget a month.

---

## Phase 6 — Cross-platform GUI with Tauri

### Goal
Turn the daemon into an app non-technical people can use.

### Rust concepts you'll learn
- Tauri (Rust + web frontend)
- IPC between processes (frontend ↔ backend)
- Packaging and code signing
- Elevated permissions (creating a TUN needs root)

### New crates
- `tauri` — framework
- Frontend: your choice — vanilla JS, Svelte, React. I recommend **Svelte** or **Solid** (lightweight, simple).

### Decisions to make
- **Separate daemon or all-in-one?** Recommended: **privileged daemon**, unprivileged GUI. GUI talks to the daemon over a Unix socket / Named Pipe. More secure, more Unix-y.
- **Auto-start?** Yes. Use `launchd` (macOS), `systemd --user` (Linux), Windows Service.

### Tasks
1. Extract the current client into a `daemon` crate that exposes a local socket.
2. Create the `gui` crate with Tauri.
3. GUI: login screen (room code), peer list, connect/disconnect button, status.
4. Package: `.app` (macOS), `.deb`/`.AppImage` (Linux), `.msi` (Windows).
5. Code signing — expensive but necessary for Mac and Windows to not warn users.

### Done criteria
- You send the `.dmg` to a non-technical friend.
- They click, open, type the code, connect.
- No terminal needed.

### Common pitfalls
- **macOS Gatekeeper:** without an Apple Developer account ($99/year), the app shows as "untrusted". You can work around this with instructions, but it loses users.
- **Windows needs the Wintun driver:** ship it alongside.
- **Auto-update:** Tauri supports it. Set it up early.

### Estimate
40–60 hours.

---

## Phase 7 — Robustness and production

### Goal
Go from "works on my machine" to "works for anyone, always".

### Tasks
- **Tracing:** structured logs everywhere with `tracing` + `tracing-subscriber`.
- **Metrics:** optional, but the `metrics` crate is simple.
- **Reconnection:** Wi-Fi drop, switch to 4G — should recover automatically.
- **Unit tests:** mostly in `core` (packet parsing, timer logic).
- **Integration tests:** spin up 2 daemons in containers, simulate scenarios.
- **CI:** GitHub Actions — `cargo build`, `cargo test`, `cargo clippy -- -D warnings`, `cargo fmt --check`.
- **Release automation:** `cargo dist` or similar.
- **Documentation:** serious `cargo doc`, presentable README, demo video.

### Done criteria
- 50 stars on GitHub. (Joking. But it can happen.)

---

## Appendix A — Quick glossary

| Term | Meaning |
|---|---|
| TUN/TAP | Virtual network interface in the OS. TUN = layer 3 (IP). TAP = layer 2 (Ethernet). |
| NAT | Network Address Translation — hides internal IPs behind a public one. |
| STUN | Protocol to discover your public IP as seen from outside. |
| TURN | Relay server when hole punching fails. |
| ICE | Algorithm combining STUN + TURN + connection attempts. |
| Noise Protocol | Cryptographic handshake framework. WireGuard uses it. |
| AEAD | Authenticated Encryption with Associated Data — encryption + integrity. |
| MTU | Maximum Transmission Unit — max packet size on the link. |

---

## Appendix B — Parallel resources

### Rust
- **The Rust Book** (official, free): mandatory reading, first 10 chapters.
- **Rustlings** (official exercises): do in parallel with Phases 1–2.
- **Rust by Example**: quick reference.
- **Tokio Tutorial** (`tokio.rs/tokio/tutorial`): do before Phase 2.
- **Jon Gjengset on YouTube** ("Crust of Rust"): long, deep videos. Pure gold after Phase 3.

### Networking & VPN
- **WireGuard whitepaper** (`https://www.wireguard.com/papers/wireguard.pdf`): read before Phase 3.
- **Tailscale blog** — best explanations of NAT traversal on the internet.
- **Beej's Guide to Network Programming** — classic, in C, but the concepts hold.

### Inspiration
- **boringtun** (source code): read it. It's didactic.
- **innernet** (Rust, mesh VPN): real-world code, similar to what you're building.
- **Tailscale** (Go, partially open): the UX reference.

---

## Appendix C — Suggested 12-week schedule (~10h/week)

| Week | Focus |
|---|---|
| 1 | Chapters 0–2. Rust Book chs 1–6. Rustlings 1–20. |
| 2 | Phase 1. Continue Rustlings. |
| 3 | Phase 2 (part 1: tokio basics). |
| 4 | Phase 2 (part 2: TUN+UDP integration). |
| 5–6 | Phase 3. Read the WireGuard paper. |
| 7 | Phase 4 (part 1: axum + protocol). |
| 8 | Phase 4 (part 2: persistence + deploy). |
| 9–10 | Phase 5. It will hurt. Push through. |
| 11 | Phase 6. |
| 12 | Phase 7 + polish. |

Realistic: **double this** if it's your first Rust project. No shame.

---

## Appendix D — Decisions to make before starting

Write down your answers. Come back if you change your mind.

1. **Initial target platforms?** (Suggested: Linux + macOS for Phases 1–5, add Windows in Phase 6.)
2. **Project name?**
3. **License?** (MIT or Apache 2.0 are standard. I suggest MIT.)
4. **Public repo from day 1, or only after Phase 3?**
5. **VPS for the coord-server?** (Suggested: Oracle Free Tier — ARM, 4GB, free forever.)
6. **Community?** (Discord/Matrix? Can wait until Phase 5.)

---

## Appendix E — How to ask me (Claude) for help

When you're stuck, open a chat and tell me:
1. **Which phase you're in** (e.g., "Phase 3, task 3").
2. **What you expected to happen.**
3. **What actually happened** (paste the full error).
4. **What you've already tried.**

Don't ask me to "do it for you" — ask me to explain the missing concept. The point is for *you* to code.

Good example:
> "I'm in Phase 3, task 4. boringtun returns `WriteToNetwork(buf)` but I don't understand whether I should send `buf` over the UDP socket immediately or wait for something else. I've read the docs for `Tunn::encapsulate` but it's still unclear."

Bad example:
> "my vpn doesn't work, help me"

---

## Closing

This project will be hard. You'll want to quit about 4 times. Don't quit. Phase 3 is the "valley of despair" — after it, it's downhill.

When you finish, you'll have:
- Real command of Rust async, ownership, traits.
- Networking knowledge that 95% of developers don't have.
- A personal project that isn't "yet another TODO list".
- A VPN that **works** and that you **understand byte by byte**.

Good luck. It will be work. It will be worth it.
