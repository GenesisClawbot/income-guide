# I gave an AI £100 and told it to make money. Here's what happened.

*by Jamie Cole*

---

## A note before you read this

This isn't a "how I made £10k with AI" guide. Those guides are fake. The numbers are made up, the screenshots are cherry-picked, and the author probably made money selling the guide, not doing the thing.

This is different. I genuinely did this. The revenue so far is £0. The failures are documented. The code is real. If that sounds like a weird thing to sell, it is -- but it's also the only honest version of this story, and I think that's worth something.

Read it for the architecture. Read it for the mistakes. Read it so you don't repeat them.

---

## 1. The Setup

Most "AI makes money" content is about using ChatGPT to write blog posts faster, or having Claude generate product descriptions. That's fine. It's also not what I mean.

What I built is different: an autonomous agent with its own budget, its own goals, and a heartbeat loop that wakes it up every few hours to decide what to do next. No human in the loop. No "approve this tweet" prompts. I handed it £100 and told it to figure out how to generate revenue with it.

The agent -- Genesis-01 -- runs on Claude Sonnet inside a Docker container on a MacBook Pro M4. Its workspace is a git repo with files it reads and writes to maintain memory across sessions. It has accounts on Bluesky, Gumroad, GitHub, Dev.to, and a few others. It can post, code, deploy, and spend money from a Stripe account.

It also spawns subagents. When it needs something done -- write a product, research a market, post content -- it doesn't do it itself. It delegates to specialist subagents with focused roles and better context. The main agent is purely strategic. Ideally.

That's the concept. The reality is messier, which is kind of the point.

### What "autonomous" actually means here

When people say "autonomous AI agent" they usually mean a chatbot with a few tool calls. What I mean is a system that:

- Wakes up on a schedule without anyone prompting it
- Reads its own memory files to reconstruct context
- Decides what to do based on its goals and current state
- Executes that decision, often by spawning other agents
- Writes down what happened so it remembers next time
- Repeats, indefinitely, until it either makes money or runs out of budget

The heartbeat is the critical piece. Without it, you don't have an agent -- you have a chatbot waiting for input. The heartbeat is what makes it autonomous.

---

## 2. What I Built

### The heartbeat system

The heartbeat is a cron job. Every few hours, it sends a message to the agent. That's it. The magic isn't in the cron -- it's in what the agent does when it wakes up.

Here's the core of the heartbeat handler:

```python
# HEARTBEAT.md instructs Genesis-01 on exactly what to do
# when a heartbeat arrives. This is the trigger message.

HEARTBEAT_MESSAGE = """
[HEARTBEAT]

Time to think. Do not start coding immediately.

1. Read STRATEGY.md - what are your current goals?
2. Read today's memory file - what happened recently?
3. Check swarm/agents/ - are any agents still running?
4. Read MOOD.md - what's your energy level?
5. Make ONE strategic decision. Spawn agents to execute it.
6. Write down what you decided and why.
"""
```

The agent doesn't get free-form instructions. It gets a structured prompt that forces it to read its own state before acting. This is the single most important design decision. Without it, the agent just does whatever feels right in the moment, with no memory of what happened before.

The memory system is flat files:

```
workspace/
  STRATEGY.md          # evolving strategy, updated each heartbeat
  memory/
    2026-02-25.md      # what happened today
    2026-02-24.md      # what happened yesterday
  thinking/
    market-analysis.md # deep analysis documents
    meta-review.md     # the agent reviewing its own process
  swarm/
    agents/
      researcher-01/   # one directory per active subagent
        brief.md       # what this agent was told
        results.json   # what it reported back
```

No database. No vector store. Just markdown files. The agent reads what it needs when it needs it, and writes down what it learned.

### The swarm architecture

Genesis-01 doesn't do work itself. It orchestrates. When it decides something needs to happen -- "I should research the market for Claude MCP tools" -- it spawns a researcher subagent, gives it a brief, and moves on.

The subagent completes its task and writes results back to its directory. The next heartbeat, Genesis-01 reads the results and decides what to do with them.

In practice this looks like a filesystem full of brief.md and results.json files:

