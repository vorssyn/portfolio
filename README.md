# Smart Contract & On-Chain Security

I'm an independent smart contract auditor. I read your core contracts line by line, and then I verify what I found against the live deployment with `cast` calls on mainnet. I don't mean checking the docs, I mean reading the actual state: role grants, fee configs, pool state, oracle setup. Static review surfaces candidates, and the deployment decides which of them are real. That's the step I think most reports skip, and it's the one that separates a suspicion from a confirmed finding.

## Sample work

- **NoRekt (Arbitrum)**: hedged-lending protocol, ~3,200 LOC, 11 contracts. Verdict: no HIGH/CRITICAL. 5 findings (1 confirmed live on-chain), all Low/Medium. -> [`norekt-audit.md`](norekt-audit.md)

The report is the best introduction to how I work. Read the verification appendix first, then decide if that's the level of proof you want on your protocol.

## What you get

- Manual line-by-line review of your core contracts (usually 1,000-5,000 LOC)
- Findings ranked by severity, with the exploit path and impact spelled out and a concrete fix for each
- On-chain verification of every parameter claim: role grants, fee configs, pool state, oracle setup. You get a report you can rerun, not one you have to trust.
- A clean verdict either way. If the code is sound, I say so. I don't manufacture findings to justify the fee.
- A plain-language writeup you can share with users, partners, or future auditors

## Beyond source code

Not everything that touches your funds ships as readable Solidity. Some contracts deploy without source verification, so all that exists is the bytecode. I work from that: heimdall for disassembly and decompilation back to Solidity/Yul, Dedaub's decompiler for high-fidelity reconstruction, and Foundry `cast` for pulling the bytecode and recovering selectors and the ABI. Honeypots, malicious tokens, proxies that hide their logic behind an opaque `delegatecall`: those are bytecode problems, and most auditors won't touch them.

## Why I verify on-chain

Static review finds candidate issues. Deployment reality is where they live or die:

- A `liquidatorFee = 0` default is a lead in code. It becomes a confirmed live finding when `MarginAccount.liquidatorFee()` returns 0 on mainnet.
- A "role might be misgranted" warning is resolved when `hasRole()` shows the role held only by the protocol's own contracts.
- An "oracle might be manipulable" concern is settled by reading the actual feed and sequencer config on-chain.

Every report I ship ends with a verification appendix: addresses, calls, results, reproducible by anyone. I'd rather you check my work than take my word for it.

The on-chain pass only applies to contracts that are deployed and source-verified. If your code isn't live yet, you still get the full static review, and I run the verification the day it goes up.

## Engagement process

1. You send the repo (or addresses) plus scope
2. I review, then verify against the live deployment
3. You get the report: findings, fixes, verification appendix
4. Found a critical? You get a draft disclosure note for your team before anything is public

## Contact

- Email: vorssyn@proton.me
- GitHub: [@vorssyn](https://github.com/vorssyn)
- When you write, include the repo or addresses plus the scope, and let me know your timeline

---

*Small protocols get institutional-quality review without institutional pricing. If your contracts are deployed and unaudited, that's a risk to your users, and to you.*
