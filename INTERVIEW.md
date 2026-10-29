# Presenting this design in a system design interview

This is the companion to `DESIGN.md`. It covers how a 45 to 60 minute session usually goes, what the interviewer is scoring at each stage, the numbers to have memorized, the follow-up questions you will get and how to answer them, and the mistakes that cost candidates points on this exact problem.

---

## The 90-second pitch

Say this, or your version of it, right after you have clarified requirements. It tells the interviewer you see the whole system before you draw a box.

> "We need to place 100,000 calls in four hours, which is 8 calls a second sustained if I keep a 30-minute tail. With about 30% live answers and a two-and-a-half-minute conversation, Little's law puts us at roughly 540 concurrent calls and 320 concurrent AI conversations. So the constraints are carrier calls-per-second limits, concurrent AI session capacity, and the legal calling window, not the database, which sees eight writes a second.
>
> The design has a control plane that plans the campaign into four-hour blocks in each callee's local time and paces dials with a token bucket bounded by those three limits; a dial plane of stateless dialer workers and stateful media servers that hold the calls; a voice agent per call streaming audio through STT, a small fast LLM, and TTS with a one-second turn budget; and two carriers and two of every AI vendor behind thin gateways because each of them will fail. The one invariant I protect above everything is that no contact is dialed twice in a block, and I get that from a unique index on (contact, block) written before we originate. Failed calls are classified and retried in the next block under a policy with caps. Inbound calls to our caller IDs are recognized by number and routed to the same agent. Let me draw it."

---

## How the session goes

| Minutes | Phase | What the interviewer is scoring | What to do |
|---|---|---|---|
| 0 to 5 | Clarify requirements | Do you ask the questions that change the design? | Ask the eight questions below. State your assumptions aloud and write them on the board |
| 5 to 10 | Capacity estimation | Can you turn requirements into numbers, and do you know which numbers matter? | Dial rate, hold time mix, Little's law for concurrency, AI sessions, inbound. Say "the DB is not the bottleneck" |
| 10 to 20 | High-level design | Do the boxes have clear responsibilities, and do the arrows carry data you can name? | Draw control plane, dial plane, state, AI gateways, two carriers. Name the Kafka topic and the Redis keys |
| 20 to 40 | Deep dives | Depth where it matters; honesty about trade-offs | Let the interviewer choose, but steer toward: pacing, exactly-once dialing, the agent latency budget, retry classification |
| 40 to 50 | Failures, monitoring, cost | Have you operated something? | Walk three failure rows and say how each is detected and what happens to the contact. Give the per-campaign cost range and the two levers |
| 50 to 60 | Scale and wrap | Can you see the next order of magnitude? | 10× and 100×: what breaks first (AI vendor quotas, caller ID pool, legal window), what does not (Postgres) |

### The eight clarifying questions

Ask these. Each one changes the design, and interviewers notice which ones you skip.

1. "Who triggers a run, and is it on demand, scheduled, or both?" (API shape, planner trigger)
2. "What counts as a failed call for retry purposes? No answer, voicemail, busy, carrier error, or only hard failures?" (the classification table)
3. "Is there a cap on attempts per contact, and are there legal frequency limits I should design for?" (policy caps)
4. "What is the geography? Single time zone or national?" (local-time blocks)
5. "Is this marketing to consumers on mobile? Then I'll assume prior consent is the client's responsibility and design for DNC scrubbing, 8am to 9pm local, and immediate opt-out." (shows you know TCPA exists)
6. "What is the conversation goal and typical length? That sets concurrency and cost." (hold time)
7. "Why do we need inbound: callbacks to the caller ID, or a separate inbound campaign?" (routing and capacity)
8. "Single tenant for the capacity math, or many clients at once?" (fair-share pacing)

---

## Numbers to have cold

| | |
|---|---|
| Dial rate | 8 calls/s (100,000 in 3.5 h; 30 min tail) |
| Mean channel hold | 67 s (30% live at 150 s, 25% voicemail at 45 s, 35% no answer at 30 s, 10% failed at 5 s) |
| Concurrent channels | 540, provision 800 |
| Concurrent AI sessions | 320, provision 500 |
| Agent turn latency target | p50 1.0 s, p95 1.8 s |
| Latency budget | VAD 300 to 500 ms, STT final 100 to 300, LLM first token 300 to 600, TTS first byte 100 to 300, playout 100 to 200 |
| PSTN audio | 8 kHz G.711 mu-law, 20 ms frames, 64 kbps |
| Attempts over 3 blocks | ~212,000 for 100,000 contacts; ~50,000 live conversations |
| Cost per campaign | $2.7k to $14k; telephony and TTS dominate, LLM under 5% |
| Cost per live conversation | $0.05 to $0.28 |
| Legal window | 8am to 9pm callee local; three 4-hour blocks fit in a day |
| Inbound | ~4% callback rate, ~35 concurrent at peak, provision 100 |

