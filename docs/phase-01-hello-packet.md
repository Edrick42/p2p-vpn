# Phase 1 — "Hello, Packet!" — Step-by-step workbook

> One atomic step at a time. Don't move forward until the current step's "Verify" passes.
> Reference for the high-level goals: `EBOOK.md` → Phase 1.

---

## Step 1.1 — Create the Cargo workspace skeleton

**Action**
- Make sure you're in `/Users/edrcikmanoel/Projects/p2p-vpn/`.
- Create a root `Cargo.toml` (you write it manually, not via `cargo new`).
- It must contain only the `[workspace]` table — no `[package]`.
- Required keys: `resolver = "2"` and `members = []` (empty for now).
- Optional but recommended: `[workspace.package]` with `version`, `edition = "2021"`, `authors`, `license = "MIT"`. Child crates will inherit these.

**Where to look**
- The Cargo Book — Workspaces: `https://doc.rust-lang.org/cargo/reference/workspaces.html`

**Verify**
```
cargo build
```
Should succeed silently (workspace has no members yet, so nothing is compiled).

**Done criterion**
- `Cargo.toml` exists at the repo root and has no `[package]` section.

---

## Step 1.2 — Create the `client` crate

**Action**
- From the repo root, run:
  ```
  cargo new --bin crates/client
  ```
- Add `"crates/client"` to the `members` array in the root `Cargo.toml`.

**Verify**
```
cargo run -p client
```
Prints `Hello, world!`.

**Done criterion**
- Folder `crates/client/` exists with `Cargo.toml` and `src/main.rs`.
- `cargo run -p client` works.

**Pitfall**
- If `cargo new` complained "destination is not empty", you probably already had a `crates/client/` folder. Delete it and re-run, or do `cargo init crates/client --bin`.

---

## Step 1.3 — Add a `.gitignore`

**Action**
- Create a `.gitignore` at the repo root with at least:
  - `/target`
  - `.DS_Store`
- For workspaces, you generally want to commit `Cargo.lock` (since the root crate is a binary workspace). So **don't** add it to `.gitignore`.

**Verify**
```
git status
```
- `target/` should not appear.

**Done criterion**
- `.gitignore` exists, `target/` is not tracked.

---

## Step 1.4 — First commit

**Action**
```
git add -A
git commit -m "chore: initialize cargo workspace with client crate"
```

**Verify**
```
git log --oneline
```
Shows your new commit.

**Done criterion**
- Working tree is clean.

---

## Step 1.5 — Pin the Rust toolchain (recommended)

**Action**
- Create a `rust-toolchain.toml` at repo root, declaring `channel = "stable"`. This guarantees everyone uses the same compiler version.
- Read about it: `https://rust-lang.github.io/rustup/overrides.html#the-toolchain-file`

**Verify**
```
rustc --version
```
Should match the channel you declared.

**Done criterion**
- `rust-toolchain.toml` exists, committed.

---

## Step 1.6 — Read the `tun` crate documentation (no code yet)

**Action**
- Open `https://docs.rs/tun/latest/tun/`.
- Locate and read about: `Configuration`, `create()`, the `Device` trait.
- Look at the example in the crate's README: `https://github.com/meh/rust-tun`.

**Done criterion**
- You can answer (out loud, to yourself):
  - How do I set the interface address?
  - What does `up()` do on the configuration?
  - What type does `create()` return?
  - What trait must I import to be able to call `.read()` on the device?

**Why this step matters**
- A huge part of Rust is reading docs of crates. Get used to it now, while the API is small.

---

## Step 1.7 — Add `tun` and `anyhow` as dependencies

**Action**
- From the repo root, run:
  ```
  cargo add tun --package client
  cargo add anyhow --package client
  ```
- Open `crates/client/Cargo.toml` and confirm both are listed under `[dependencies]`.

**Verify**
```
cargo build -p client
```
Compiles successfully (will download the crates).

**Done criterion**
- `cargo build` succeeds, `Cargo.lock` updated.

---

## Step 1.8 — Create the TUN interface (no reading yet)

**Action**
- In `crates/client/src/main.rs`, replace `Hello, world!` with code that:
  1. Builds a `tun::Configuration` with address `10.8.0.1`, netmask `255.255.255.0`, brings it up.
  2. Calls `tun::create()` — handle the error with `expect("...")` for now.
  3. Prints `"TUN interface created. Press Ctrl+C to stop."`.
  4. Loops forever with `std::thread::sleep` (e.g. 60 seconds) — we just want the interface to stay alive.

**Run**
- macOS: `sudo cargo run -p client`
  - You may need `sudo -E cargo run -p client` to preserve env vars.

**Verify**
- In another terminal:
  - macOS: `ifconfig | grep -B1 -A3 utun`
  - Linux: `ip addr | grep -A3 vpn0`
