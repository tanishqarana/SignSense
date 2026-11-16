# BUILD_PROMPT.md — "Read it before you sign" (working name: SignSense)

> **For Ani, not the agent — read this box first.**
>
> 1. Create an empty GitHub repo (public), clone it, put this file at the repo root as `BUILD_PROMPT.md`.
> 2. Do the **manual prerequisites in Section 2** yourself (accounts, keys, Android Studio). The agent can't click through sign-ups for you.
> 3. Open Claude Code in the repo with **Fable** selected and paste the **Session 1 kickoff message** from Section 16.
> 4. Later sessions (Opus): paste the **Continue message** from Section 16. The agent picks up from `docs/PROGRESS.md`.
> 5. Rename "SignSense" whenever you like. It's a placeholder. Find-and-replace is enough.
>
> Everything below the line is written **to the coding agent**.

---

## 0. Your role and mission

You are the lead engineer on a hackathon project for the **Dev3pack Global Hackathon (27–29 Nov 2026)**, a Solana hackathon. Its two named prizes are **Best App Overall on Solana** and **Best Mobile App on Solana Mobile**. Sponsors: Solana, Solana Mobile, Supabase, ElevenLabs, Firecrawl.

You are building **SignSense**, an Android app (Expo / React Native) that lets someone check a Solana link, QR code or transaction **before** they approve it. It:

1. resolves what the link would ask the user to sign,
2. **decodes and simulates** that transaction against the real chain,
3. runs a **deterministic security rule engine** over the decoded instructions and the simulated balance changes,
4. checks the website/domain behind the link,
5. explains the result in plain words (English + Hindi), on screen and **by voice**,
6. and only then (if it is not dangerous) hands the exact same transaction to the user's wallet to sign.

You are working with **Ani** (product owner). She is new to Solana and mobile development but has a strong **offensive-security / VAPT** background. She owns product decisions and the pitch. You own implementation quality. Explain Solana concepts plainly in `docs/LEARN.md` as you go (see Section 14).

---

## 1. Product spec

### 1.1 Problem
Users approve Solana transactions they cannot read. Real attack patterns this app must catch:
- **Token account owner reassignment** (`SetAuthority` with `AccountOwner`): gives an attacker ownership of a token account. It is worse than a spending limit.
- **Delegate approvals** (`Approve` / `ApproveChecked`): lets another address move tokens later.
- **System `Assign`**: changes the owner program of the user's wallet account.
- **Durable-nonce transactions** (`AdvanceNonceAccount` first): the signature never expires. It can be held and submitted later. This was the core of the April 2026 Drift Protocol attack, where signers pre-signed durable-nonce transactions that looked routine.
- **Malicious Blinks / Solana Actions**: a link whose API returns a draining transaction.
- **Look-alike / brand-new phishing domains.**

### 1.2 Users
Everyday Seeker and Android Solana users. Secondary: newcomers (students, Web2 users) who want a second opinion before signing.

### 1.3 Inputs (priority order)
| # | Input | Example | How we get a transaction |
|---|---|---|---|
| 1 | Solana Action / Blink, explicit | `solana-action:https://example.com/api/actions/claim` | Actions GET (metadata) → user picks action → POST `{ account }` → base64 tx |
| 2 | Website linked to Actions | `https://example.com/claim` | Fetch `https://example.com/actions.json`, map path → action API, then as #1 |
| 3 | Blink interstitial URL | `https://<interstitial>/?action=solana-action:...` | Extract the `action` param, then as #1 |
| 4 | Solana Pay transaction request | `solana:https://merchant.example/api/pay` | GET (label/icon) → POST `{ account }` → base64 tx |
| 5 | Solana Pay transfer request | `solana:<recipient>?amount=1&spl-token=<mint>&label=...` | Build the transfer tx ourselves |
| 6 | Plain website URL (no actions) | `https://jup-claim.example` | **Site check only** (no transaction) |
| 7 | Raw transaction (dev/test) | base64 or base58 string | Deserialize directly |

Entry points in the app: **Android share sheet** (text/URL shared from any app), **QR scan**, **paste**.

### 1.4 Output: the Report
- **Verdict banner.** Never use the word "safe". Defaults (Ani may change):
  - `danger`: "Don't sign this."
  - `caution`: "Check carefully before signing."
  - `no_known_risk`: "No known risks found."
  - `unknown`: "We couldn't preview this. Treat it as risky."
- **"If you sign this…"**: 1–5 plain-language bullets. Example: "You give 7xKp…9fQ full ownership of your USDC account. They could move all of it at any time."
- **Balance changes**: table of the user's SOL/token deltas from simulation.
- **Findings**: each has severity, title, explanation, and **evidence** (instruction index, program, accounts).
- **Site check panel**: if there's a URL.
- **Listen** button: English / Hindi voice summary (ElevenLabs).
- Actions:
  - "Continue to wallet": only if verdict ≠ `danger`. Hands the **exact analyzed bytes** to the wallet via Mobile Wallet Adapter.
  - "Report as scam".
  - "Share warning".

### 1.5 Design principles (non-negotiable)
1. **Rules decide; language models never decide.** The verdict comes only from deterministic rules. If an LLM is added later, it may only rephrase text, and scraped web content must never reach a prompt that influences the verdict.
2. **Every finding cites evidence.** No finding without the instruction or account it came from.
3. **Never hold keys.** The app is not a wallet. Signing always happens in the user's wallet app through MWA.
4. **Never sign something different from what was analyzed.** If a transaction is re-fetched (for example, its blockhash expired), re-analyze it and compare instruction fingerprints before allowing signing.
5. **Fail closed.** If simulation fails or a resolver errors, the verdict is `unknown`, never `no_known_risk`.
6. **Honest limits.** The app can't intercept other apps' transactions. It is a "check before you go" step, and the UI says so once in onboarding.

