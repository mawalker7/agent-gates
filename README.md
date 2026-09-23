# Agents are cheap to run and expensive to trust


I run most of my operating work through AI agents now: compliance, finance, go-to-market analytics, the status board that tells me what needs my attention each morning. The system I help operate at one company merged more than a thousand agent-produced pull requests this year. I've spent enough time watching agents be confidently wrong to have an opinion about what actually makes this work, and it isn't the agents.

It's what stands between the agent and production.

## The premise

An agent will do a large amount of plausible work, quickly, for almost nothing. That's both the appeal and the problem: plausible is not the same as correct, and an agent's account of what it did is not evidence that it did the thing, or that it did it correctly. Every failure I describe below came with a success message attached.

The interesting engineering question isn't how to prompt an agent into being right; it's how to build a loop where being wrong is cheap, visible and contained. My loop is designed with five core parts, and I've run some version of this loop on everything from a company codebase to a twice-daily personal status board.

## The loop

*One: bounded input.* A deterministic, non-model step gathers what the agent is allowed to see: size-capped, with a manifest that records, per input, whether it was read, blocked, failed or never configured. The agent reads that bundle. It doesn't explore the machine, and silent input loss becomes a line in a manifest instead of a mystery.

*Two: a narrow contract.* The agent's instructions say what it may do, what it may never do (originate work, message anyone, cite a source it didn't see), and what to do with anything it couldn't check: declare it. Never drop it quietly.

*Three: machine checks.* The output is parsed against a fixed schema before it can be used. Tests are written first and treated as the physical truth. Anything crossing a size or risk threshold gets an independent review pass that traces the change against the actual code rather than the agent's description of it. And the check has to be a refusal, not a warning. A gate that prints a caution and proceeds is not a gate.

*Four: a human decides, at fixed points.* Merges; releases; spend; anything outward-facing; any flip from observing to enforcing. The list is short and specific enough to answer quickly, and it's the same list every time. On my own status board, the only write path back into the system is me marking an item handled on the page. An agent cannot close its own loop.

*Five: failure containment.* When the agent is wrong, the run fails *loudly*. Read positions roll back so nothing is consumed unread; timeouts and a bounded retry budget cap the damage; the failure lands in a log and a notification rather than being absorbed. Then the fix addresses the class of error, not just the instance, and gets written down where the next session will find it.

## Why gates beat prompts

I used to believe better instructions were the answer. A great prompt helps, but instructions are the weakest layer, because a prompt is a request and the agent's compliance with it is unverifiable from the inside. Every part of the loop above works because it doesn't depend on the agent's cooperation: the collector is deterministic; the schema check is code; the review pass reads the diff, not the summary; the human gate is a page the agent can't write to.

The principle underneath all of it: the primary source beats the summary. When an agent's account of a file, a script or a system conflicts with the artifact, the artifact wins, and the account is a hypothesis until someone opens the thing.

## Where the humans go

Not everywhere. A gate that asks a person to approve every step becomes a rubber stamp within a week: a lazy habit mistaken for an actual gate. I put people where the blast radius is: anything hard to reverse, anything that reaches outside the workspace, anything that changes what the gates themselves enforce. Everything else either proves itself mechanically or refuses.

The one rule I hold tightest: invoking any skip or bypass needs approval *before* the fact, and a documented exception gets registered rather than silently waived. The gate mechanism is auditable like everything else. Two of the failures below are the gate being wrong.

## What the gates have actually caught

These are real, dated, and reduced to mechanism. Names are omitted on purpose.

*A harness that reported a verdict it couldn't read.* A test harness ran a batch of deliberate code mutations and reported that every one of them survived, which would mean the test suite caught nothing. The real cause was the harness itself: it was reading its runner's results from the wrong output stream, matched nothing, and treated "nothing found" as zero failures. It was caught because an identical result across unrelated mutations was treated as a deviation to explain before being believed. The rule that came out of it: a harness that can't parse its runner's output must report *unreadable*, never a pass.

*A scheduled job that succeeded for nine weeks while delivering nothing.* Its delivery path was blocked by an egress policy that returned a refusal the job never surfaced, so every run logged success. A person reading the actual log, instead of trusting the status, found it. When a job reports success and the side effect is missing, read the log before theorizing.

