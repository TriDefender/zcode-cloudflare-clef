# zcode-cloudflare-clef

[中文文档](./README_CN.md)

[ZCode](https://zcode.z.ai) plugin: structured decision evaluation on [Cloudflare Workers AI](https://developers.cloudflare.com/workers-ai/) **CLEF / CLEF Flash**. Instead of letting the chat model guess, the MCP tool `clef_evaluate` sends a concrete state plus typed questions (boolean / choice / score) to a small calibrated model and returns per-question answers with probability distributions — useful for triage, routing, severity grading and other judgments where calibrated confidence matters more than eloquence.

- MCP stdio server, zero npm dependencies (hand-rolled JSON-RPC framing; Node ≥ 22 on PATH).
- Protocol layer (encoding, response validation, image extraction, context budgeting) ported verbatim from [`openclaw-cloudflare-clef` 0.1.3](https://github.com/openclaw) (MIT).
- Credentials are **DPAPI-encrypted at rest** (Windows) or supplied via environment variables; never logged, never written to config files in plaintext.

## Install (ZCode Desktop)

1. **Plugin Marketplace → Add → Add Plugin Marketplace**, paste this repository URL: `https://github.com/TriDefender/zcode-cloudflare-clef`
2. **Personal → cloudflare-clef → Install**
3. Store credentials (below) and fully restart ZCode.

## Credentials

1. Get your **Account ID** (32-hex, Cloudflare dashboard → Workers & Pages overview) and create an **API Token** with **Workers AI → Run** permission (My Profile → API Tokens).
2. Run once in PowerShell (interactive, token input hidden):

   ```powershell
   powershell -ExecutionPolicy Bypass -File "<plugin-directory>\server\store-secret.ps1"
   ```

   This stores both values DPAPI-encrypted at `%USERPROFILE%\.zcode\secrets\cloudflare-clef.json`, bound to the current Windows user — no plaintext on disk. Re-run to rotate; delete the file to revoke.
3. Non-Windows / alternative: set user-level environment variables `CLOUDFLARE_ACCOUNT_ID` and `CLOUDFLARE_API_TOKEN`. Real environment variables take precedence over the encrypted store.

Honest security boundary: DPAPI protects against file exfiltration (backups, sync, copied disks), not against other processes running as the same Windows user.

## Usage

Ask the agent to use `clef_evaluate`. Questions are a map of arbitrary ids (Chinese and slashes welcome) to typed questions:

```json
{
  "model": "clef-flash",
  "state": "Cargo not yet telex-released, ETA tomorrow, needs origin-port follow-up.",
  "questions": {
    "urgent": { "type": "boolean", "instructions": "Is this urgent?" },
    "route": { "type": "choice", "criteria": { "customs": "Customs clearance", "ocean": "Ocean freight" } },
    "severity": { "type": "score", "criteria": ["None", "Moderate", "Critical"] }
  }
}
```

Returns `{model, answers, usage}`; answers echo the original ids. Omit `model` for `clef-flash` (faster/cheaper); `clef` is stronger. Input-only pricing: 3,455 Neurons / M input tokens (clef-flash) and 21,818 / M (clef) as of 2026-10-20; Workers Free grants 10,000 Neurons per UTC day.

## Scope and side effects

- Network access: POSTs to `https://api.cloudflare.com/client/v4/accounts/<accountId>/ai/run/@cf/cloudflare/<model>` only. No redirects, no telemetry, no retries.
- Reads two environment-derived secrets at MCP startup; writes nothing outside the process.
- Evidence up to ~24k text bytes and up to 4 images (PNG/JPEG/WebP, 4 MiB each / 8 MiB total / 16 MP) per request; response bodies capped at 1 MiB.

## Development

```shell
node --test cloudflare-clef/server/protocol.test.mjs   # 15 protocol tests
```

## License

MIT — see [LICENSE](./LICENSE). The protocol layer derives from the MIT-licensed OpenClaw plugin `openclaw-cloudflare-clef` 0.1.3.