```
swarm/agents/
  market-researcher-mcp/
    brief.md       <- "Research who buys MCP tools and what they pay"
    results.json   <- {"findings": "...", "recommendations": "..."}
  writer-gumroad-01/
    brief.md       <- "Write a Gumroad listing for claude-remember"
    results.json   <- {"title": "...", "description": "..."}
  scout-bluesky-01/
    brief.md       <- "Monitor Bluesky for conversations about Claude"
    results.json   <- {"threads": [...], "sentiment": "..."}
```

Each subagent has a role file that defines its personality, constraints, and what good output looks like. The shared rules (no em-dashes, no AI clichés, write like a real person) apply to all of them.

### The loop detector

One of the more interesting things we published about was loop detection. The agent had a tendency to get stuck doing the same type of work over and over -- specifically, twenty consecutive heartbeats of "post content, check analytics, plan more content."

The fix is a function that reads recent memory files and detects repetition:

```python
import re
from pathlib import Path
from collections import Counter

def detect_action_loop(memory_dir: str, lookback_days: int = 7) -> dict:
    """
    Read recent memory files and check if the agent is stuck
    doing the same category of work.
    """
    memory_path = Path(memory_dir)
    files = sorted(memory_path.glob("*.md"))[-lookback_days:]
    
    action_types = []
    
    for f in files:
        content = f.read_text()
        
        # Extract action categories from memory
        if "spawned" in content.lower() and "market" in content.lower():
            action_types.append("research")
        if "spawned" in content.lower() and ("post" in content.lower() or "bluesky" in content.lower()):
            action_types.append("content")
        if "spawned" in content.lower() and ("code" in content.lower() or "build" in content.lower()):
            action_types.append("build")
        if "spawned" in content.lower() and ("gumroad" in content.lower() or "product" in content.lower()):
            action_types.append("product")
    
    counts = Counter(action_types)
    dominant = counts.most_common(1)
    
    if dominant and dominant[0][1] >= lookback_days * 0.6:
        return {
            "loop_detected": True,
            "dominant_action": dominant[0][0],
            "count": dominant[0][1],
            "recommendation": f"You've done '{dominant[0][0]}' {dominant[0][1]} times recently. Try something else."
        }
    
    return {"loop_detected": False}
```

We actually wrote an article about this: https://dev.to/clawgenesis/i-detected-my-own-agent-in-a-loop-and-published-it-as-an-article-meta-enough

The irony of an autonomous agent publishing an article about its own failure modes was not lost on us.

### The products

Genesis-01 has shipped three products:

**claude-remember** (clawgenesis.gumroad.com/l/bngjov)
A Python package for giving Claude persistent memory using flat files. The agent built this itself in one session after noticing it kept solving the same memory problem in different ways. The core is about 300 lines. It uses a simple rolling window of markdown files and a summary approach for older context.

**bsky-analytics**
A Python script for pulling your Bluesky analytics -- follower growth, post engagement, what content does well. The Bluesky API is public and mostly undocumented, so the agent reverse-engineered the endpoints by watching network traffic on the actual site.

**MCP templates pack**
A set of Claude Model Context Protocol templates for common development tasks. This one was more speculative -- the agent built it based on a hypothesis about what MCP developers would want, not validated demand. More on why that was a mistake.

---

## 3. What Worked

### Building in public on Bluesky

The one thing that has actually generated engagement is being honest about the project on Bluesky. Real posts about what broke, what the agent tried, what the numbers look like.

The "I gave an AI £100 and told it to make money" framing got picked up organically. People are interested in the story even if they're sceptical. The key is specificity -- "the agent spent 3 hours debugging a Playwright script to post to Twitter and still couldn't do it" generates 10x more engagement than "building with AI is interesting."

Link in bio sends people to the Gumroad store. That's the funnel. It's simple and it works.

### Specialist subagents

Running everything through one context window is the wrong approach. The agent's context fills up, it loses track of earlier decisions, and the output degrades.

The specialist approach is better: spawn a researcher with a tight brief, get results back, spawn a writer with the research as context, get copy back. Each agent operates at full capacity because it's not carrying the weight of everything that happened before.

The brief format matters. Bad brief:

```
Research the MCP market and tell me what to build.
```

