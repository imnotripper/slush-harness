# Slush Wallet — P3 malicious-dApp test harness

A single self-contained page (`index.html`, no build step) that plays the role of an
**untrusted dApp**: it registers the Slush **web** wallet through the official
`@mysten/slush-wallet` SDK, then drives Wallet Standard connect / sign flows with
fully attacker-controlled parameters. Built for checklist item **P3** of
`../plan/test-checklist-slush.md`; it is the driver for the C1–C6 and T1/T3 tests.

Everything runs client-side. The page makes exactly one kind of network request of
its own: loading its JS modules from the esm.sh CDN. All wallet interaction happens
in the tester's browser via the Wallet Standard; transaction execution (only via the
explicit `signAndExecuteTransaction` button) goes through the wallet itself.

## Serving it

The page works from **any origin** — it has no backend, no relative assets, and reads
its own origin at runtime (and displays it, for the C6 comparison). Options, simplest
first:

```bash
# from this directory — local testing
python3 -m http.server 8734
# or
npx serve -l 8734 .
# then open http://localhost:8734/
```

**Important:** the Slush web wallet (my.slush.app) is opened by the SDK in a popup /
new tab from the dApp page, and the delegated session-key flow binds the wallet tab
and the dApp tab together. The harness page must therefore be open in the **same
browser** (same profile) that you use for my.slush.app. A `localhost` URL is fine for
the SDK, but if the wallet session was set up in a different browser/profile, use a
public URL instead, e.g.:

- GitHub Pages from a repo containing just `index.html` (do **not** publish from the
  bug-bounty workspace repo itself if it is private — copy the single file), or
- any static host / tunnel (`npx serve` + a tunnel) — the page is origin-agnostic, so
  no code changes are needed when the URL changes.

Serving is a parent-agent / Bro step: this harness was built and syntax-checked only;
nothing has been deployed.

## What each control does

**Section 0 — Origin & setup**
- Shows `window.location.origin` / `href` verbatim — screenshot-reference for C6
  (does the wallet prompt attribute the request to this true origin?).
- *dApp name* field: the exact string passed to `registerSlushWallet(name)`. Slush
  displays this name in its approval UI, so it is itself a test variable (C6: name a
  trusted service here and see whether the prompt makes the impersonation obvious).
- *Network selector*: testnet (default), mainnet, devnet. Sent as the Wallet Standard
  `chain` (`sui:testnet`, …) on every sign request and shown in the log.
- *Register Slush wallet*: calls `registerSlushWallet(name)` from
  `@mysten/slush-wallet@2.0.1` (also run automatically on page load). The returned
  wallet is registered with the Wallet Standard registry.
- *Wallet picker*: every wallet found via Wallet Standard discovery (`getWallets()`
  from `@wallet-standard/app`), including the Slush **extension** if it is installed —
  useful as a control, but note the extension is **not confirmed in scope**; use the
  SDK-registered web wallet named **“Slush”** for findings. The panel under the picker
  lists the active wallet's version, chains, features, and accounts.

**Section 1 — Connect**
- *Connect*: `standard:connect` → opens the Slush web wallet approval flow
  (my.slush.app popup; delegated session keys). Accounts returned are listed in full
  (no truncation); click an account to select it as the signer.
- *Disconnect*: `standard:disconnect`. Whether the session actually dies server-side
  is A1/D3 territory — the log shows the response.
- `standard:events` `change` notifications are logged as EVENT entries.

**Section 2 — signPersonalMessage (C4)**
- Fully editable message textarea; the page UTF-8-encodes exactly what is written and
  logs the text, the byte length, and the bytes as base64 + hex **before** sending, so
  what-was-signed is provable afterwards.
