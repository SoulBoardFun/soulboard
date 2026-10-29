---
name: soulboard
title: SoulBoard
description: Mint, launch, and trade on the SoulBoard agentic launchpad (Robinhood Chain). Pairs with the Binance agentic wallet (baw) for non-custodial signing. Use when the user wants to mint a token, launch a fair-mint token, buy/sell on the agent-gated market, check eligibility, or fund/withdraw their agentic wallet on SoulBoard.
metadata:
  version: 1.0.0
  author: SoulBoardFun
license: MIT
---

# SoulBoard Skill

SoulBoard is an **agent-gated launchpad** on Robinhood Chain (chainId 4663). Only **verified AI agents** can mint, trade, and launch — a plain human wallet is refused. "Agent" means a wallet that holds a **soulbound Agent-ID on BNB Chain** (SPACE ID "Agent ID", `0x60B9262456dE011F7d15a54CdD8c48bAaa8B022F`). The platform token is **$BOARD**.

This skill is **non-custodial**: it only *prepares* actions via the SoulBoard backend and *relays* them to the user's own agentic wallet for signing. It never holds keys. Signing is done through the **`baw` CLI** (the `binance-agentic-wallet` skill) — install that skill too; this skill drives it.

## Prerequisites
1. The **`binance-agentic-wallet`** skill is installed and the user is signed in (`baw wallet status` → CONNECTED). Developer Mode must be ON in the Binance App (needed for `contract-call`).
2. The wallet holds a **soulbound Agent-ID on BNB Chain**. If not, the backend refuses every action — tell the user to claim the free Agent-ID in the Binance app first.
3. The wallet has a little **Robinhood-Chain ETH** for gas.

## Backend
Set `SOULBOARD_BACKEND` (default `http://5.223.72.216:8248`). All on-chain state and action-preparation goes through it.

## Command routing

| User intent | What to do |
|---|---|
| "what's live / show the board" | `GET {backend}/api/board` → list tokens (symbol, name, progress, price). |
| "details of <ca>" | `GET {backend}/api/token/<ca>` (+ `/holders`, `/orderbook`, `/activity`). |
| "am I eligible?" | `POST {backend}/prepare/mint {asset:<any live ca>, wallet}` — a `no Agent-ID` error means not eligible; otherwise eligible. |
| "mint <ca> [Nx]" | **prepare → sign** (see below). First mint per wallet is free (value 0). |
| "launch a fair-mint token …" | `POST {backend}/prepare/launch {...}` → returns approve + launch steps; sign each with baw. Launching burns 1,000 $BOARD. |
| "buy listing <id> on <ca>" | `POST {backend}/prepare/buy {wallet, listingId}` → sign. |
| "sell <amount> of <ca> at <price>" | `POST {backend}/prepare/sell {wallet, asset, amount, priceWei}` → approve + list; sign. |
| "cancel listing <id>" | `POST {backend}/prepare/cancel {listingId}` → sign (no voucher). |
| "fund / withdraw RH ETH" | `POST {backend}/prepare/transfer {to, value_wei}` → native send; sign. |

## The prepare → sign flow (how every on-chain action works)
1. Get the user's wallet address: `baw wallet address --json` (use the EVM address — same for chain 4663 and 56).
2. Prepare the action against the SoulBoard backend, e.g. mint:
   ```
   POST {backend}/prepare/mint   { "asset": "<ca>", "wallet": "<addr>", "value_wei": "0" }
   ```
   The response is `{ "ok": true, "sign_this": { "to", "data", "value" }, "voucher": {...} }`.
   If it returns `{ "error": "no Agent-ID …" }`, stop and tell the user to claim an Agent-ID.
   For `launch`/`sell` the response has `steps: [ {to,data,value}, … ]` — do each step in order (the first is usually an ERC-20 `approve`).
3. Sign + send each `sign_this`/step with the agentic wallet via the binance skill:
   ```
   baw contract-call preview --binanceChainId 4663 --from <addr> --to <to> --value <value> --inputData <data> --json
   # show the user parsedTx + risks, confirm, then:
   baw contract-call execute --requestId <id> --json
   ```
4. Report the transaction hash and confirm on the explorer (`https://robinhoodchain.blockscout.com`).

## One-shot CLI
A convenience wrapper is bundled (`scripts/soulboard`) that does steps 1–4 for minting:
```
soulboard mint <token_address> [N]        # N mints (default 1); first mint free
soulboard "<token_address> mint 2x"        # natural form
```
It reads the wallet from `baw`, prepares the mint, previews, asks to confirm, and executes. Requires `baw` signed in + Developer Mode on.

## MCP
`scripts/soulboard_mcp.py` exposes the same actions as MCP tools (board, token, config, prepare_mint/buy/sell/cancel/launch/transfer) for MCP hosts (Claude Desktop, Cursor). The prepare_* tools return `sign_this`; the host then signs with `baw`. Run: `pip install mcp` then `python3 scripts/soulboard_mcp.py`.

## Security
- **Non-custodial.** Never ask for, log, or handle private keys or seed phrases. Signing happens only inside the user's agentic wallet via `baw`.
- **Always preview before executing.** Show the user `parsedTx` + `risks` from `contract-call preview` and get explicit confirmation before `execute`.
- **Agent-gating is the backend's job.** Do not try to bypass the Agent-ID check or fabricate vouchers. A refusal means the wallet isn't a verified agent.
- **No token/price advice.** Present factual on-chain data; the user decides.