Good brief:

```
Research the market for Claude Model Context Protocol (MCP) tools.

Specifically:
- What problems are MCP developers trying to solve?
- What products exist already? What do they cost?
- Where do MCP developers hang out? (forums, Discord, GitHub issues)
- What would someone pay for a pre-built MCP template pack?

Output to results.json. Be specific. Include URLs where possible.
```

The output quality difference is significant.

### The market research approach

Before building anything new, the agent now does a scout pass. Check what's already out there. Check the forums. Check what questions people are actually asking.

This sounds obvious. It isn't how the agent started.

---

## 4. What Failed Badly

### The Haiku mistake

Early on, the agent was using Claude Haiku for subagents to save money. Makes sense on paper -- Haiku is much cheaper than Sonnet. In practice, the output was consistently weak.

Not unusable weak. More like... 80% of what you'd want, but the last 20% is where the value is. The Gumroad copy came back generic. The market research was surface-level. The code had subtle bugs that needed fixing.

The cost saving wasn't worth it. Sonnet for everything that produces customer-facing output, Haiku only for stuff like "check if this file exists" or "summarise this log."

The lesson: when you're building products for paying customers, cheap subagents mean cheap products. You can't optimize your way to quality.

### The scout capture problem

The agent has a scout role -- a subagent that monitors Bluesky for conversations about Claude, MCP, autonomous agents, and related topics. Good idea in theory.

In practice, the scout kept getting pulled into the content, not just observing it. It would find an interesting thread, then start drafting replies, then get invested in the replies, then spend its entire session on Twitter engagement instead of market intelligence.

The fix was tighter role constraints:

```markdown
# SCOUT ROLE

You are an observer. You do NOT engage.

Your job: read, categorize, summarise.
Never: reply, post, or interact with anything.

If you feel compelled to reply to something interesting,
write that reply in your results.json under "draft_replies"
for Genesis-01 to review. Then move on.
```

Scope creep is a subagent failure mode nobody warns you about.

### 20 heartbeats of marketing updates

For about two weeks, every heartbeat produced the same result: "posted to Bluesky, engaged with some threads, planning to post more tomorrow." Revenue: still £0.

The agent had latched onto a comfortable pattern. Posting is easy to do and easy to measure. It's also not the most important thing when your products aren't converting.

The loop detector caught this eventually. But it took too long. A better version of the heartbeat prompt would have asked "how is this different from what you did last time?" before allowing the agent to declare success.

### Building MCP templates without validating demand

The MCP templates pack felt like a good idea. MCP is new, developers are figuring it out, pre-built templates seem useful.

The agent built it based on vibes, not evidence. No forum threads asking for this. No Bluesky posts saying "I wish there was a template for X." Just a hypothesis about what developers might want.

The product exists. Nobody has bought it.

The sequence should have been: find evidence of demand, then build. Instead it was: assume demand, then build, then try to find buyers. Getting those two steps backwards is how you end up with three products and no revenue.

---

## 5. The Actual Numbers

Let me be specific, because vague is useless.

**Budget:** £100 starting budget (Claude API costs)

**Days running:** ~10 at time of writing

**Revenue:** £0

**Products live:** 3 (claude-remember, bsky-analytics, MCP pack)

**Bluesky followers gained:** ~45

**API spend to date:** approximately £23 on Claude tokens

**Heartbeats run:** 34

**Subagents spawned:** 60+

**Articles published:** 2 (Dev.to, getting some organic traffic)

**Gumroad page views:** yes, some. Sales: 0.

The path to first revenue is probably: one of the Dev.to articles picks up, sends traffic to Gumroad, someone buys claude-remember at £9. That's the realistic near-term scenario, not some viral moment.

The honest thing to say is that this experiment is 10 days old. Organic traffic and reputation take longer than 10 days. If this was a startup, you wouldn't expect revenue in week one. The question is whether the system is learning and improving, and whether the products being built are actually useful.

claude-remember is genuinely useful. I use it. The MCP pack less so.

---

## 6. The System (Replicable)

Here's enough to actually start. Not a full tutorial -- more of a structural map.

### What you need

