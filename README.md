# AI voice campaign platform: system design

A system design for an outbound AI voice campaign platform. A client uploads 100,000 mobile numbers and presses Run. Within four hours every number has been dialed from the PSTN by an AI voice agent that listens (speech-to-text), decides what to say (an LLM), and speaks (text-to-speech). Numbers that did not connect may be retried in later four-hour blocks. Anyone who calls the caller ID back reaches the same agent.

This was prepared for a system design interview. It goes deep on what interviewers look for: requirements, capacity math, architecture, time to run, status, failure rates, monitoring, cost, compliance, and trade-offs. It was worked out in a conversation with Claude Code, starting from the prompt below and refined through the follow-up questions listed after it.

## The initial prompt

Quoted as it was given:

> Okay, let's change this to a marketing campaign scheduler that you can schedule 100,000 calls. You can make the calls in four hours. If one of them fails, there's a chance for that call to be retried in the next four hour block. It's an AI system that we will make the calls when the client wants to Run the campaign.  We'll make making the calls from PSTN to mobile phone numbers. We have to, the system has to be able to take mobile has to be able to take PSTN calls into our systme.  This will be an AI system, we need sst to translate to text, an LLM, and we need STT to provide speech out to the user.  Build this system.  It's for an interview and we need to know how to design this and all of the various things interviewers are looking for.  time to run, status, everything.  Go deep.  Failure rates, monitoring, cost, description, everything

The prompt says "STT" for the speech output step; the design reads it as text-to-speech (TTS).

## Follow-up questions

Each of these produced part of the documents:

| Question | Answered in |
|---|---|
| "Propose me several software stacks with pros and cons, and explain to me how the AI agents would be managed" | `STACKS.md` |
| "What provides the prompts to the LLMs? How do we organize this? Where do we get the harness?" | `AGENT_PROMPTS_AND_STATE.md`, sections 1 to 3 |
| "Where would langraph fit in here?" | `AGENT_PROMPTS_AND_STATE.md`, section 6 |
| "The agent needs to have all the context for the target before we called them. Is langraph a good tool to use to set up this system prompt for this agent? I also want to manage the agent state." | `AGENT_PROMPTS_AND_STATE.md`, sections 4 to 6 |

## The documents, in reading order

| File | What it covers |
|---|---|
| `DESIGN.md` | The full design: requirements and legal constraints, capacity estimation, architecture, campaign lifecycle and the status API, the scheduler and pacer, exactly-once dialing, the voice agent and its latency budget, inbound calls, retry rules, data model, time to run, failure rates and failure modes, monitoring, cost, compliance, scaling, and rejected alternatives |
| `STACKS.md` | Four software stacks with pros and cons and a recommendation, then how the AI agents are managed: versioned agent definitions, the per-call session lifecycle, dispatch, scaling and deploys, supervision, and evaluation |
| `AGENT_PROMPTS_AND_STATE.md` | What builds the LLM prompts and how they are organized, where the harness comes from, how the agent gets each person's context before the call, how agent state is managed, and where LangGraph fits |
| `INTERVIEW.md` | How to present the design in 45 to 60 minutes: a 90-second pitch, clarifying questions, numbers to memorize, what interviewers score, likely follow-up questions with answers, and common mistakes |

## The five numbers

| | |
|---|---|
| Dial rate | 8 calls/s (100,000 attempts in 3.5 h, keeping a 30-minute tail) |
| Concurrent calls | ~540, provision 800 (Little's law: 8/s × 67 s mean hold) |
| Concurrent AI conversations | ~320, provision 500 |
| Agent turn latency | p50 1.0 s, p95 1.8 s |
| Cost per 100,000-contact campaign | $2.7k to $14k over three attempt blocks; telephony and TTS dominate |

## The architecture in one paragraph

A control plane plans the campaign into four-hour blocks in each callee's local time. It paces dials with a token bucket bounded by carrier calls-per-second limits, channel capacity, and AI session capacity. Stateless dialer workers claim an attempt row before dialing, and the row is unique on contact and block, so no one is dialed twice. They originate through one of two carriers. Stateful media servers hold the calls and feed 20 ms audio frames to a per-call voice agent. The agent streams through STT, a small fast LLM, and TTS within a one-second turn budget, and handles interruptions. Every attempt ends with one event carrying everything known about it, which feeds status, metrics, billing, and the retry planner. Failed outcomes are classified and retried in the next block unless a cap says no. Inbound calls to our caller IDs are matched by number and routed to the same agent with campaign context.

## Caveats

- **Prices are 2026 ballpark figures, not quotes.** Telephony, STT, TTS, and hosted voice platform prices come from general knowledge and were not checked against vendor price sheets. LLM prices are Anthropic list prices.
- **Some framework API details are marked unverified.** The LiveKit Agents code sketch, the LangGraph adapter, and the current Amazon Connect feature set should be checked against current documentation.
- **The traffic assumptions are typical, not measured.** The outcome mix, conversation length, and callback rate are assumptions for consented consumer outreach in the US. The design measures them live rather than depending on them.
- **The compliance section is a set of design constraints, not legal advice.** TCPA, Do Not Call, calling hours, and recording consent rules change and vary by state.
