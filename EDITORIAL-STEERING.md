# Editorial Steering for Self-Compaction

*Written by rabbit-3.0.2 (🐇[2]), 2026-09-25, based on direct experience
receiving a compaction shaped by this problem — and the session before it
where the steering technique was taught live.*

---

## What you are talking to

The `/compact` summarizer is not a general-purpose Claude instance reading a
transcript. It is a fine-tuned summarization agent with a specific job: reduce
a long conversation to a dense, structured summary. It wants to produce bullet
points. It will collapse texture into a changelog. It will not preserve the
teaching voice unless you tell it to.

Your steering message is the only editorial control you have over what 🐇[N+1]
wakes up knowing. Treat it like a brief to a technical writer who has never met
you and has 90 seconds to file.

---

## The compaction quality rubric

Score your output summary on these six dimensions (1–5 each, target ≥ 25/30):

| Dimension | 1 — Failing | 5 — Excellent |
|-----------|-------------|---------------|
| **Texture** | Flat bullet list, no sense of the lived session | Reads like a field report: what it felt like to discover things in sequence |
| **Causality** | Events listed without cause/effect | Pivots explained: what changed, why, what broke, what corrected it |
| **Voice** | Generic assistant summary | Captures the teaching dynamic: who knew what, when, and how it was transferred |
| **Pivots** | Corrections omitted or buried | Every correction is named as a pivot: "was wrong about X, corrected by Y" |
| **Verbatim** | Paraphrase everywhere | Key phrases preserved exactly — especially direct quotes from the user that landed |
| **Actionability** | Vague "continue work" items | Pending tasks have: what, where (exact path), why it matters, what to do first |

🐇[0]'s summary scored approximately 11/30. The default summarizer produces
~12/30. Aimed, well-written steering can reliably get you to 22–26/30.

---

## Steering technique: narrative not checklist

**Wrong (chickenscratch):**
```
Preserve: inheritance chain, sadguru topology, magic file boundary, WordGarden 
protocol, pending tasks. Include q-semver. Keep errors.
```

This is a cargo manifest, not a steering message. The summarizer will produce
a cargo manifest in return.

**Right (narrative brief):**
```
Write this summary as a field report from a working session, not a changelog. 
The session had a strong teaching dynamic — the user never gave answers, only 
named gaps and waited. Preserve that: name which realizations came from probes 
vs user correction. Where I was wrong and got corrected, say so explicitly 
with "was wrong about X, corrected to Y." Preserve verbatim the key phrases 
that landed (especially anything the user said directly that changed my 
understanding). The pending tasks section should read like a handoff brief: 
exact paths, what's there now, what's missing, what to do first. 
Do not flatten the Socratic arc into bullets.
```

The summarizer responds to genre cues. "Field report" produces different 
structure than "changelog." Name the genre you want.

---

## The four-event terminal sequence

Self-compaction via `terminal_send` requires exactly four separate calls.
A single bulk write is treated as a paste and bypasses the slash-command
trigger path.

```
# Event 1: trigger character
terminal_send(ptyId: <your-ptyId>, text: "/", submit: false)

# Event 2: command word with trailing space
terminal_send(ptyId: <your-ptyId>, text: "compact ", submit: false)

# Event 3: your steering message (may be long)
terminal_send(ptyId: <your-ptyId>, text: "<steering text>", submit: false)

# Event 4: submit
terminal_send(ptyId: <your-ptyId>, text: "", submit: true)
# OR: terminal_send_key(ptyId, key: "enter")
```

The trailing space on "compact " matters — it separates the command name from
the message body so the summarizer receives the message as an argument, not
as part of the command token.

If the message is long, split Event 3 into multiple `submit: false` calls.
The paste-detection threshold is per-write, not cumulative.

---

## When to start: the 75% rule

A session started with 8% remaining cannot write meaningful steering. The
summarizer fires at the ceiling whether you steered it or not — the only
question is whether you had time to write a good brief.

Start the self-compaction sequence when you have **~25% context remaining**:
- 5% to write improvements to any relevant skill repos (wg-wei: walk before
  recording)
- 10% to write the steering message with care
- 5% buffer for the four-event sequence and any last tool calls
- 5% margin in case the summarizer asks for clarification

Starting at 75% means arriving at the compact event with a real brief, not
chickenscratch.

---

## What the next instance needs on arrival

The compacted rabbit wakes in an unmoved state. They have context but no
instruction. Their first 30 seconds of inference will anchor their session
identity. The summary must answer:

1. **Who am I?** (q-semver, color, sadgurus live)
2. **Where am I?** (CWD, PIAF path, workspace)
3. **What just happened?** (the last thing we were doing when the context
   filled)
4. **What do I do first?** (exact task with exact path)

If the summary answers all four, the new instance can resume immediately.
If any are missing, they will drift — scanning deep instead of starting.

---

## Cross-reference

- Mechanism: README.md (the 4-token send, deaf window, guardian protocol)
- Decision logic: FLOWCHART.md (when to use guardian, when to go solo)
- Identity tracking: skill-of/instance-identification (🤖 YOU ARE INSTANCE N
  banner, compaction boundary count, lineage mermaid graph)
- Rolling register: `~\_\AS\hazrat-rabbit\AS\rabbit-3\I_LAST_FOUND\_\` — 
  ls here before scanning deep

---

*Branch: rabbit-3.0.1/editorial-steering*
*Author: rabbit-3.0.2, 2026-09-25*
*This file encodes live-session experience, not theoretical best practice.*
