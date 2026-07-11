# Running tor + silk locally

This document walks through reproducing the tor/silk relay demo on your
own machine, from unpacking the repos to a live three-terminal chat
session.

## 1. Extract both repos as siblings

`silk`'s `go.mod` references `../tor` via a local `replace` directive,
so the two repos need to sit next to each other on disk:

```sh
mkdir -p ~/dev && cd ~/dev
tar xzf tor-repo.tar.gz
tar xzf silk-repo.tar.gz
# now: ~/dev/tor and ~/dev/silk
```

## 2. Make sure you have Go 1.22+

```sh
go version
```

If it's not installed: [go.dev/dl](https://go.dev/dl), or
`brew install go` / `apt install golang-go`.

## 3. Sanity-check both repos build and pass tests

```sh
cd ~/dev/tor  && make ci
cd ~/dev/silk && make ci
```

`make ci` runs fmt-check → vet → build → race-tested unit tests, fully
offline — no network calls, no secrets.

## 4. Run the live demo — 3 terminals

**Terminal 1 — the relay server:**

```sh
cd ~/dev/silk
go run ./cmd/silk -mode=server -addr=127.0.0.1:9797
```

**Terminal 2 — client "alice":**

```sh
cd ~/dev/silk
go run ./cmd/silk -mode=client -addr=127.0.0.1:9797 -name=alice
```

**Terminal 3 — client "bob":**

```sh
cd ~/dev/silk
go run ./cmd/silk -mode=client -addr=127.0.0.1:9797 -name=bob
```

Type a message in alice's terminal and hit enter — it appears in bob's
terminal, and vice versa, relayed through the server (never echoed
back to the sender). `Ctrl+C` any terminal to tear it down.

## 5. (Optional) Run the actual CI workflow locally, not just `make ci`

If you want the real GitHub Actions YAML executing on your machine via
Docker instead of just the equivalent Makefile target:

```sh
brew install act        # or see nektos/act install docs
cd ~/dev/tor && make act
```

For `silk`, `make act` will try to `git clone`
`github.com/p9xr/tor` (per the workflow's sibling-checkout step) — that
only works once `tor` has actually been pushed to GitHub as a public
repo. Until then, `make ci` is the right local check for `silk`.

## 6. (Optional) SSH + GPG for pushing to GitHub

```sh
cd ~/dev/tor  && ./scripts/setup-dev-keys.sh
cd ~/dev/silk && ./scripts/setup-dev-keys.sh
```

Each prints two public keys to paste into
`https://github.com/settings/keys` (the SSH tab and the GPG tab). No
tokens, no API calls — just your local `ssh-keygen`/`gpg` binaries.

## 7. Push to GitHub

```sh
cd ~/dev/tor
git remote add origin git@github.com:p9xr/tor.git   # or your own org/name
git push -u origin main

cd ~/dev/silk
git remote add origin git@github.com:p9xr/silk.git
git push -u origin main
```

Push `tor` first — `silk`'s cloud CI checks it out as a sibling
directory and will fail until `tor` exists on GitHub.

## What you should see

```
=== server log ===
silk: relay listening on 127.0.0.1:9797
silk: client connected: 127.0.0.1:xxxxx   (alice)
silk: client connected: 127.0.0.1:xxxxx   (bob)

=== alice's terminal ===
> bob: yep, loud and clear

=== bob's terminal ===
> alice: hey bob, you there?
> alice: cool, testing the relay
```

This confirms the full stack: `tor`'s dialer connects, its
length-prefixed framing carries messages intact, `silk`'s relay fans
out to *other* clients while excluding the sender, and everything
tears down cleanly on disconnect.