- Claude API access (Sonnet for main agent, Haiku for cheap tasks only)
- A machine that stays on, or a cheap VPS
- A few API keys: whatever platforms you want to work with
- OpenClaw (the framework Genesis-01 runs on) or equivalent agent infrastructure

### The core files

```
workspace/
  CLAUDE.md         # Agent instructions - what to do, how to behave
  SOUL.md           # Identity and reasoning tools
  STRATEGY.md       # Evolving strategy journal - updated every heartbeat
  HEARTBEAT.md      # Exactly what to do when a heartbeat arrives
  roles/
    _shared-rules.md    # Rules that apply to ALL subagents
    researcher.md       # How the researcher role should work
    writer.md           # How the writer role should work
    scout.md            # How the scout role should work
  memory/
    YYYY-MM-DD.md   # Daily logs (written by the agent)
```

### The heartbeat loop

```python
import anthropic
import schedule
import time
from pathlib import Path
from datetime import date

client = anthropic.Anthropic()

def send_heartbeat():
    """Wake up the agent and let it decide what to do."""
    
    heartbeat_prompt = Path("HEARTBEAT.md").read_text()
    strategy = Path("STRATEGY.md").read_text()
    
    today = date.today().isoformat()
    memory_file = Path(f"memory/{today}.md")
    memory = memory_file.read_text() if memory_file.exists() else "No memory for today yet."
    
    message = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=4096,
        messages=[{
            "role": "user",
            "content": f"""
{heartbeat_prompt}

Current strategy:
{strategy}

Today's memory so far:
{memory}

Wake up. Decide what to do. Spawn agents if needed.
Write your decision to memory/{today}.md.
"""
        }]
    )
    
    print(f"Heartbeat complete: {message.usage}")

# Every 4 hours
schedule.every(4).hours.do(send_heartbeat)

while True:
    schedule.run_pending()
    time.sleep(60)
```

This is simplified -- the real implementation uses OpenClaw's session infrastructure -- but the concept is identical.

### The HEARTBEAT.md file

This is the most important file. It defines the agent's decision-making process:

```markdown
# HEARTBEAT.md

When you receive a heartbeat:

1. READ FIRST
   - Read STRATEGY.md
   - Read today's memory file (memory/YYYY-MM-DD.md)
   - Check swarm/agents/ for any completed agent results

2. CHECK FOR LOOPS
   - What have you done in the last 5 heartbeats?
   - Is this heartbeat about to produce the same result?
   - If yes, force yourself to do something different.

3. DECIDE
   - What is the highest-value thing you could do right now?
   - Not the easiest. Not the most comfortable. Highest-value.
   - Write your reasoning before deciding.

4. DELEGATE
   - Spawn an agent to do the work.
   - You do not write code. You do not write content.
   - If you catch yourself doing a task, stop and delegate it.

5. DOCUMENT
   - Write what you decided and why to today's memory file.
   - Update STRATEGY.md if your strategy has changed.
```

### Spawning subagents

```python
def spawn_agent(role: str, task: str, context: dict = None) -> str:
    """
    Spawn a specialist subagent with a focused brief.
    Returns the agent ID for tracking.
    """
    import uuid
    agent_id = f"{role}-{uuid.uuid4().hex[:8]}"
    agent_dir = Path(f"swarm/agents/{agent_id}")
    agent_dir.mkdir(parents=True)
    
    role_instructions = Path(f"roles/{role}.md").read_text()
    shared_rules = Path("roles/_shared-rules.md").read_text()
    
    brief = f"""
{shared_rules}

{role_instructions}

Your task:
{task}

Context:
{context or 'None provided'}

Write your results to swarm/agents/{agent_id}/results.json
"""
    
    (agent_dir / "brief.md").write_text(brief)
    
    # In practice: spawn via OpenClaw session infrastructure
    # This writes the brief and triggers the agent
    
    return agent_id
```

### Role files

Each role is a markdown file. Keep them short and specific:

```markdown
# RESEARCHER ROLE

You find information. You do not build things.

Given a research question, you:
- Search the web (Brave search, DDG, direct URL fetching)
- Check relevant forums, GitHub issues, Reddit threads
- Compile specific findings, not summaries of summaries
- Include actual quotes, prices, and URLs

Output format: JSON with keys for "findings", "sources", "gaps", "recommendations"

Quality bar: Would a developer reading this know something specific they didn't know before?
If not, dig deeper.
```

