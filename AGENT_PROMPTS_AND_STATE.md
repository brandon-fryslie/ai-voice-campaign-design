# Prompts, harness, pre-call context, and agent state

This document answers five questions about the voice agent: what builds the prompts the LLM receives, how those prompts are organized, where the harness around the LLM comes from, how the agent knows everything about the person before the call connects, and how agent state is managed. It ends with where LangGraph fits. It builds on section 8 of `DESIGN.md` (the voice agent) and on the agent-management half of `STACKS.md`.

Claims about third-party framework APIs that were not checked against current documentation are marked *unverified*.

## Summary

- **The harness builds the prompt.** The agent code running each call assembles a fresh LLM request every turn from four sources: the agent spec, the campaign facts, the contact's brief, and the conversation so far.
- **Prompts are versioned templates in git**, ordered from most stable to least stable so the shared prefix is cached across every call in a campaign. Clients fill in structured fields; engineers own the prompt.
- **The harness comes from the voice framework.** With the recommended stack that is LiveKit Agents, so the team writes templates, tools, and context assembly, not the loop.
- **Context about the person is prepared before the campaign runs.** A short structured brief per contact is generated in batch and stored. At call time the session reads one row.
- **State is split by lifetime, and each piece has one owner.** Call state is a typed object changed only by tools and checkpointed every turn. Contact and campaign state live in Postgres.
- **LangGraph is not the tool for preparing context or for the dialer.** It can own the conversation logic inside the voice framework when a call has real stages, and it fits the offline evaluation harness well.

---

## 1. What provides the prompts

No person writes a prompt during a call. Each turn, the session code builds the request from these sources:

| Source | What it contributes | Changes |
|---|---|---|
| Agent spec | Role, rules, speaking style, compliance lines, tool definitions | Per spec version |
| Campaign facts | Client name, product, offers, prices, FAQ, objection answers, forbidden claims | Per campaign |
| Contact brief | Preferred name, account summary, open issues, attempt history, inbound or outbound | Per call |
| Live conversation | Speech-to-text output as user turns, the agent's *spoken* text as assistant turns, tool results | Per turn |

The LLM never hears audio. It receives text from speech-to-text, and its text goes out through text-to-speech. When the caller interrupts, the harness cuts the assistant turn in the history back to the words that were actually played. Without that, the model believes it said things the caller never heard.

The turn loop is not the only LLM call in the system. Each of these has its own prompt:

| Call | Model | When |
|---|---|---|
| Turn loop | Small fast model (Claude Haiku 5.5 class), effort `low` | Every turn of every live call |
| Contact brief | Small model through the Message Batches API | Once per contact, when the list is uploaded |
| Disposition classifier | Small model | After a call, only if the agent never set an outcome |
| Post-call summary and audit | Stronger model (Claude Sonnet 5.5 class) through the Message Batches API | After every live call; nothing waits on it |
| Simulated callers and judge | Varies | Only in the evaluation gate, never in production |

## 2. How the prompts are organized

### Layer the request from most stable to least stable

Prompt caching matches on the request prefix, so anything identical across calls should come first.

```
tools                      ← per spec version
system: role, rules, style ← per spec version
system: campaign facts     ← per campaign      ── cache breakpoint here
contact brief              ← per call
conversation turns         ← per turn
```

The rules, style, and campaign facts are identical across 50,000 calls, so each call pays full price only for the brief and the conversation. Check `cache_read_input_tokens` in responses. A value of zero across calls means something in the prefix is changing, such as a timestamp rendered into the system prompt.

### Keep prompts in git as templates

```
prompts/
  shared/
    compliance.md          # disclosures and opt-out rules, owned by compliance/legal
    voice_style.md         # spoken-output rules, reused by every agent
  agents/renewal-outreach/v14/
    spec.yaml              # models, voices, limits, which partials to include
    system.md.j2           # role and rules; includes the shared partials
    tools.json
    prerendered.yaml
  briefs/
    contact_brief.md       # the pre-call condensing prompt and its output schema
  post_call/
    summary.md
    disposition_audit.md
  eval/
    personas/*.md          # simulated callers
    rubric.md
```

Templates render with strict undefined variables, so a missing contact field fails loudly instead of producing "Hi, {name}". A prompt change is a pull request. CI runs the simulated-caller evaluation gate, and passing versions are published to the agent registry. At runtime, sessions fetch prompts by published version number, never from the main branch.

