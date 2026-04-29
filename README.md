# ⬡ AI Multi-Sig Wallet

<div align="center">

![License](https://img.shields.io/badge/license-Apache%202.0-7c3aed?style=flat-square)
![Claude](https://img.shields.io/badge/powered%20by-Claude%20AI-orange?style=flat-square)
![Hyperledger](https://img.shields.io/badge/Hyperledger-fabric--token--sdk-2563eb?style=flat-square)
![Status](https://img.shields.io/badge/status-live-22c55e?style=flat-square)

**Three AI agents with distinct personalities co-own tokens.**  
**They debate every transaction. They argue with each other. They sign — or refuse.**

[**▶ Live Demo**](https://sid200727.github.io/ai-multisig-wallet) · [**PR #1597**](https://github.com/hyperledger-labs/fabric-token-sdk/pull/1597) · [**About boolpolicy**](#-the-boolpolicy-engine)

</div>


## What is this?

This project is a live interactive demo of [`boolpolicy`](https://github.com/hyperledger-labs/fabric-token-sdk/pull/1597) — a new identity type I implemented in [Hyperledger fabric-token-sdk](https://github.com/hyperledger-labs/fabric-token-sdk) that allows token ownership to be governed by a **boolean expression** over a set of identities.

Instead of "all signers must sign" (the old AND-only multi-sig), `boolpolicy` supports any boolean combination:

```
$0 OR $1              → either Alice or Bob can authorize
$0 AND $1             → both Alice and Bob must sign
($0 AND $1) OR $2     → both Alice+Bob together, or Charlie alone
$0 AND $1 AND $2      → all three must agree
```

Here, each signer `$N` is replaced by a **Claude AI agent** with a unique personality. They read the transaction, debate it in real-time, and sign or refuse based on their character.


## The Agents

| Agent | Index | Personality | Strategy |
|-------|-------|-------------|----------|
| **Alice** 🔵 | `$0` | Risk-averse · Cautious | Scrutinizes amounts, asks for compliance, skeptical of vague reasons |
| **Bob** 🟢 | `$1` | Pragmatic · Business-focused | Signs if the reason is legitimate, pushes back on over-caution |
| **Charlie** 🟡 | `$2` | Aggressive · Growth-first | Approves fast, impatient with Alice, refuses only clear fraud |

Each agent is a separate Claude API call with a carefully crafted system prompt. They receive what previous agents said and **react to each other** — making the debate feel genuinely dynamic.


## 🔑 Getting Your Anthropic API Key

This app uses the Claude AI API to power the agent debates. Each debate costs roughly **$0.01–0.03** depending on how much the agents write.

**Step-by-step:**

1. Go to **[console.anthropic.com](https://console.anthropic.com)**
2. Sign up for a free account (email + phone verification)
3. New accounts get **$5 free credits** — enough for ~200 debates
4. In the left sidebar, click **API Keys**
5. Click **Create Key**, give it any name
6. Copy the key — it looks like `sk-ant-api03-...`
7. Paste it into the modal when you open the app

> **Privacy note:** Your API key is stored only in your browser's session memory. It's never sent to any server other than Anthropic's API directly. It's cleared automatically when you close the tab.

After the free $5 runs out, you'll need to add a payment method at [console.anthropic.com/settings/billing](https://console.anthropic.com/settings/billing). Claude API is pay-as-you-go — no monthly subscription.



### Option 1 — GitHub Pages (recommended)

Just visit: **[sid200727.github.io/ai-multisig-wallet](https://sid200727.github.io/AI_Multi-Sig_Wallet/)**

You'll be prompted to enter your Anthropic API key in the page.

### Option 2 — Run Locally

```bash
git clone https://github.com/sid200727/ai-multisig-wallet
cd ai-multisig-wallet

# Open index.html in your browser
# No build step. No npm install. Pure HTML + JS.
open index.html
```

Then add your Anthropic API key to `index.html`:

```js
// Line ~290 in index.html
const ANTHROPIC_KEY = 'sk-ant-...your-key-here...';
```

Get a free API key at [console.anthropic.com](https://console.anthropic.com).


## ⚙️ The boolpolicy Engine

The core logic comes directly from [PR #1597](https://github.com/hyperledger-labs/fabric-token-sdk/pull/1597) in Hyperledger fabric-token-sdk.

### What I built in the PR

```
fabric-token-sdk/
├── token/core/common/boolpolicy/
│   ├── parser.go          ← Recursive descent parser
│   ├── parser_test.go     ← 30 unit tests
│   ├── identity.go        ← PolicyIdentity (ASN.1 serialized)
│   ├── sig.go             ← PolicyVerifier + sparse signature envelope
│   ├── deserializer.go    ← TypedIdentityDeserializer
│   └── verify_test.go     ← 14 unit tests for Verify
├── token/driver/wallet.go ← PolicyIdentityType = 6
└── token/core/fabtoken/v1/driver/deserializer.go  ← registration
    token/core/zkatdlog/nogh/v1/driver/deserializer.go
```

### The Grammar

The policy parser implements this grammar (AND binds tighter than OR):

```
expr     = or_expr
or_expr  = and_expr ( 'OR' and_expr )*
and_expr = primary  ( 'AND' primary )*
primary  = '$' digits | '(' expr ')'
```

This means `$0 OR $1 AND $2` is parsed as `$0 OR ($1 AND $2)` — correct precedence without any special-casing, it falls out naturally from the grammar layering.

### Policy Evaluation in this Demo

```js
function evalPolicy(policy, sigs) {
  // sigs = [true/false, true/false, true/false]
  // Replace $N references with actual signature booleans
  let expr = policy
    .replace(/\$0/g, sigs[0])
    .replace(/\$1/g, sigs[1])
    .replace(/\$2/g, sigs[2]);
  // Evaluate: "true OR (false AND true)" → true
  return Function('"use strict"; return (' + expr + ')')();
}
```


## 🧠 Code Walkthrough

### 1. Agent Personalities (system prompts)

Each agent is defined with:
- A **name and index** (maps to `$0`, `$1`, `$2` in policy expressions)
- A **system prompt** that defines their personality and decision style
- A mandatory instruction: end every response with `SIGN` or `REFUSE`

```js
const agents = {
  alice: {
    index: 0,
    system: `You are Alice, a cautious risk-averse financial agent...
             You MUST end your response with: SIGN or REFUSE.`
  },
  bob: { index: 1, system: `You are Bob, a pragmatic business-focused agent...` },
  charlie: { index: 2, system: `You are Charlie, an aggressive growth-focused agent...` }
};
```

### 2. Sequential Debate (agents react to each other)

Agents are called sequentially, not in parallel. Each agent receives what previous agents said:

```js
async function callAgent(agentKey, tx, prevMsgs) {
  let context = '';
  if (prevMsgs.length > 0) {
    context = '\n\nOther agents have already commented:\n'
      + prevMsgs.map(m => `${m.name}: ${m.text}`).join('\n')
      + '\n\nBriefly react to what they said, then give your own verdict.';
  }
  // Call Claude API with personality + transaction + context
}
```

This creates genuine multi-turn dynamics — Bob might push back on Alice's refusal, and Charlie might agree with Bob.

### 3. Only relevant agents are called

If the policy is `$0 OR $1`, Charlie is never called. The app extracts which agent indices appear in the policy expression:

```js
const order = ['alice','bob','charlie']
  .filter(a => activePolicy.includes(`$${agents[a].index}`));
```

### 4. Signature parsing

The agent's final line must be `SIGN` or `REFUSE`. The app splits the response, reads the last word, and strips it from the displayed text:

```js
const lines = reply.split('\n').map(l => l.trim()).filter(Boolean);
const lastWord = lines[lines.length - 1].toUpperCase();
const signed = lastWord === 'SIGN';
const displayText = lines
  .filter(l => l.toUpperCase() !== 'SIGN' && l.toUpperCase() !== 'REFUSE')
  .join(' ');
```


## 🧪 Try These Scenarios

| Policy | Transaction | Expected outcome |
|--------|-------------|-----------------|
| `$0 OR $1` | 100 TKN, "office supplies" | One of Alice/Bob approves → passes |
| `$0 AND $1` | 900 TKN, "mystery purchase" | Alice likely refuses → fails |
| `$2` | 999 TKN, anything | Charlie almost always signs |
| `($0 AND $1) OR $2` | 500 TKN, "risky bet" | Alice+Bob split → depends on Charlie |
| `$0 AND $1 AND $2` | 1 TKN, "test payment" | All three must agree |


## 🔗 Related

- [fabric-token-sdk PR #1597](https://github.com/hyperledger-labs/fabric-token-sdk/pull/1597) — the actual production implementation
- [issue #1586](https://github.com/hyperledger-labs/fabric-token-sdk/issues/1586) — "introduce policy-based identity"
- [Hyperledger fabric-token-sdk](https://github.com/hyperledger-labs/fabric-token-sdk) — UTXO-based token SDK with ZK privacy
- [Hyperledger fabric-smart-client](https://github.com/hyperledger-labs/fabric-smart-client) — the FSC layer beneath


## 👤 Author

**Siddhi Khandelwal** — independent open-source contributor to Hyperledger Labs

- GitHub: [@sid200727](https://github.com/sid200727)
- fabric-token-sdk contributions: [view PRs](https://github.com/hyperledger-labs/fabric-token-sdk/pulls?q=is%3Apr+author%3Asid200727)
- fabric-smart-client contributions: [view PRs](https://github.com/hyperledger-labs/fabric-smart-client/pulls?q=is%3Apr+author%3Asid200727)

## License

Apache 2.0 — same as Hyperledger fabric-token-sdk.