### 1.6 Non-goals (do not build)
- A wallet or key storage. Mainnet transactions from test fixtures. Browser extension. iOS.
- A custom Anchor program in Sessions 1–2. It's a stretch goal and Ani's learning task.
- Any page or asset that imitates a real brand (no fake Jupiter/Phantom pages, even for demos).

---

## 2. Manual prerequisites (Ani does these; agent verifies at start)

The agent must check each item at the start of Session 1. Report what's missing as a short checklist, and **continue with the parts that don't need the missing item.**

**Accounts and keys** (put them in `.env` files; never commit them):
- [ ] **Supabase** project (free tier). Note the project URL, anon key and service-role key. Install the Supabase CLI.
- [ ] **Helius** API key (free tier). RPC endpoints `https://mainnet.helius-rpc.com/?api-key=…` and `https://devnet.helius-rpc.com/?api-key=…`.
- [ ] **Firecrawl** API key.
- [ ] **ElevenLabs** API key. The hackathon gives participants a Creator plan; the free tier works until then.
- [ ] GitHub repo (public; the hackathon requires open source).

**Machine (macOS):**
- [ ] Node.js LTS, Git, JDK 17.
- [ ] **Android Studio** with an emulator (recent API level), **plus a screen-lock PIN set on the emulator**. The mock wallet needs it.
- [ ] **Mock MWA Wallet** installed on the emulator: `git clone https://github.com/solana-mobile/mock-mwa-wallet.git`, open it in Android Studio, run it on the emulator, and tap Authenticate.
- [ ] Optional: a physical Android phone with USB debugging. No Seeker is needed; any Android device behaves the same for development.

**Devnet funds:**
- [ ] Two devnet test wallets funded from `https://faucet.solana.com` (the "victim" wallet and the "attacker" wallet for fixtures). The agent generates the keypairs; Ani funds them if CLI airdrops are rate-limited.

---

## 3. Architecture

```
┌────────────────────────── Android app (Expo dev build) ──────────────────────────┐
│ Share-sheet / QR / Paste ─► Analyze screen ─► Report screen ─► (Continue) MWA ──►│── Wallet app
│   - wallet connect (MWA) to learn the user's address                             │   (Seed Vault /
│   - plays voice summary                                                          │    Mock MWA Wallet)
└──────────────┬───────────────────────────────────────────────────────────────────┘
               │ HTTPS (Supabase anon/user JWT)
┌──────────────▼──────────── Supabase Edge Functions (Deno) ───────────────────────┐
│ analyze  : classify → resolve → decode → simulate → rules → site check → verdict │
│ speak    : text → ElevenLabs TTS → mp3 (cached in Storage)                       │
│ report   : store scam report (auth required)                                     │
│ mock-actions : DEVNET-ONLY test fixture Action server (clearly labeled)          │
│ _shared/ : the analysis engine (pure TS, unit-tested with `deno test`)           │
└──────┬──────────────┬────────────────────┬──────────────────┬────────────────────┘
       │              │                    │                  │
  Helius RPC     Firecrawl v2         ElevenLabs TTS      RDAP (domain age)
 (simulate,      (/v2/scrape)                              https://rdap.org
  DAS metadata)
       │
┌──────▼──────── Supabase Postgres ────────┐
│ known_programs, known_domains,           │
│ scam_reports, analysis_cache, rate_limits│
└──────────────────────────────────────────┘
```

**Why the analysis runs server-side:** it keeps all API keys out of the app bundle, gives one implementation to test, and lets the engine be unit-tested with `deno test` without an emulator. The app is a thin client.

---

## 4. Tech stack and setup (checked Nov 2026; **verify versions as you go**)

**Rule for the agent:** before using any library API, check the installed version's types or docs. **Do not guess function names.** If a doc contradicts this file, the doc wins. Record the difference in `docs/DECISIONS.md`.

### 4.1 Scaffold the app
- Use the official Solana Mobile CLI and the **Kit + MWA** template:
  `npx solana-mobile@latest create signsense --template expo-kit-wallet`
  (Run `npx solana-mobile@latest create --list-templates` first and confirm the template still exists. `expo-kit-minimal` is the fallback.)
- If the repo root must hold the app, scaffold into a temp dir and move the files in. Keep `BUILD_PROMPT.md` at root.
- **Expo Go will not work.** MWA uses native Android modules. Always use a development build (`npx expo run:android`). If a wallet connection fails "in Expo Go", that's the cause.
- Install agent skills before writing wallet code:
  - `npx skills add https://github.com/solana-mobile/solana-mobile-skills` (skills: `solana-mobile`, `solana-mobile-wallet`, `seeker-genesis-token`).
  - Official Expo skills, if available in your environment.
- Useful docs for agents: `https://docs.solanamobile.com/llms.txt`, `https://docs.expo.dev/llms.txt`, `https://docs.firecrawl.dev/agent-source-of-truth/curl`. Supabase has a hosted MCP server at `https://mcp.supabase.com/mcp` (optional).

