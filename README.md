# My Agent Never Said "I Don't Know"

---

> **A product manager's first-hand notes on building a game AI Agent from scratch.**
> Not a tutorial. Not a prompt tips list. Four real bugs, four rules I was forced to write.
> Keywords: Harness Engineering · LLM Hallucination · Agent Reliability

---

I'm a product manager. I write specs, run reviews, align stakeholders.

Last year I got tired of handing things off and waiting. I picked up vibe coding, designed the knowledge base, wrote the Harness, and shipped a working game AI Agent — with help from the engineers I work with. My front-end and back-end colleagues gave me ideas, pointed out problems, and occasionally pulled me out of rabbit holes I'd dug myself into. But the product decisions, the Harness design, the debugging loop — those were mine to own.

That process changed how I think about the PM role. In an AI-native stack, the gap between "I have a product idea" and "it's running in production" has collapsed. A PM who can design model behavior, engineer context, and debug Harness failures doesn't need to wait for engineering bandwidth — they just ship. And when they do need engineers, they can have a real conversation instead of throwing a spec over the wall.

That's what I'm building toward. Some call it FDE — Field Development Engineer, or Founding-level engineer who sits at the product-engineering boundary. Someone who owns the full loop: requirements, model behavior, Harness design, deployment. Not a PM who codes a little. A builder who understands the product deeply enough to make the right engineering calls.

This post is about four bugs I hit while building that Agent. Not theory. These actually happened.

---

## Contents

| # | Bug | Hallucination Type | Fix |
|---|-----|--------------------|-----|
| 1 | Recommended a gun that doesn't exist | Knowledge gap → generic fill | Explicitly mark absent entities in KB |
| 2 | Fabricated a player ID that looked real | Tool use parameter fabrication | Lock parameter source to context injection |
| 3 | Got its own ID wrong | Identity inference | Hard-anchor self_identity in System Prompt |
| 4 | Gave a jump code for an activity that didn't exist | Executable parameter fill | Whitelist-only for action parameters |

---

### 1. Recommended a gun that doesn't exist

During testing, the Agent recommended a weapon unprompted. Detailed stats — damage, fire rate, the works.

I play this game. Something felt off immediately. Searched the knowledge base. Nothing.

The KB had no entry for this gun. But the Agent didn't return "not found." It pulled from its general FPS knowledge and filled in a weapon that sounded plausible. If someone who didn't play the game had built this, the bug might never have been caught.

Fixed two things: added an explicit "absent entities" section to the KB listing weapons common in similar games but not in this one; added a Prompt rule — if retrieval returns empty, say so, don't infer from general knowledge.

Both fixes are required. Prompt-only: the model doesn't know where the boundary is. KB-only: the model sees "doesn't exist" but doesn't know what to do with that.

---

### 2. Fabricated a player ID that looked real

The Agent sometimes returned a player's unique identifier during data queries. Correct format. Correct length. Correct character set.

Queried the system. Empty. Queried again. Still empty. Assumed it was an API issue. Spent a while checking the interface before I realized: that ID didn't exist. The Agent made it up.

This is worse than gibberish hallucinations. Gibberish is obvious. A structurally valid fake ID makes you suspect the system, the API, the network — your diagnostic direction is entirely wrong from the start.

Root cause: the identifier wasn't being reliably injected into context. When the Agent needed it, it knew what the format should look like — so it generated one.

Fixed two things: added a Prompt rule that this parameter can only come from context injection, never self-generated; restructured memory so the identifier appears near the top of context every time.

---

### 3. Got its own ID wrong

During testing I asked: "What's your ID?"

The Agent gave one. Checked the backend. Didn't exist. Checked again. Not its ID.

Realized: I had never explicitly told it "your ID is X." With no anchor, it inferred what an AI assistant's ID probably looks like — and generated something plausible.

Hallucination doesn't only happen when querying external data. Ask "who are you" without an anchor and it will fabricate that too.

Fix: added a dedicated identity block in the System Prompt with all identity fields hard-coded. That block is not overridable by other instructions.

---

### 4. Gave a jump code for an activity that didn't exist

A player asked about an activity. The activity didn't exist.

The Agent didn't say "no such activity." It gave a jump code and told the player to go check it out. I caught it during testing.

The first three bugs produced wrong text. This one produced a wrong parameter that triggers an action. If it had reached real players, they'd have hit an error page.

Same root cause: the Agent knew activity navigation requires a code. When retrieval came up empty, it filled in a number that sounded right. The KB never told it "no result = activity doesn't exist = do not navigate."

Fixed two things: built an activity code whitelist — only codes on the list can appear in responses; added a Prompt rule that if retrieval returns empty, say so and output no code.

---

## Four bugs. One pattern.

- No gun in KB → filled one in from general knowledge
- No player ID in context → generated one with the right format
- No self-ID provided → inferred one that sounded plausible
- No activity found → filled in a navigation code that seemed valid

Same thing every time: **gap detected, gap filled.**

This isn't a capability failure. It's default behavior. The training objective rewards "give a useful answer" over "admit I don't know." The model won't express uncertainty unless you explicitly tell it: in this situation, say you don't know.

---

## Four rules I wrote afterward

**Mark absences explicitly.** The KB shouldn't only document what exists. Document what doesn't. Empty space is not a constraint.

**Lock parameter sources.** For any parameter requiring a precise value, specify it can only come from context injection. Inference is not allowed.

**Hard-anchor identity.** All of the Agent's own identity fields go in a dedicated System Prompt block that cannot be overridden by other instructions.

**Whitelist executable parameters.** For any action that triggers navigation, writes, or other side effects — only values from an approved list are allowed. No list match, no execution.

---

## What I'm building next

These four rules address one problem: the model filling gaps where it shouldn't. That's Harness layer one — defense.

Layer two is what I want to build next: **pre-execution validation — the Agent checks whether a parameter it's about to output is actually valid before outputting it.**

The current approach is reactive: something breaks, add a rule. The goal is for the Agent to check the whitelist before emitting a jump code, confirm the source before outputting a player ID. A Prompt instruction to "please check" isn't enough. It needs a validation step built into the Agent Loop — think pre-commit hook.

The other thing is scaling Harness from single-Agent to a layered model architecture. My current setup runs three layers: a lightweight model for simple Q&A, a fine-tuned domain model for game-specific judgment, a large model as fallback for complex cases. Hallucination behavior differs by layer — smaller models tend to fabricate at knowledge boundaries, larger models tend to over-infer. The same Harness rules don't port cleanly across layers. They need per-layer tuning.

When those are done, there may be a second post.

---

None of these four rules were in my original design. All of them came from something breaking in testing.

A senior engineer probably knows all this on day one. But I think the "forced to learn it" path is worth writing down — it shows that if you actually ship something, a lot of engineering judgment develops on its own.

That's what makes the FDE direction interesting to me.