*An agent that rebuilt my status board from scratch and dropped half of it.* The context bundle had grown past what a single read returns, so the tail, including the previous board, became invisible. The agent did what it could see, which was start over. Nothing in its output looked wrong. It was caught by a number: item carry-forward between consecutive boards, normally 84% on average, fell to 4% and then to 0%. That metric exists because I'd decided early that the board's continuity was the thing to measure. Without it, I'd have noticed when something I'd been tracking simply stopped existing.

*A publishing step that reported done and wasn't.* The agent added an undocumented conversion flag on its own initiative and produced a file the page couldn't render. The fix wasn't a better prompt. Publication is now confirmed by looking for a concrete identifier on the published artifact, not by the agent saying it published.

*A draft policy that matched the story and not the code.* A public-facing draft made a data-handling claim that was plausible, consistent with how everyone described the product, and wrong: reading the code showed two flows where it didn't hold. Caught by a verify-against-code step that runs before anything is published. For compliance work this is the failure that matters most, because the reviewer who'd have caught it later is an auditor.

*A model asked to analyze nothing.* I keep a deliberately junk fixture, a captured error page, in one research tool specifically to test whether the analyst invents analysis when there's no post to analyze. It refused, and left every analytical section as not applicable. That's a negative control: it tests the failure direction, not the success direction, and it's cheap. The same tool had earlier been saving notes cut off at the token limit as though they were complete, which is the quieter version of the same problem; now truncation is treated as no result at all.

*Three sessions that agreed on a wrong premise.* Three separate agent sessions proposed the same design for carrying memory between two machines, on the shared assumption that both machines already committed and pushed a particular repository. All three had read the documentation. None had read the scripts, which staged exactly one fixed path. The design would have broken a working system. It was caught by reading the two scripts, and the lesson generalized: prose in your own repository is a claim, not evidence.

## Applying it to compliance

This is the domain where I've found the loop pays off most, because compliance work is mostly assertion, which is exactly what agents are good at producing and bad at grounding.

The pattern is the same. Evidence collection: an agent drafts the summary of what a control's evidence shows; a person confirms it against the artifact before it's attached to anything. Questionnaire responses: drafted from an approved answer bank, never from the model's general knowledge, and every answer traces to a source document before it goes to a customer. Policy currency: an agent flags drift between policy text and the systems it describes; the flag is a ticket, not an edit. Control mapping: agent proposes, reviewer disposes, and "not applicable" carries a named sign-off rather than being a silent escape.

The rule that governs all of it is the one from the privacy-draft failure. Every figure and every claim traces to an owning engineering document, never to the marketing copy, never to the last version of the policy, and never to the agent's recollection. The site is downstream of the engineering record, not a source for it.

## What it costs

Gates aren't free.

They add latency. A twice-daily board run is a chain of capped phases with a bounded retry, and a hard failure notifies me rather than quietly trying again. They add maintenance: a small personal tool of a few thousand lines carries fifty-odd tests, and every gate is code that can itself be wrong. Two of the failures above were the gate failing: a harness that couldn't read its runner, and a drift check that compared a template against the file generated from it, a comparison that could never disagree. When that check was rewritten to compare against the real example files, it immediately found dozens of missing keys in two templates that had been reported clean.

They also produce false alarms. A partly completed fast-forward once looked exactly like another writer's uncommitted changes on the same files, and the protocol escalated. Cheap checks (look for active sessions; byte-compare a few flagged files) showed it was my own partial checkout. The escalation now reserves itself for content that genuinely diverges.

And the pattern doesn't prevent agent error. Agents were wrong repeatedly, including in ways that would have broken working systems. What the gates did was catch the errors *before they landed*, which I find to be a more durable and honest claim. On my own board: thirty-one scheduled runs over eleven days, twenty-three clean, five failed loudly with read positions rolled back and nothing consumed unread. I'll take that ratio over a system that has never been seen to fail, because I know what that one is hiding.

## The short version

Agents are cheap to run and expensive to trust. Spend the engineering on the trust: bound what they see, narrow what they may do, check the artifact and not the story, put people where the blast radius is, and make failure loud. Then measure it, because the number is what tells you when the agent has quietly started over.

*Matt Walker runs operations, compliance and go-to-market for a small portfolio of companies through agentic workflows he designs and operates. The engine behind the status board described here is public under an MIT licence.*