- You should see an interface with address `10.8.0.1`.

**Done criterion**
- The interface is visible in `ifconfig` while the program runs, and disappears when you Ctrl+C.

**Pitfall**
- macOS doesn't let you choose the interface name. The crate will use `utun0`, `utun1`, etc. (the next available).
- If `sudo cargo` complains about cargo not being found, try `sudo $(which cargo) run -p client`.

---

## Step 1.9 — Read raw bytes from the TUN

**Action**
- Bring `std::io::Read` into scope (you need this trait for `.read()` on the device).
- In the loop, replace the `sleep` with:
  - Allocate a buffer `[0u8; 1500]`.
  - Call `dev.read(&mut buf)` — returns `Result<usize>`. Use `expect()` for now.
  - Print `"got {n} bytes"`.

**Run**
- `sudo cargo run -p client`.
- In another terminal: `ping 10.8.0.2`.

**Verify**
- Your program prints lines like `got 84 bytes`, one per ping packet.
- The ping itself **fails** (no one is replying yet). That's expected.

**Done criterion**
- Bytes flow into your program every second when you ping.

**Pitfall**
- If you see `0 bytes` or block forever: check that the interface is actually up (`ifconfig`).
- If you see size 4 always: on macOS, `utun` prepends a 4-byte protocol header to every packet. We'll handle that next step.

---

## Step 1.10 — Parse the IP header

**Action**
- `cargo add etherparse --package client`.
- After `dev.read(&mut buf)` returning `n`, slice the bytes you actually want to parse:
  - Linux: `&buf[..n]`
  - macOS: `&buf[4..n]` (skip the 4-byte protocol family prefix). Look at the first 4 bytes for diagnostic — they should be `00 00 00 02` for IPv4.
- Use `etherparse::Ipv4HeaderSlice::from_slice(...)` to parse the IPv4 header.
- On `Ok`, print: `src -> dst (proto, len)`. Get those fields from the header slice.
- On `Err`, print `"non-IPv4 packet, skipping"` and continue.

**Verify**
- Run, then `ping 10.8.0.2` from another terminal. You should see:
  ```
  10.8.0.1 -> 10.8.0.2 (ICMP, 84 bytes)
  10.8.0.1 -> 10.8.0.2 (ICMP, 84 bytes)
  ```

**Done criterion**
- ✅ This matches the Phase 1 "Done" criterion in the EBOOK.

**Pitfall**
- `etherparse` returns proto as a number, not a string. Match on it (1 = ICMP, 6 = TCP, 17 = UDP) to print a friendly name. Use a `match` expression — perfect place to practice that.

---

## Step 1.11 — Replace `expect` with proper error handling

**Action**
- Change `main` signature from `fn main()` to `fn main() -> anyhow::Result<()>`.
- Replace every `expect(...)` with `?`.
- Add a `use anyhow::Context;` import.
- On the `tun::create()` call, chain `.context("failed to create TUN interface — did you run with sudo?")`.
- Return `Ok(())` at the end (even though the loop is infinite, the compiler still wants this).

**Verify**
- Run **without** `sudo`: `cargo run -p client`.
- You should see a clean error message that mentions sudo, not a panic with stack trace.

**Done criterion**
- No `expect()` or `unwrap()` left in `main.rs`.
- Error messages are human-readable.

**Why this step matters**
- This is the moment the `Result` / `?` mental model clicks. The earlier confusion about `?` "returning null" — by the end of this step, you'll have used `?` enough to feel what it does.

---

## Step 1.12 — Commit and reflect

**Action**
```
git add -A
git commit -m "feat(client): read and parse IP packets from TUN interface"
```

**Reflection prompts** (no need to write down — just think for 60 seconds each):
- Where did the compiler push back the most? What did you misread?
- What was the most surprising thing about Rust ownership in this phase?
- If you had to explain `?` to a friend now, would you say it differently than before?

**Done criterion**
- Phase 1 commits are pushed (or at least committed locally).
- You're ready for Phase 2.

---

## Phase 1 — Self-assessment checklist

Before declaring Phase 1 complete, confirm all of these:

- [ ] Cargo workspace builds with `cargo build` (no warnings besides `dead_code` on stuff you'll use later).
- [ ] `cargo run -p client` (with sudo) creates a visible TUN interface.
- [ ] You can `ping 10.8.0.2` and see `src -> dst (ICMP, N bytes)` lines flow.
- [ ] Running without sudo prints a friendly error, not a stack trace.
- [ ] You used `?` in at least 3 places.
- [ ] Code is committed.

Once all six are checked, ping me with: "Phase 1 done, ready for Phase 2." I'll create `phase-02-bare-tunnel.md` and we'll move on.