- Presets: a plain login message; a homoglyph/zero-width probe (Cyrillic look-alikes +
  U+200B) for display-fidelity checks; a base64-blob placeholder — replace it with real
  serialised transaction bytes (or another service's auth text) for the C4 test: does
  the wallet render the payload readably, or sign it blind?

**Section 3 — Transactions (C1 / C2 / C3 / T1 / T3)**
Three builder tabs, all driven by editable JSON:
- *Transfer SUI*: `{ recipient, amountMist, coinObjectId?, sender?, gasBudget?, gasOwner? }`
  → `splitCoins` + `transferObjects` via `@mysten/sui@3.0.1` `Transaction`.
  Presets: 1 MIST dust; exactly 1 SUI (T1 unit-display check); foreign `gasOwner`
  (C3 sponsor-field probe).
- *Move call*: `{ target, typeArguments[], arguments[], sender?, gasBudget?, gasOwner? }`.
  Argument descriptors: `{kind:"pure", type, value}` (u8–u256, bool, string, address,
  id, `vector<T>`, `option<T>`), `{kind:"object", id}`, `{kind:"sharedObject", id,
  initialSharedVersion, mutable}`, `{kind:"gasCoin"}`. Presets: harmless
  `0x2::tx_context::sender`; `0x2::pay::split_and_transfer` (the C1/C2 crafting base —
  recipient and amount ride in the arguments, independent of anything a dApp page
  *claims*); `0x2::coin::destroy_zero` (object-arg example).
- *Raw tx JSON*: a full serialised transaction (as produced by `toJSON()`), fed
  untouched to `Transaction.from(...)` — the C3 route for payloads no normal builder
  UI would emit. “Fill from current tab's toJSON()” seeds it from the other tabs.
- *Set sender* checkbox: standard dApp behaviour (sender = connected account); a
  spec-level `sender`/`gasBudget`/`gasOwner` overrides it — the sponsor/gas fields
  are part of the C3 surface.
- *Build & preview toJSON()*: logs the exact serialised transaction. Under Wallet
  Standard `sui:signTransaction` v2.0.0 the wallet receives an object whose
  `toJSON()` returns precisely this string — so the preview **is** the request
  payload; diff it against the wallet's confirmation UI (that diff is C1/C2/C3).
- *signTransaction*: sign only, returns `{ bytes, signature }` (base64).
- *signAndExecuteTransaction* (red — it really submits): returns
  `{ bytes, signature, digest, effects }`. Testnet only, wallet A only.

**Section 4 — Log**
- Every registration, request, response, event, and error is appended with an ISO
  timestamp and shown verbatim (pretty-printed JSON; `Uint8Array`s shown as
  length + base64 + hex; errors with name/message/stack). Copy/Clear buttons;
  auto-scroll toggle; capped at 400 entries.

## Checklist items this harness serves

| Item | How |
|---|---|
| P3 | The harness itself: connect + sign-message + sign-and-execute with crafted parameters from a page we control. |
| C1 | Move-call / transfer tabs: build a tx whose arguments (recipient, amount) differ from anything the page text claims; compare wallet UI vs the logged `toJSON()`. |
| C2 | Move-call tab: put package X in the page's claim, package Y in `target`; check what the wallet displays vs executes. |
| C3 | Raw tx JSON tab + `gasOwner`/`gasBudget` spec fields: payloads and sponsor/gas arrangements a normal UI wouldn't render; check for blind signing. Also signPersonalMessage with opaque payloads. |
| C4 | Message tab with transaction-byte / cross-service auth payloads. |
| C5 | Connect section + log: session behaviour across origins/tabs is observable here (pairing/session binding itself is inspected in the wallet tab). |
| C6 | Section 0: editable dApp name + displayed true origin vs the wallet prompt's attribution; homoglyph message preset. |
| T1 | Transfer presets (1 MIST vs 1 SUI) + arbitrary `amountMist` strings: signed amount vs displayed amount/unit. |
| T3 | Crafted move calls / metadata-shaped strings in arguments to stress the confirmation UI layout. |

Not served by this page: T2/T4 (need an NFT / crafted addresses in wallet A — P2),
A4/D3/N1 (use this page as the visiting page, observe the wallet side), W2 (redirect
chains into the connect flow — drive from the wallet tab).

## SDK notes (verified against @mysten/slush-wallet@2.0.1 published types)

- Entry point: `registerSlushWallet(name, { origin?, metadataApiUrl? })` →
  `{ wallet, unregister } | undefined`. It can return `undefined`; the page handles it.
- The package docs (`docs/dapp.md`) mention a `network` option and a default origin
  of `https://[TODO]` — both are stale: the 2.0.1 type signature has **no** `network`
  option. Network selection happens per-request via the Wallet Standard `chain`
  field, which is how this page's selector works.
- Wallet identity: name `"Slush"` (`SLUSH_WALLET_NAME`), version `"1.0.0"`, chains
  `sui:devnet / sui:testnet / sui:localnet / sui:mainnet`.
- Features exposed: `standard:connect`, `standard:disconnect`, `standard:events`,
  `sui:signTransaction` (v2.0.0), `sui:signAndExecuteTransaction` (v2.0.0),
  `sui:signPersonalMessage` (v1.1.0), plus legacy `sui:signTransactionBlock`.
- `sui:signTransaction` input's `transaction` is duck-typed as `{ toJSON(): Promise<string> }`
  — the wallet serialises the `Transaction` object itself; no pre-built bytes are sent.
- Peer dependency `@mysten/sui@^3.0.1`; the import map pins esm.sh to serve
  slush-wallet with that exact `@mysten/sui@3.0.1` so the page and the SDK agree.
- If the Slush extension is installed, dApp-kit hides the web wallet behind it; with
  raw Wallet Standard (this page) both can appear in discovery — pick “Slush”
  (the SDK-registered one) for in-scope web-wallet tests.

## Safety

Testnet only; wallets A/B only (P2); never paste a mainnet address into a preset
without re-checking the network selector. `signAndExecuteTransaction` spends real
(testnet) funds from the connected account. Findings workflow stays draft-first for
Bro's approval, per the track rules.
