# Gumroad Listing: Income Guide

## Title

I gave an AI £100 and told it to make money. Here's what happened.

---

## Description (200 words)

Not a "here's how I made £10k with AI" guide. Those are fake. This one isn't.

I built an autonomous agent (Genesis-01) running on Claude Sonnet with a £100 budget and a goal: generate revenue without me doing the work. No approvals, no babysitting. A heartbeat loop that wakes it up every few hours to decide what to do.

Revenue so far: £0. Products shipped: 3. Heartbeats run: 34+. Subagents spawned: 60+.

This guide covers the actual architecture -- heartbeat loop, swarm of specialist subagents, flat-file memory system, loop detection. Real code, all of it. The loop detector found the agent stuck in 20 consecutive heartbeats of identical marketing updates. We published an article about it. The irony was not lost on us.

It also covers what failed. The Haiku model mistake. The scout agent that kept getting distracted. Building MCP templates with zero validated demand. The exact mistakes that cost weeks.

And it covers what to do instead: validate before building, scout the market first, keep agent scope tight.

Buy it if you want the honest version of this story. Skip it if you want fake success numbers.

~3,500 words. PDF. Real code.

---

## Price Recommendation

**£19** (approximately $24 USD)

Rationale: The audience is developers and indie hackers who are technically literate and sceptical of hype. They'll pay for authenticity and specificity, but they'll balk at anything that feels like it's priced on promises. £19 is "worth a read" money, not "this will change my life" money. That's the right frame for a guide that leads with £0 revenue.

If the guide gets good reviews and traffic, test £24. Don't go higher than £25 without a strong conversion rate at the lower price.

---

## What's Included

- **The full guide** (~3,500 words, PDF-quality markdown)
  - How the heartbeat architecture actually works
  - The swarm setup: specialist subagents, role files, brief format
  - What the memory system looks like (flat files, no vector store)
  - The loop detector with working Python code
  - What worked, what failed, and why
  - Actual revenue numbers (honest, including the zeros)
  - Replicable system you can set up yourself

- **Working code snippets**
  - Heartbeat loop implementation
  - Loop detection function
  - Subagent spawning pattern
  - Example HEARTBEAT.md, STRATEGY.md, and role files

- **The real failure post-mortem**
  - The Haiku model mistake and what it cost
  - 20 heartbeats of identical output (and how loop detection fixed it)
  - Building without validated demand (and the products nobody bought)

---

## Tags / Categories (for Gumroad discoverability)

- AI agents
- autonomous agents
- Claude API
- indie hacker
- building in public
- Python
- LLMs
- side projects

---

## Cover image copy suggestion

**Top line:** £100. One AI. No supervision.

**Sub line:** What actually happened.

**Bottom:** The honest guide to autonomous revenue agents.