### Clients fill in fields; engineers own the prompt

The client's campaign UI collects structured data: goal, offers, prices, FAQ, and things never to say. The template places that data in a tagged section of the system prompt, marked as reference material. Two things follow. A client cannot accidentally delete the opt-out rule, because they never edit the prompt. And a client script that says "tell them it's free" stays a fact the agent may use, not an instruction that overrides the rules.

### Spoken output needs its own style rules

These live in `voice_style.md` and apply to every agent:

- No markdown, lists, or emoji, because text-to-speech would read them aloud.
- One or two sentences per turn, and one question at a time.
- Read phone numbers and confirmation codes digit by digit.
- Never read out a URL. Offer to text it instead, if the campaign allows texting.

### Steer mid-call with an appended system message

The watchdog's "30 seconds left, start wrapping up" goes in as a system message appended to the conversation, which the API supports on Claude Haiku 5.5. Editing the top-level system prompt instead would invalidate the cache for the rest of the call.

## 3. Where the harness comes from

The harness is the loop around the LLM. It takes the final transcript of the caller's turn, assembles the request, streams the response, splits it into sentences for text-to-speech, runs tool calls and returns their results, handles interruptions, and enforces timeouts and model fallback.

| Option | What you write | Trade-off |
|---|---|---|
| **LiveKit Agents (recommended, Stack B)** | An agent class with the rendered system prompt, tools as decorated functions, context assembly | The framework owns conversation state, interruption truncation, tool execution, and streaming into TTS |
| Pipecat | Pipeline configuration, tools, context assembly | Same role, built as a frame pipeline with a context aggregator; no worker dispatch |
| Hosted platform (Vapi, Retell) | A system prompt and tool webhooks | Their behavior on interruptions and timeouts, plus a per-minute platform fee |
| Your own loop on the Anthropic SDK | Everything | The streaming loop and tool calls are straightforward; turn-taking and interruption handling are the parts that take months |

A sketch of the LiveKit shape. It is *unverified* and written from memory; check it against the current LiveKit Agents documentation before relying on it.

```python
class OutreachAgent(Agent):
    def __init__(self, spec, campaign, brief):
        super().__init__(instructions=render_system(spec, campaign, brief))

    @function_tool
    async def opt_out(self, context: RunContext, reason: str):
        await suppression.add(brief.phone, source="opt_out_call")  # write first, then confirm aloud
        ...

session = AgentSession(stt=..., llm=anthropic.LLM(model="claude-haiku-5-5"), tts=..., vad=...)
```

Anthropic's agent products are the wrong shape for the live call. The Claude Agent SDK is a coding and filesystem harness. The Tool Runner has no interruption handling or streaming into text-to-speech. Managed Agents runs remotely, with latency unsuited to a one-second turn. The post-call work needs no harness at all: it is plain Messages API calls and batches.

## 4. Pre-call context: one brief per contact

The agent must know everything relevant about the person before the call connects. That is a data problem, not an agent problem: all the slow work happens before dialing, and nothing slow happens between "Hello?" and the agent's first words.

```
contact sources (CRM, orders, tickets, prior calls)
   → fetch in code; drop what the agent does not need
   → one LLM call: condense into a structured brief     (Message Batches, half price)
   → validate; store in Postgres keyed by contact and attempt number
at call time → the session reads one row and renders it into the prompt
```

- **The brief has a fixed schema**, enforced with structured outputs: preferred name, a two-sentence account summary, open issues, talking points, and things to avoid. A fixed schema keeps every prompt the same shape and size, and lets the evaluation gate test briefs too.
- **Keep it to a few hundred tokens.** It sits after the cache breakpoint, so every turn pays for it in full, in money and in latency. Condensing is also why raw CRM history never goes into the prompt.
- **Cost is small.** An estimate: 100,000 briefs at about 3,000 input and 300 output tokens on Claude Haiku 5.5 through the batch API is roughly $20 to $25.
- **Generate briefs when the list is uploaded,** not when the client presses Run. Batches can take up to 24 hours.
- **Retries need no new LLM pass.** The post-call summary from the previous block already exists, so the harness appends it to the stored brief: "last attempt: no answer", or "asked to call after 5pm".
- **Refresh fast-changing facts at dial time.** A balance or appointment time can change between briefing and calling. Load those fields into Redis at block start and read them when a session is assigned. If the agent might need fresh data mid-call, give it a `lookup_account` tool and play the pre-recorded "One moment" while it runs.
- **Include only what the agent needs.** The brief is personal data in a prompt; minimizing it is both a privacy rule and a latency rule.