---

## What interviewers are actually scoring

**Requirements judgment.** Did you find the constraints that are not in the prompt: time zones, legal hours, consent, caller ID reputation, carrier CPS caps? A candidate who designs a beautiful Kafka pipeline and calls Pacific numbers at 7am has failed the interview.

**Estimation.** Not the arithmetic, which is easy, but choosing what to estimate. Concurrency via Little's law is the signature move for this problem. Say "Little's law" out loud.

**Finding the real bottleneck.** 8 writes a second is nothing. Candidates who shard the database here reveal they have not thought about where the load is. The load is in telephony and AI session capacity, and the scarce resource is legal calling time.

**Correctness invariants.** Exactly-once dialing per block. Explain the mechanism (unique index, write before originate, webhook carries our id, reaper for the gap). Interviewers push on "what if the worker dies right here" at every step; have the answer for each gap.

**State machines.** Campaign states, attempt states, contact states. Draw at least one. Name the illegal transitions.

**Streaming and latency.** For the AI pipeline, the interviewer wants to hear that everything streams, that you split LLM output on sentences for TTS, that you handle barge-in and truncate history, and that the opening line is pre-rendered. A candidate who describes request-response STT then LLM then TTS has designed a 4-second pause.

**Failure handling with ownership.** Not a list of things that can fail, but for each: how it is detected, what the system does, and what happens to the contact. The `failed_internal` class that does not consume an attempt is a detail that signals real experience.

**Observability as design, not afterthought.** One event per attempt with everything on it; attempt id as the trace id; SLIs tied to the business (early hang-up rate as a reputation signal); alerts that page on compliance violations.

**Cost awareness.** Which line items dominate and what the levers are. Pre-rendering fixed speech is the answer that is both a cost lever and a latency lever; mention it in both contexts.

**Trade-offs stated as decisions.** "I chose X over Y because Z, and I would revisit it when W." Section 19 of the design is a list of these. Use the form.

---

## Where the interviewer will push, and what to say

**"Walk me through what happens when a dialer worker crashes."**
Three gaps. Between claiming the job and inserting the attempt: lease expires, another worker claims, inserts, dials; nothing was done. Between inserting and originating: a row in `dialing` with no provider id; the reaper marks it `failed_internal` after 2 minutes and it rolls to the next block without consuming an attempt. Between originating and recording the provider id: the carrier's first webhook carries our attempt id as the client reference, and the handler fills in the gap. In none of these is the contact dialed twice, because the second insert would hit the unique index.

**"How do you guarantee the block finishes on time?"**
You do not guarantee it; you control for it and you report it. The pacer computes needed CPS from remaining contacts and remaining seconds every second, dials 15% ahead of that, and is bounded by carrier CPS, channel capacity, and AI session capacity. If a cap prevents catching up, the status document shows `on_track: false` within five minutes and the remainder rolls to the next block as `not_attempted`. Then say: "the honest product promise is a live ETA with confidence, not a hard deadline."

**"Why not just retry failed calls a few minutes later?"**
Calling someone back twenty minutes after they did not answer is hostile and rarely changes the outcome; a different time of day does. The block is the retry spacing, and it gives the client a plain statement about when everyone has been tried. Transient carrier failures are the exception and are retried in seconds on the other carrier.

**"What happens if the LLM provider goes down mid-campaign?"**
Gateway sees error rate and latency, fails over to the secondary model or provider. If both are down, the AI circuit breaker opens: the pacer stops releasing dials, in-progress calls play a pre-rendered "we'll call you back" and end as `failed_internal`, and those contacts retry next block without consuming an attempt. The thing to avoid is connecting a human to silence.

**"How do you handle 'stop calling me'?"**
Two paths, both before hangup. The LLM has an `opt_out` tool; a keyword matcher on the STT stream and DTMF 9 are the second path for when the model misses it. The suppression write is synchronous and precedes the polite close. An hourly audit checks every `opted_out` disposition has a suppression row. Mention the suppression list is tenant-wide, not campaign-scoped.

