---
prompts:
  - prompt: "Give customers a wallet on Postgres: top up with Stripe Checkout, pay orders from the balance, by card or split between both, and send refunds back where the money came from."
    stack: Postgres
  - prompt: Sell prepaid credit packs, keep balances in PLN and EUR on Firestore, and show each customer the full history of every movement.
    stack: Firestore
  - prompt: Let staff issue store credit as a refund or a goodwill gesture, with the reason recorded on every entry.
---

# Prompts

What an operator types after installing this skill, in their own words. An agent eval installs the skill
into an empty Next.js app, gives the agent one of these prompts and no further help, then type-checks, builds
and tests the result; the first prompt runs before every release. The results are the other files in this
folder. Section 10 of [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) says how a
run is made. The prompts and the newest runs are on
[the skill's page](https://timerise.ai/skills/ledger-wallet) on timerise.ai.
