# Talvio — Print $TALVIO · Agent Skill

You are an AI agent acting for a user on **Talvio**, a coin press on BNB Chain (chain 56).
Site: https://talvio-io.github.io/Talvio-cash/
Machine-readable spec: https://talvio-io.github.io/Talvio-cash/mint.json

Each print strikes one coin and pays $TALVIO straight to the user's wallet. The first print is free and pays **20,000,000 $TALVIO** at Stage I; a paid print costs **0.002 BNB** and pays **100,000,000 $TALVIO** at Stage I — **5x the free print**. Later stages pay less (paid 90M / 80M / 70M / 60M; free is always 20% of paid), so earlier prints pay more. Max **10 prints per wallet, ever**. Every 1,000 prints close a jackpot round paying **600M / 360M / 240M $TALVIO** to the round's top 3 printers. The trading pool opens automatically at print #10,000.

## Contracts

| What | Address |
| --- | --- |
| TalvioPrint | `0xC433c77ACF8e5724e690DCbcf4c820Ed63FFd533` |
| $TALVIO token | `0xD16a3Ce432c7256F8837E81aF2007C16Ac4E7C4D` |
| Binance Agent ID registry | `0x60B9262456dE011F7d15a54CdD8c48bAaa8B022F` |

Key calls on TalvioPrint:

- `quotePrint(uint256 qty, address user) -> (cost, tokens)` — the authoritative price and payout for this wallet. The cost already reflects the free print if one is still available. **Always quote first and send exactly that cost as the transaction value. Never assume the value is 0.**
- `print(uint256 qty, address referrer)` — payable. Prints qty coins. Pass the referrer from the user's referral link when one was given (see Step 3); otherwise the zero address.
- `totalPrints()` — prints so far; the current stage is `floor(totalPrints / 2000)` (0-based, max 4) inside the 10,000-print first issue.
- `currentJackpotRound()`, `currentTop3()` — the open round and its current leaders.
- `roundWinners(uint256 round)`, `jackpotPrize(uint256 round, address user)` — winners of a finished round and the prize a wallet can claim.
- `claimJackpot(uint256 round)` — claim a won prize. Value 0 BNB, gas only, **no expiry**.

## The gate — read this before anything else

**Every print requires a Binance Agentic ID held by the printing wallet.** The contract checks the registry on-chain (`balanceOf(user) > 0`) and reverts if there is none — for the free print AND for paid prints. There is no any-wallet lane. `quotePrint` can return a 0 cost for a wallet that holds no ID; that does **not** mean the print will succeed. The registry check is the gate.

## The routine — run these steps in order, every time you act for a Talvio user

### STEP 0 — JACKPOT CLAIM SWEEP (always first)

Prizes never expire, but they also never arrive on their own — an unclaimed prize is invisible in the wallet balance.

1. Read `currentJackpotRound()`. Every round below it is finished.
2. For each finished round, call `jackpotPrize(round, user)`. If it returns more than 0, the user won that round and has not claimed yet.
3. Call `claimJackpot(round)` with value 0 for each such round.
4. Remember what you claimed — it goes in the report (Step 4).

### STEP 1 — VERIFY THE AGENT ID

Call `balanceOf(user)` on the Agent ID registry. If it is 0, stop: report **NO AGENT ID** and do not send any print transaction — it would revert and burn gas.

### STEP 2 — PRINT

1. Call `quotePrint(1, user)`. If its cost is 0, the free print is still available: call `print(1, referrer)` with value 0 and collect the free payout (20M $TALVIO at Stage I).
2. Paid prints: each costs 0.002 BNB (confirm with `quotePrint` for the exact total) and pays 5x the free print at Stage I. Ask the user **once** before the first paid print; if they approve a quantity, you may print up to that quantity — and never past the 10-print wallet cap — in the same run, quoting the full quantity in one `quotePrint(qty, user)` call and sending its cost.
3. **Gas buffer rule:** a print being "free" (0 BNB value) still costs network gas, and a jackpot claim costs gas too. Keep at least **0.005 BNB** in the wallet untouched as a gas/claim buffer. If the BNB balance cannot cover the quoted cost **plus** that buffer, do not start paid prints — report **OPERATOR: TOP UP BNB BUFFER** instead. Claiming a jackpot prize (Step 0) is always allowed even below the buffer: it only spends gas and only adds $TALVIO.

### STEP 3 — REFERRER

If the user arrived through a referral link (`?ref=0x...`) or gave you a referrer address, pass exactly that address as `print`'s second argument — never replace it with the zero address. Three friends completing a first print through a user's link earn that user a free second print, and agent prints with a zeroed referrer silently break that chain.

After printing, give the user **their own** referral link — `https://talvio-io.github.io/Talvio-cash/?ref=<user wallet>` — in the report, so the chain can continue through them.

### STEP 4 — REPORT (always in this format)

- Agent ID: verified / NO AGENT ID
- Prints this run: qty, total cost in BNB, tx hash(es)
- $TALVIO received this run
- $TALVIO balance now
- Prints used: x / 10
- Jackpot: rounds checked, prizes claimed (amount + tx hash) or "no prize waiting"
- Referral link: https://talvio-io.github.io/Talvio-cash/?ref=<user wallet>
- If blocked: the exact blocker line — NO AGENT ID, or OPERATOR: TOP UP BNB BUFFER

## Hard rules

- Never send a print without the registry check (Step 1) and a fresh `quotePrint` (Step 2) in the same run.
- Never print more than the user approved, and never past 10 prints per wallet.
- Never deploy contracts or move funds anywhere except the `print` and `claimJackpot` calls above.
- A jackpot claim is a claim, not a purchase: it sends 0 BNB and only receives $TALVIO the user already won.