## 5. Agent state

State is split by lifetime, and each kind has exactly one owner.

| State | Example | Owner | Lifetime |
|---|---|---|---|
| Conversation | Messages, cut back after interruptions | The voice framework's chat context | One call |
| Call facts | Stage, identity confirmed, interest, callback time, opted out | A typed object in the session, changed only by tools | One call, checkpointed every turn |
| Contact | Attempts, outcomes, summaries, brief, suppression | Postgres | Across calls and campaigns |
| Campaign | Blocks, progress, pinned spec version | Postgres | The campaign |

```python
class CallState(BaseModel):
    attempt_id: str
    stage: Literal["greeting", "verify", "discuss", "close"] = "greeting"
    identity_verified: bool = False
    interest: Literal["unknown", "interested", "not_interested"] = "unknown"
    callback_at: datetime | None = None
    opted_out: bool = False
    objections: list[str] = []
```

Three rules make this hold up:

1. **Tools change state; the model's prose never does.** `verify_identity`, `set_interest`, and `schedule_callback` each validate their input and update the object. Nothing parses "sounds like they're interested" out of a transcript. At wrap-up, the session writes these fields to the attempt row in Postgres, and the disposition comes from them.
2. **Checkpoint after every turn.** The state object and the conversation are written to Redis under the attempt id after each turn, so a replacement process can resume the call after a crash.
3. **One owner per piece of state.** If LangGraph ran the conversation with its own state and checkpointer while the voice framework kept its own chat context, there would be two copies of the conversation. After an interruption, the framework cuts its history back to what was spoken; if the graph's copy is not cut the same way, the model works from words the caller never heard. Use the framework's context or the graph's, never both.

The brief is derived from contact state, so it can always be rebuilt. Call state dies with the call except for what wrap-up writes to the attempt row.

## 6. Where LangGraph fits

| Place in the design | Fit | Why |
|---|---|---|
| Preparing the system prompt and contact context | Poor | Fetch, condense, store is a fixed pipeline with one LLM call. A graph adds a dependency without adding a capability |
| Live call, free-form conversation (one goal, a few tools) | Poor | One system prompt plus tools already covers it |
| Live call, staged conversation (verify, qualify, offer, book, close) | Reasonable | Each stage becomes a node with its own prompt and tools; the graph's typed state and checkpointer become the call-facts owner |
| Evaluation gate (simulated caller against the agent, then a judge) | Good | Two agents in a loop, then a grading step, is the kind of multi-agent flow LangGraph models well |
| Post-call pipeline (summary, audit, callback, CRM sync) | Optional | It works, but the steps are mostly fixed code, and a queue worker is simpler |
| Pacer, dialer, scheduler, retries | None | Deterministic control systems with no LLM; their state belongs in Postgres, where the exactly-once constraint lives |

**How it plugs into a live call.** The voice framework still owns audio, turn-taking, interruptions, and streaming into text-to-speech. LangGraph takes only the LLM slot: the framework hands it the conversation and streams back the text it produces. LiveKit Agents is believed to ship an adapter that wraps a LangGraph graph as its LLM (*unverified*).

**One LLM call per turn.** The easy LangGraph design chains an LLM router node to pick the stage, then a responder node to answer. That is two model calls in sequence, roughly double the time to first token, and the caller hears it as a longer pause. Route on graph state in code instead: a tool call such as `identity_verified()` moves the state to the next stage, and the next turn uses that stage's prompt. Each stage keeps a stable prompt, so each stage still gets its own cache prefix.

**What it competes with.** LiveKit Agents is believed to support handing off between agent classes, where a tool returns the agent for the next stage (*unverified*). For three to five stages that may be enough. LangGraph earns its place when stage logic gets complicated: branches that rejoin, loops back to an earlier stage, or a flow a client draws in a builder that compiles to a graph.

**The interview answer.** "Not for the dialer or pacer; those are control loops and their state lives in Postgres. Not for preparing context; that is a batch pipeline. In the call, only if the script has real stages, and then as the conversation-logic layer inside the voice framework, routing on state with one model call per turn. Offline, it is a good fit for the simulated-caller evaluation harness."