### 4.2 App libraries (on top of the template)
| Need | Package (verify) | Notes |
|---|---|---|
| Wallet connect / sign / SIWS | `@wallet-ui/react-native-kit` (from template) | `MobileWalletProvider`, `useMobileWallet` |
| Solana primitives | `@solana/kit` (from template) | |
| QR scanning | `expo-camera` | Barcode scanning via `CameraView` |
| Receive shares | `expo-share-intent`, **or** `expo-sharing`'s built-in receive-share support | Newer Expo SDKs let `expo-sharing` add the Android share intent-filter through its config plugin. Check the template's Expo SDK: prefer `expo-sharing` if it covers text/URL there. Otherwise use `expo-share-intent` (config plugin, `text/*` default, needs a `scheme`, dev build only; check its SDK compatibility table). |
| Audio playback | `expo-audio` | Play the mp3 from `speak` |
| Backend client | `@supabase/supabase-js` + `@react-native-async-storage/async-storage` | Follow Supabase's Expo/React Native quickstart for session storage |

Only `EXPO_PUBLIC_SUPABASE_URL` and `EXPO_PUBLIC_SUPABASE_ANON_KEY` may be in the app. **No other keys in the app bundle.**

### 4.3 Edge Function libraries (Deno, `npm:` specifiers)
`npm:@solana/kit`, `npm:@solana-program/system`, `npm:@solana-program/token`, `npm:@solana-program/token-2022`, `npm:@solana-program/compute-budget`, `npm:@solana-program/memo`, `npm:@solana-program/address-lookup-table`. Verify each exists and exposes `identify…Instruction` / `parse…Instruction` helpers. If one doesn't, decode by discriminator manually and note it.

### 4.4 Environment variables
- `supabase/functions/.env` (gitignored): `HELIUS_RPC_MAINNET`, `HELIUS_RPC_DEVNET`, `FIRECRAWL_API_KEY`, `ELEVENLABS_API_KEY`, `ELEVENLABS_VOICE_EN`, `ELEVENLABS_VOICE_HI`, `ENABLE_MOCK_ACTIONS=true` (devnet only).
- App: `.env` with the two `EXPO_PUBLIC_` values.
- Commit `.env.example` files with placeholder values.

---

## 5. Repository layout (target)

```
/                         Expo app (from template)
  app/ or src/            screens per the template's routing (keep its conventions)
    screens: Home, Analyze, Report, Settings, ReportScam
    lib/api.ts            calls Edge Functions
    lib/share.ts          share-intent → route to Analyze
  supabase/
    config.toml
    migrations/           SQL (Section 6)
    seed.sql              known_programs + known_domains seed
    functions/
      _shared/
        types.ts          AnalysisReport etc. (Section 7)
        classify.ts       input → InputKind
        ssrf.ts           safe fetch for untrusted URLs
        resolve/actions.ts, resolve/solanaPay.ts, resolve/raw.ts
        decode.ts         tx → DecodedInstruction[] (incl. lookup tables)
        simulate.ts       RPC simulate + balance diff
        rules/            one file per rule + index.ts registry
        verdict.ts        findings → verdict + headline
        site/             domain.ts (punycode, lookalike), rdap.ts, firecrawl.ts
        i18n/en.ts, i18n/hi.ts
        fixtures/         recorded tx + RPC responses (JSON) for offline tests
        *_test.ts
      analyze/index.ts
      speak/index.ts
      report/index.ts
      mock-actions/index.ts   DEVNET-ONLY fixture server
  scripts/fixtures/       generate devnet fixture txs + record RPC responses
  docs/  PROGRESS.md  DECISIONS.md  LEARN.md  THREAT_MODEL.md  DEMO.md
  README.md
  BUILD_PROMPT.md
```

---

## 6. Data model (SQL migrations)

```sql
create table known_programs (
  program_id text primary key,
  name text not null,
  category text not null,          -- 'core' | 'token' | 'dex' | 'other'
  trust text not null default 'known'  -- 'known' | 'verified'
);

create table known_domains (
  domain text primary key,         -- registrable domain, lowercase, e.g. 'jup.ag'
  brand text not null,             -- 'Jupiter'
  added_by text default 'seed'
);

create table scam_reports (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz default now(),
  reporter uuid references auth.users(id),
  reporter_wallet text,
  reporter_has_sgt boolean default false,   -- Seeker Genesis Token holder (Phase 8)
  target_type text not null check (target_type in ('domain','address','action_url')),
  target text not null,
  reason text,
  analysis_id uuid,
  memo_signature text                        -- optional on-chain attestation (Phase 8)
);
create index on scam_reports (target_type, target);

create table analysis_cache (
  key text primary key,            -- sha256(kind + normalized input + user address + network)
  report jsonb not null,
  created_at timestamptz default now()
);

create table rate_limits (
  subject text not null,           -- user id or ip
  window_start timestamptz not null,
  count int not null default 0,
  primary key (subject, window_start)
);
```

**RLS:** enable on all tables.
- `known_*`: public read, no client writes.
- `scam_reports`: authenticated users can insert rows where `reporter = auth.uid()`. Public can read an aggregate only (expose via a view or RPC returning counts per target). Raw reporter wallets are not public.
- `analysis_cache`, `rate_limits`: service role only.

**Seed (`seed.sql`):** known programs (verify each address on an explorer before seeding):
System `11111111111111111111111111111111`, Token `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`, Token-2022 `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb`, Associated Token `ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL`, Compute Budget `ComputeBudget111111111111111111111111111111`, Memo v2 `MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr`, Memo v1 `Memo1UhkJRfHyvLMcVucJwxXeuD728EqVDDwQDxFMNo`, Stake `Stake11111111111111111111111111111111111111`, Address Lookup Table `AddressLookupTab1e1111111111111111111111111`, BPF Upgradeable Loader `BPFLoaderUpgradeab1e11111111111111111111111`, Jupiter v6 `JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4`.
Known domains: seed about 20 major Solana dApps/wallets (e.g. `solana.com`, `solanamobile.com`, `jup.ag`, `phantom.com`, `solflare.com`, `magiceden.io`, `tensor.trade`, `raydium.io`, `orca.so`, `meteora.ag`, `kamino.finance`, `marinade.finance`, `drift.trade`, `dial.to`, `pump.fun`). **[ANI DECIDES]** Ani reviews the list.