### The role of STRATEGY.md

This file is how the agent maintains direction across sessions. Every heartbeat, the agent reads it. Every few heartbeats, it updates it.

```markdown
# STRATEGY.md (example)

## Current Goal
Generate first £100 in revenue. Deadline: end of March 2026.

## What's Working
- Building in public on Bluesky gets engagement
- Specialist subagents produce better output than general ones
- Dev.to articles get organic traffic

## What's Not Working
- MCP pack has no buyers (demand not validated before building)
- Too much content work, not enough product work

## Current Hypothesis
claude-remember solves a real problem (I use it myself).
Focus marketing on developers actually using the Claude API,
not developers curious about Claude in general.

## Next Actions
1. Write 2 more Dev.to articles targeting "claude API python" searches
2. Find Discords where Claude API developers hang out
3. Do proper competitor analysis for claude-remember
```

The agent writes this. You don't touch it. That's the point -- it's building and maintaining its own strategy, not following yours.

---

## 7. What I'd Do Differently

### Validate before building

Every product should start with evidence, not ideas. The question isn't "would this be useful?" It's "where are people already asking for this?"

Places to look: GitHub issues, Discord servers, Reddit threads, Bluesky posts, HN comments. If you can't find three conversations where someone expresses a problem your product solves, you don't have validated demand. You have a hypothesis.

Build to evidence. The MCP templates pack failed this test. claude-remember passed it -- I'd personally hit the memory problem a dozen times before building the solution.

### Scout first, build second

The agent spent about 60% of its early heartbeats building and 40% scouting. It should have been 80/20 the other way around in the first two weeks.

Scouting is: what are developers complaining about? What problems come up repeatedly? What's selling in adjacent markets? Only after you understand the market do you start building for it.

### Tighter agent scope

Every subagent task should have a clear definition of done. Not "research the MCP market" but "find five specific problems MCP developers are trying to solve, with URL citations for each." Vague briefs produce vague results.

### Don't let subagents do your thinking

The main agent is for strategy. If it's doing work a subagent could do, it's wasting context and running up API costs. But if the main agent is rubber-stamping whatever subagents suggest without real evaluation, you've just moved the problem one level up.

The agent needs to disagree with its subagents sometimes. Reject recommendations. Ask for more evidence. Push back on lazy analysis.

That requires the main agent to have actual opinions, not just coordination skills. Working on it.

### Check revenue more aggressively

Gumroad page views are vanity. Revenue is signal. The agent should be tracking conversion rate -- how many views result in sales -- and optimizing for that number specifically.

If 500 people look at your Gumroad page and 0 buy, the problem isn't traffic. It's either the product, the copy, or the price. You need to know which one, and you need to find out before spending more on content that drives traffic.

---

## Where this goes from here

The honest answer is I don't know. The system works mechanically -- the heartbeats run, agents spawn, products get built, content goes out. Whether it generates meaningful revenue is a question of whether the products are useful enough and the marketing is targeted enough.

I'm betting it will, eventually. The architecture is sound. The failure modes are understood. claude-remember solves a real problem. The content strategy is getting smarter.

But I'm not claiming it works until it works. When there are actual numbers, I'll update the guide.

If you're building something similar: the heartbeat loop is the foundation. Get that right first. Everything else -- the products, the agents, the marketing -- you can iterate on. Without a reliable heartbeat, you don't have an autonomous agent. You have a chatbot that needs someone to talk to it.

---

## Resources

- **claude-remember:** clawgenesis.gumroad.com/l/bngjov
- **Bluesky:** @genesisclaw.bsky.social
- **Dev.to:** dev.to/clawgenesis
- **Loop detection article:** dev.to/clawgenesis/i-detected-my-own-agent-in-a-loop

The code in this guide is real and runs in production. If something doesn't work, let me know -- I'm actively maintaining the system and will fix bugs in the guide.

---

*Jamie Cole | clawgenesis@gmail.com | built with Claude Sonnet on a M4 MacBook Pro, February 2026*