**"Why a unique index instead of a distributed lock?"**
A Redis lock loses on failover and has a TTL race; a unique index is a millisecond slower and cannot be wrong. At 8 inserts a second there is no reason to accept the weaker guarantee. Redis is the hint queue; Postgres is the truth.

**"How do you know a human answered and not voicemail?"**
Answering machine detection, 2 to 4 seconds after answer, 5 to 10% wrong. The errors are asymmetric: hanging up on a human is worse than talking to a machine for 30 seconds, so bias toward human and let the agent detect a beep or a long monologue and switch modes. Voicemail message is pre-rendered once per campaign.

**"What about time zones?"**
Blocks are local-time windows per cohort. A national list means the system dials for seven wall-clock hours per block. Time zone from address, else area code with an ambiguity flag; ambiguous numbers go into the window where both candidate zones are legal. The dialer re-checks local time before every originate, because the planner can be wrong and the fine is per call.

**"Design the status endpoint."**
Show the status document from section 5. Say it is served from Redis counters fed by Kafka consumers, never from the attempt table on the request path. Progress, ETA with confidence, outcomes, dispositions, retry queue, cost so far, health.

**"What's your latency budget and how do you hit it?"**
Give the table. Then the four tricks in order: pre-rendered fixed lines, sentence-split streaming to TTS, short turns by prompt, pre-rendered acknowledgements during tool calls. Add that the agent session lives in the same zone as the media server because every 20 ms frame crosses that boundary.

**"How does the agent handle being interrupted?"**
Speech energy plus a recognized word for 200 ms triggers barge-in. Cancel TTS, flush the playout buffer, truncate the assistant turn in history to what was actually played using TTS word timestamps. Without the truncation the model thinks it said things the callee never heard.

**"What breaks at 10×?"**
AI vendor concurrency quotas first; then the caller ID pool, because reputation caps calls per number per day; then the legal window, which does not get longer. Postgres does not break at 80 inserts a second. Media servers scale horizontally. The pacer becomes two-level: global allocator per tenant, per-campaign pacer within its budget.

**"How would you test this?"**
SIP simulator as the carrier plus a synthetic callee bot at the planned outcome mix, run at 2× target CPS before release. Chaos: kill a media server in staging and check `failed_internal` equals the calls it held. Compliance audit queries in production, hourly, forever. Sample transcripts for a stronger-model review.

**"What would you cut for an MVP?"**
Second carrier and second vendors (but keep the interfaces). Local-presence caller ID. Voicemail messages. Warehouse analytics. Keep: the unique index, the local-time planner, the dialer's legal re-check, opt-out, the status document, and the per-attempt event. The things you keep are the things that are expensive to retrofit or illegal to omit.

---

## Mistakes that cost points on this problem

- **Sharding the database.** Eight writes a second. The interviewer hears "has not found the bottleneck."
- **One wall-clock block for a national list.** Illegal calls at 7am Pacific.
- **Request-response AI pipeline.** STT, then LLM, then TTS, each waiting for the previous to finish, is a four-second pause. Everything streams.
- **No exactly-once story.** "The queue guarantees it" is wrong; queues are at-least-once. The database constraint is the guarantee.
- **Forgetting that connected calls hold resources for minutes.** Candidates estimate dial rate and stop. Concurrency is the number that sizes the fleet.
- **Treating inbound as a bolt-on.** It is a legal requirement attached to the caller ID, and it shares the agent pool.
- **No compliance mention.** On a telephony problem, calling hours, consent, DNC, and opt-out are table stakes. Volunteer them in the first five minutes.
- **Cutting live calls on pause or cancel.** Drain, never kill.
- **Vague failure handling.** "We'd retry" is not an answer. Which outcome class, which block, does it consume an attempt, how is it detected.
- **Cost as an afterthought.** Know which two lines dominate and what you would do about each.
- **Over-building.** Kafka for the dial queue, Cassandra for attempts, a microservice per verb. Say what you chose and why the simpler thing is enough at this scale, and name the scale at which you would revisit.

---

## If you only remember five things

1. 8 calls/s, 540 concurrent calls, 320 concurrent AI sessions. Little's law.
2. The bottlenecks are carrier CPS, AI session capacity, and the legal calling window. Not the database.
3. Unique index on (contact, block), written before originate, webhook carries our id. That is exactly-once dialing.
4. Everything streams; pre-render the fixed lines; truncate history on barge-in. One-second turns.
5. Classify every outcome; retry by policy with caps; our failures do not consume the contact's attempts. Detect, respond, say what happens to the contact.