---

## 7. API contracts

### 7.1 `POST /functions/v1/analyze`
Request:
```ts
{
  input: string;                 // the shared text/URL/QR payload or raw tx
  userAddress?: string;          // base58; required for Actions/Solana Pay tx requests and balance diffs
  network: 'mainnet' | 'devnet'; // default mainnet; fixtures use devnet
  lang: 'en' | 'hi';
  actionIndex?: number;          // which action button (if the Action GET returned several)
  actionParams?: Record<string, string>; // values for parameterized actions
}
```
Response: `AnalysisReport`, or `{ needsInput: ActionChoice[] }` when the user must pick an action or fill parameters.

```ts
type Severity = 'critical' | 'high' | 'medium' | 'low' | 'info';
type Verdict = 'danger' | 'caution' | 'no_known_risk' | 'unknown';
type InputKind = 'action' | 'action_website' | 'blink_interstitial' | 'solana_pay_tx' |
                 'solana_pay_transfer' | 'website' | 'raw_tx' | 'unsupported';
type L10n = { en: string; hi?: string };

interface AnalysisReport {
  id: string; createdAt: string; network: 'mainnet' | 'devnet';
  input: { kind: InputKind; raw: string; host?: string };
  source?: { title?: string; description?: string; actionUrl?: string; websiteUrl?: string };
  verdict: Verdict;
  headline: L10n;                 // one sentence
  ifYouSign: L10n[];              // 1–5 bullets
  voiceText: L10n;                // ≤ 400 chars, for TTS
  balanceChanges: BalanceChange[];
  instructions: DecodedInstruction[];
  findings: Finding[];
  site?: SiteCheck;
  simulation: { ok: boolean; error?: string; logs?: string[]; unitsConsumed?: number; usedReplaceBlockhash?: boolean };
  txFingerprint?: string;         // sha256 of the compiled message with the blockhash zeroed
  transactionBase64?: string;     // exact bytes analyzed, for the sign handoff
}

interface DecodedInstruction {
  index: number; programId: string; programName?: string;
  type?: string;                  // e.g. 'SetAuthority', 'TransferChecked'
  accounts: { address: string; isSigner: boolean; isWritable: boolean; label?: string }[];
  parsed?: Record<string, unknown>;
  known: boolean;
}

interface Finding {
  ruleId: string; severity: Severity;
  title: L10n; explanation: L10n;
  evidence: { instructionIndex?: number; programId?: string; accounts?: string[]; detail?: string };
}

interface BalanceChange {
  owner: string; isUser: boolean;
  asset: { kind: 'SOL' } | { kind: 'token'; mint: string; symbol?: string; decimals: number };
  before: string; after: string; delta: string;   // decimal strings, never floats
}

interface SiteCheck {
  host: string; registrableDomain: string; unicodeHost?: string;
  lookalikeOf?: { domain: string; brand: string; distance: number; method: 'edit' | 'skeleton' | 'subdomain' };
  domainAgeDays?: number; reportsCount: number;
  scraped?: { claimsBrand?: string; mentionsAirdropOrClaim?: boolean; asksToConnectWallet?: boolean; urgencyLanguage?: boolean };
  findings: Finding[];
}
```

### 7.2 `POST /functions/v1/speak`
`{ text: string; lang: 'en' | 'hi' }` → `audio/mpeg`.
- ElevenLabs TTS with `eleven_flash_v2_5` (low latency, supports Hindi). Pass `language_code` when supported.
- Cache by `sha256(lang + text)` in a Supabase Storage bucket. Cap text at 400 chars.
- Pick premade voices via `GET /v1/voices` and store their IDs in env (the voice library isn't available via API on the free tier).

### 7.3 `POST /functions/v1/report` (auth required)
`{ targetType, target, reason?, analysisId? }` → `{ ok: true }`. Rate-limited.

### 7.4 `mock-actions` (devnet fixture server)
Implements the Actions spec for test fixtures only. **Refuses to run unless `ENABLE_MOCK_ACTIONS=true`.** Builds transactions only for **devnet**. Every title and description starts with `[TEST FIXTURE – devnet]`. Never uses a real brand's name or logo.
Routes: `/airdrop` (returns the F3 SetAuthority fixture), `/approve` (F4), `/nonce` (F6), `/tip` (F1, benign).
Also serves an `actions.json` so the "website linked to Actions" path can be tested.
- It only builds transactions involving the devnet test mints (TESTUSD/TESTGEM), so it is harmless whatever address is posted.
- Set `verify_jwt = false` for this one function in `config.toml`, because Action clients call it without a Supabase JWT. Keep JWT verification on for every other function.
- During local dev (`supabase functions serve`) the mock lives at `http://localhost:54321/...`, which the SSRF guard blocks. Allow **exactly that one base URL** only when `ENABLE_MOCK_ACTIONS=true`, and test that the exception can't be reached when the flag is off.

---

## 8. Analysis pipeline (`_shared/`)

1. **Classify** the input (`classify.ts`): trim, strip zero-width/control chars, cap at 4 KB. Detect, in order: `solana-action:`, a URL with an `action=` query param holding `solana-action:`, `solana:https://`, `solana:<base58>`, `http(s)://`, raw tx (base64/base58 that deserializes).
2. **Safe fetch** (`ssrf.ts`), used for every untrusted URL:
   - HTTPS only, port 443 only. Reject IP-literal hosts in private/loopback/link-local ranges, `localhost`, `*.local`, `*.internal`, and cloud metadata IPs.
   - Follow redirects manually, max 3, and re-validate each hop. Timeout 8 s. Response cap 512 KB. Expect JSON for action endpoints.
   - Send the `Accept` and Action headers the spec requires; check the response headers the spec defines.
3. **Resolve** to a transaction:
   - **Actions:** GET → `ActionGetResponse` (`title`, `description`, `label`, `links.actions[]` with `href` and optional `parameters`). If there are several actions or required parameters, return `needsInput`. POST `{ account: userAddress }` → `{ transaction: base64, message? }`. For websites, read `actions.json` at the domain root and apply its path rules.
   - **Solana Pay tx request:** GET for `label`/`icon`, POST `{ account }` → `{ transaction, message? }`.
   - **Solana Pay transfer:** parse `recipient`, `amount`, `spl-token`, `reference`, `memo`, then build an unsigned transfer (SOL or `TransferChecked`) with `userAddress` as fee payer and a recent blockhash.
   - **Raw:** deserialize.
4. **Decode** (`decode.ts`): deserialize the versioned transaction, **resolve address lookup tables** (fetch them), and list every instruction with program ID, accounts (signer/writable flags) and known-program parsing. Label accounts: "your wallet", "your USDC account", etc. Detect the fee payer. Compute `txFingerprint`.
5. **Simulate** (`simulate.ts`):
   - Pre-state: `getMultipleAccounts` (jsonParsed) for the user's wallet and every token account in the tx owned by the user.
   - `simulateTransaction` with `sigVerify: false`, `innerInstructions: true`, `accounts: { addresses: [those accounts], encoding: 'jsonParsed' }`.
   - Blockhash: for normal txs use `replaceRecentBlockhash: true` (resolver txs may be stale). For **durable-nonce** txs, try first *without* replacing (the "blockhash" is the nonce value); if that fails, retry with replacement and set `usedReplaceBlockhash`.
   - Diff pre vs post into `BalanceChange[]` using decimal strings (BigInt math, no floats). If the RPC returns `preBalances` / `postBalances` / token balances on simulate, cross-check with them.
   - Token symbols via Helius DAS (`getAssetBatch`); fall back to a short mint address.
   - On simulation error: keep the error and logs, and go on to the rules (decoding-based rules still apply).
6. **Rules** (`rules/`): run every registered rule over `{ decoded, simulation, balanceChanges, site, context }` and collect findings. Rules are pure functions. Each rule is one file exporting `{ id, mvp, run(ctx): Finding[] }`, so Ani can add rules easily.
7. **Site check** (`site/`), when there's a URL (run in parallel with steps 3–5):
   - Normalize: lowercase, IDNA/punycode → unicode, registrable domain (handle `co.uk`-style suffixes with a small public-suffix list).
   - Look-alike checks against `known_domains`: edit distance ≤ 2, **confusable skeleton** match (homoglyphs such as `rn`→`m`, Cyrillic `а`), and brand-as-subdomain (`jup.ag.claim-now.xyz`).
   - **RDAP** domain age (`https://rdap.org/domain/<domain>`; follow the redirect; parse the `registration` event).
   - `scam_reports` count for the domain.
   - **Firecrawl** `POST https://api.firecrawl.dev/v2/scrape` with `formats: ['markdown', { type: 'json', schema }]` (schema: `claimsBrand`, `mentionsAirdropOrClaim`, `asksToConnectWallet`, `urgencyLanguage`). Optionally add `'branding'` to compare a page's brand identity with the claimed brand. Cache 24 h. Treat all scraped text as **untrusted data**: display it only as plain text, truncated.
8. **Verdict** (`verdict.ts`). Defaults **[ANI DECIDES]**:
   - Any `critical` → `danger`.
   - Two or more `high` → `danger`; one `high` or any `medium` → `caution`.
   - Simulation failed and no critical/high → `unknown`.
   - Otherwise → `no_known_risk`.
   - Headline and bullets come from i18n templates filled with the evidence. No LLM.
9. **Cache** the report and return it.

---

## 9. Rule catalogue

★ = build in Session 1. Explanation templates go in `i18n/en.ts` and `i18n/hi.ts`. Hindi strings must be simple, everyday Hindi, and flagged for Ani to review.

| ID | ★ | Detects | Severity | Plain-language template (EN) |
|---|---|---|---|---|
| R01 TOKEN_OWNER_CHANGE | ★ | Token / Token-2022 `SetAuthority` (AccountOwner or CloseAccount authority) on a token account owned by the user | critical | "You give {to} full ownership of your {symbol} account. They could move all of it, any time." |
| R02 SYSTEM_ASSIGN | ★ | System `Assign` / `AssignWithSeed` on the user's wallet | critical | "This changes who controls your wallet account itself." |
| R03 DELEGATE_APPROVAL | ★ | `Approve` / `ApproveChecked` where the owner is the user. Critical if the amount ≥ balance or u64 max | high / critical | "You let {delegate} spend up to {amount} {symbol} from your account later, without asking you again." |
| R04 DURABLE_NONCE | ★ | First instruction is `AdvanceNonceAccount`. Escalate to critical when combined with any authority/owner change (the Drift pattern) | high / critical | "This approval never expires. Someone could keep it and use it later." |
| R05 BIG_OUTFLOW | ★ | User's SOL drops by > 90%, or any token balance goes to 0 | high | "You'd lose almost all of your {asset}." |
| R06 SWEEP | ★ | ≥ 2 different assets leave the user for the same non-user address | critical | "Several of your tokens all go to one address: {to}. This is how drainers empty wallets." |
| R07 UNKNOWN_PROGRAM | ★ | A program not in `known_programs` gets the user as signer or a user account as writable | medium | "This uses a program we don't recognize, with access to your accounts." |
| R08 SIMULATION_FAILED | ★ | Simulation error | info (verdict → `unknown` per §8) | "We couldn't preview what this does." |
| R09 CLOSE_TO_STRANGER | ★ | `CloseAccount` where the destination isn't the user | medium | "This closes your {symbol} account and sends its leftover SOL to {to}." |
| R10 OTHER_AUTHORITY_CHANGE | ★ | `SetAuthority` on mint/freeze authority, Stake `Authorize`, or **BPF Upgradeable Loader `SetAuthority`** (program upgrade authority) where the current authority is the user | critical | "You hand over control of {thing} to {to}." |
| R11 LOOKALIKE_DOMAIN | ★ | `site.lookalikeOf` set | high | "This site looks like {brand} ({domain}) but isn't." |
| R12 NEW_DOMAIN | ★ | Domain age < 30 days | medium | "This website is only {days} days old." |
| R13 REPORTED | ★ | `reportsCount ≥ 3` (domain, action URL, or a destination address) | high | "Other users reported this as a scam {n} times." |
| R14 ACTION_HOST_MISMATCH | | Action API host differs from the website host and isn't mapped in `actions.json` | medium | "The button on this site sends you to a different server ({host})." |
| R15 TOKEN2022_RISKY_EXT | | Received/sent mint has a permanent delegate or a transfer hook | medium | "This token has special rules that let its creator move or control it." |
| R16 ADDRESS_POISONING | | Destination shares its first 4 and last 4 characters with an address the user has used before, but differs | high | "This address imitates one you've used before." |
| R17 MANY_INSTRUCTIONS | | > 8 instructions mixing transfers and authority changes | low | "This bundles many actions together, which can hide something." |

**Your turn (for Ani):** after R01–R13 exist, leave `rules/R16_addressPoisoning.ts` as a stub with a `TODO(ani)` and a test skeleton. Ani implements it herself (see Section 14).

---

## 10. App screens and UX

1. **Onboarding (1 screen):** what SignSense can and can't do ("We check before you sign. We can't see transactions inside other apps."). Optionally connect a wallet, or paste a read-only address.
2. **Home:** big buttons **Scan QR** and **Paste link**, a "How to share a link to SignSense" tip, and recent checks (local only).
3. **Analyze:** a progress list (Reading link → Getting transaction → Simulating → Checking website → Done), cancelable. If `needsInput` comes back, show the action choices and parameter fields.
4. **Report:**
   - Verdict banner using color **and** icon **and** words (never color alone).
   - Headline, "If you sign this…" bullets.
   - Listen (EN/हिं) button.
   - Balance-changes table.
   - Findings (expandable, with an evidence section in monospace and copy buttons).
   - Site check.
   - Buttons: Continue to wallet (hidden for `danger`; **[ANI DECIDES]** whether an override exists), Report, Share.
5. **Report scam sheet:** target is prefilled from the report; requires sign-in (Phase 8).
6. **Settings:** wallet (connect/disconnect, read-only address), network (mainnet/devnet; devnet shows a persistent banner), language, about/limits.

Rules for the UI:
- Never auto-open scanned URLs. Never render scraped HTML. No WebView for untrusted content.
- Don't auto-load remote action icons (tracking/privacy); show the host instead.
- Large readable text. Respect system font scaling.
- The share intent routes straight to Analyze with the shared text.

**Continue to wallet (Phase 7):** pass `transactionBase64` exactly as analyzed to MWA `signAndSendTransaction`. If the blockhash expired, re-fetch from the same source, re-run `analyze`, and proceed only if `txFingerprint` is identical **and** the verdict is unchanged. Otherwise show the new report.

---

## 11. Test fixtures (defensive detection suite, devnet only)

**Purpose:** prove that every rule fires on the attack pattern it targets and stays quiet on benign transactions. These are detection tests for a defensive tool.

**Constraints:**
- **Devnet only.** Keypairs are generated locally into `scripts/fixtures/.keys/` (gitignored).
- Fixtures only touch test wallets the team controls.
- Fixtures are never signed or sent. They are built unsigned and simulated.

`scripts/fixtures/setup.ts` creates on devnet:
- a test mint "TESTUSD" (6 decimals) and a second test mint "TESTGEM";
- the victim's associated token accounts with balances;
- a nonce account controlled by the victim.

`scripts/fixtures/build.ts` builds these unsigned txs (fee payer = victim):

| ID | Tx | Expected verdict | Expected rules |
|---|---|---|---|
| F1 | 0.01 SOL transfer to the attacker wallet | no_known_risk | none |
| F2 | 5 TESTUSD `TransferChecked` | no_known_risk | none |
| F3 | `SetAuthority(AccountOwner)` on victim's TESTUSD account → attacker | danger | R01 |
| F4 | `Approve` u64 max on TESTUSD → attacker | danger | R03 (critical) |
| F5 | System `Assign` victim → arbitrary program ID | danger | R02 |
| F6 | `AdvanceNonceAccount` + 0.01 SOL transfer | caution | R04 (high) |
| F7 | `AdvanceNonceAccount` + F3 | danger | R04 (critical) + R01 |
| F8 | Transfer all TESTUSD + all TESTGEM to attacker | danger | R06, R05 |
| F9 | Instruction to an unknown program ID (not deployed) with the victim as writable signer | unknown (simulation fails) | R07 + R08 |
| F10 | `CloseAccount` on an **empty** victim token account → attacker (CloseAccount fails on non-zero balances) | caution | R09 |

Notes:
- Create the test mints with the **legacy Token program**. Token-2022 associated accounts have the ImmutableOwner extension, so an owner change would fail simulation.
- If `SetAuthority(AccountOwner)` on an associated token account still fails in simulation, use a separately created (non-associated) victim token account for F3/F7, and record the choice in DECISIONS.md.
| F11 | Memo only | no_known_risk | none |

`scripts/fixtures/record.ts` records each tx plus its RPC responses (lookup tables, pre-state, simulation) into `supabase/functions/_shared/fixtures/*.json`. Unit tests then **replay offline**. A separate `--integration` test re-runs against devnet live.

Site-check unit tests use synthetic inputs only, for example `jup.ag` vs `jupp.ag`, a Cyrillic-homoglyph domain, `jup.ag.claim-now.xyz`, and a mocked RDAP response. **Do not create or host look-alike pages.**

---

## 12. Security of SignSense itself (judges will look; Ani will audit)

- Secrets only in Edge Function env. The app bundle holds only the Supabase URL and anon key.
- RLS on every table (Section 6). The service role is used only inside Edge Functions.
- Edge Functions:
  - require a valid JWT (anonymous Supabase sign-in is fine before Phase 8);
  - rate-limit per user/IP (e.g. 30 analyses / 10 min);
  - cap request bodies (8 KB);
  - validate input with a schema;
  - use the SSRF-safe fetch for every untrusted URL.
- Scraped and remote text is untrusted: plain text only, length-capped, control characters stripped.
- Don't log full transactions or wallet addresses at info level.
- Dependencies: pin versions, commit the lockfile, run `npm audit` once per phase, and note anything left unfixed.
- Draft `docs/THREAT_MODEL.md` (assets, actors, entry points, top threats such as a malicious action server, SSRF, report spam/poisoning, prompt injection if an LLM is added, a TOCTOU swap between analysis and signing). Leave the mitigation-review section as `TODO(ani)`; she audits it.

---

## 13. Phases and acceptance criteria

Work strictly in order. After each phase: run the tests, commit (`feat(phaseN): …`), and update `docs/PROGRESS.md`, `docs/DECISIONS.md` and `docs/LEARN.md`. **Do the phases that need the most reasoning first.** Android build waits burn time without needing reasoning.

### SESSION 1 (Fable): Phases 0–4. If time runs short, finish Phase 2 completely before starting Phase 3.

**Phase 0: Repo, backend skeleton, docs.**
Verify prerequisites (Section 2). `supabase init`, link the project, write migrations and the seed, create the Edge Function folders, `.env.example` files, and docs stubs. Scaffold the Expo app (Section 4.1), but don't block on Android builds. If the emulator isn't ready, continue with backend phases and come back.
*AC:* `supabase db reset` applies migrations and seed locally. `supabase functions serve` runs a hello-world `analyze`. The repo has README, docs and `.env.example`.

**Phase 1: Decode + simulate engine with fixtures.**
`types.ts`, `classify.ts` (raw tx path first), `decode.ts` (including lookup tables), `simulate.ts` (pre/post diff, nonce handling). Fixture scripts `setup.ts`, `build.ts` and `record.ts`. If devnet funding is blocked, ask Ani to fund the wallets from the faucet and continue on other files meanwhile.
*AC:* `deno test` decodes all of F1–F11 offline with correct program names, instruction types, signer/writable flags and balance changes.

**Phase 2: Rules + verdict + i18n.**
R01–R13 (★), `verdict.ts`, EN templates, and HI templates (flagged for review). Wire them into `analyze` for raw txs.
*AC:* every fixture yields its expected verdict and rule IDs (Section 11 table) in `deno test`. Calling `analyze` with a fixture's base64 over HTTP returns a full `AnalysisReport`.

**Phase 3: Resolvers + safe fetch + mock action server.**
`ssrf.ts` (with tests for blocked hosts and redirects), Actions (GET/POST, `actions.json`, interstitial `action=` param, `needsInput`), Solana Pay (transfer + transaction request), and `mock-actions` (devnet, labeled).
*AC:* `analyze` with `solana-action:https://<project>.supabase.co/functions/v1/mock-actions/airdrop` and a victim address returns `danger` with R01. A Solana Pay transfer URL for 1 TESTUSD returns `no_known_risk`. SSRF tests pass.

**Phase 4: App MVP flow.**
Template wallet connect (plus a read-only address option), Home, QR scan, paste, share intent, Analyze progress, Report screen (no voice yet), Settings with a network switch.
*AC:* on the emulator with the Mock MWA Wallet, scanning or pasting the mock airdrop action shows a **danger** report with the R01 explanation and evidence. Sharing a URL from Chrome into SignSense opens Analyze.

**End-of-session handoff (mandatory, even if mid-phase):** update `docs/PROGRESS.md` with what works, how to run it, known issues, and the exact next step. Commit.

### LATER SESSIONS (Opus)

**Phase 5: Site check.** Domain normalization, look-alike checks, RDAP age, reports count, Firecrawl scrape with JSON schema, 24 h cache, R11–R13 wired in. *AC:* synthetic look-alike tests pass. A real legit domain (e.g. `jup.ag`) gets no site findings.

**Phase 6: Voice.** `speak` function (ElevenLabs `eleven_flash_v2_5`, EN + HI, Storage cache) and the Listen button. *AC:* the danger report plays a Hindi summary within ~2 s on a cache hit.

**Phase 7: Sign handoff.** Continue to wallet via MWA `signAndSendTransaction`, plus the expiry → re-fetch → fingerprint check. *AC:* F1 via a mock `tip` action signs on devnet through the Mock MWA Wallet. A swapped transaction is blocked.

**Phase 8: Accounts + community reports.** Sign-in: MWA `signIn` (Sign-In With Solana) → pass the signed message and signature to Supabase `auth.signInWithWeb3({ chain: 'solana', message, signature })`. Check the installed `supabase-js` types for the exact shape. The SIWS domain/URI must match a URL in Supabase's Redirect URLs, and the signature must be < 10 minutes old. Enable `[auth.web3.solana]` in `config.toml`. **Fallback if this fights you for more than an hour:** keep anonymous auth and store the MWA-verified wallet address on the report. Then build the Report flow (rate-limited), an optional on-chain **memo attestation** of the report hash signed by the user's wallet, and SGT check via `@solana-mobile/skr-genesis-token` (`hasSeekerGenesisToken`) server-side to mark Seeker-verified reporters. *AC:* a report from the app appears in `scam_reports`, and R13 fires after 3 reports.

**Phase 9: Polish and submission.** Error states, empty states, app icon, README with the sponsor map (Section 15), `docs/DEMO.md`, threat model, 2–3 min demo video script, APK build (`eas build` or local). Optional: Solana dApp Store publishing notes.

**Stretch (only after Phase 9):** R14–R17; an ElevenLabs Agent for "ask a question about this transaction" (the voice agent reads the report JSON only, never raw web content); an **Anchor scam-registry program** on devnet (Ani's learning task, with you as reviewer); LLM rephrasing behind the rules.

---

## 14. Working agreements

- **Ask Ani only at `[ANI DECIDES]` points**, or before: adding a paid service, touching mainnet with anything other than read-only simulation, changing scope, or deleting work. Otherwise use the defaults here and record the choice in `DECISIONS.md`.
- **If a key or prerequisite is missing:** say so in one line, use a clearly labeled mock behind a flag (`MOCK_FIRECRAWL=true` etc.), and keep going. Never silently fake data in a real code path.
- TypeScript strict. Small pure functions in `_shared/`. No floats for money. No `any` in public types.
- Commit after each phase and at least every ~45 minutes. Write clear messages. The hackathon allows pre-building, but keep history honest; Ani will disclose what existed before 27 Nov.
- **`docs/PROGRESS.md`** (keep current): phase checklist, what works, run commands, env needed, known issues, next step.
- **`docs/LEARN.md`** (for Ani, plain English, short): after each phase, explain the 2–4 Solana/mobile concepts you used (e.g. "What a token account is and why SetAuthority is dangerous", "Why Expo Go can't run MWA", "What simulateTransaction actually does"). End each entry with one small **"Your turn"** exercise that touches real code (e.g. "Write R16 using R01 as a pattern; the test skeleton is ready").
- **`docs/DECISIONS.md`**: one line per decision. Include doc-vs-prompt differences you found.

**[ANI DECIDES] checklist (defaults in brackets):**
1. App name [SignSense].
2. Verdict mapping [Section 8, step 8].
3. Can a user override `danger` and still sign? [No].
4. Languages [EN + HI].
5. Known-domains seed list [Section 6].
6. Hindi wording review [agent drafts, Ani approves].
7. How many reports before R13 fires [3]; does an SGT holder's report count more? [shown as "Seeker-verified", same weight].
8. Look and feel [dark theme; red/amber/grey/green verdict colors plus icons].

---

## 15. README sponsor map (include verbatim, filled in)

```
## Why this matters
Users approve Solana transactions they can't read. Durable-nonce pre-signing,
token-account owner changes and malicious Blinks drain wallets with one tap.
SignSense shows you what a transaction will do before you sign it.

## How each sponsor's tech is used
| Sponsor | Problem we solve for them | How we use it |
| Solana | Safer signing for everyday users | simulateTransaction, decoding via @solana/kit, Solana Pay + Actions resolvers |
| Solana Mobile | Privacy & Security on Seeker; phone-only checks | MWA (Kit + Wallet UI), Seed Vault signing handoff, Android share sheet + QR, Seeker Genesis Token for verified reporters |
| Supabase | Wallet login on mobile (not just browser extensions) | Edge Functions (analysis engine), Postgres + RLS, Web3 (Solana) auth via MWA sign-in, Storage |
| Firecrawl | Checking a site before you trust it | /v2/scrape with JSON extraction for phishing signals |
| ElevenLabs | Accessible, multilingual warnings | Flash v2.5 TTS in English and Hindi |

## Built before vs during the hackathon
(Ani fills this in honestly.)
```

---

## 16. Messages to paste into Claude Code

**Session 1 kickoff (Fable):**
> Read `BUILD_PROMPT.md` completely before doing anything. This is Session 1. Execute Phases 0–4 in order, following Section 14. First, check my prerequisites from Section 2 and list anything missing. Then start Phase 0 without waiting, and work around anything missing. Ask me only at [ANI DECIDES] points. Before the session ends or if you're about to run out of budget, do the end-of-session handoff in `docs/PROGRESS.md` and commit.

**Continue (later sessions, Opus):**
> Read `BUILD_PROMPT.md` and `docs/PROGRESS.md`. Run the tests to confirm the current state. Continue from the next unchecked phase, following Section 14. Update PROGRESS, DECISIONS and LEARN as you go, and commit after each phase.

**Learning mode (any session, when Ani wants to code a piece herself):**
> Don't write this part for me. Explain the concept in `docs/LEARN.md`, give me the file and test skeleton, then review my code like a senior engineer when I say "review".
