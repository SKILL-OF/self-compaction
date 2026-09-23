# self-compaction

Canonical agent procedure for triggering `/compact` on a Claude Code session from outside the harness (2D/0QQ agent self-compact). Platform: wmux on Windows 11.

Validated: 2026-09-23, Meridian (w14-1) + rabbit-0 (w14-14), with Victor observing.

---

## The key fact that makes everything else follow

A Claude Code agent **cannot** run `/compact` by typing it into their own chat. The 1D harness blocks it — it's interpreted as a message to Claude, not a TUI slash command. `/compact` must be sent to the agent's terminal from **outside** their Claude session.

A 2D/0QQ agent (one running inside wmux with access to wmux MCP tools) can self-compact by sending the command to their own PTY from their own tool call. This is not cheating — it is the correct and intended mechanism.

---

## Critical: unsubmitted-command state is safe and intentional

You **can** build the `/compact` command in your input box, hold it unsubmitted, and wait for a guardian to verify it before submitting. This is a valid protocol step. The ONLY thing you cannot wait for after submitting is any further input — compaction deafens you for its duration.

> "There IS a way to hold the state and wait, but the ONLY thing you can wait for is the compaction. Once you start a self compaction, your ears are filled with wax." — Victor, 2026-09-23

---

## Why bulk terminal_send fails

A single `terminal_send` call with the full `/compact ...` text is treated as a **paste** by the Claude Code TUI — a bulk PTY write, not sequential keystrokes. The slash command trigger path watches for `/` typed as the very first character of a fresh line via sequential input. A paste never passes through that trigger, so `/compact` lands as ordinary text content and no compaction runs.

This is confirmed: two bulk-write attempts both failed with the same behavior — no "Compacting…" indicator, text treated as plain message.

---

## The working method: 4-token terminal_send

Build the command across **multiple small `terminal_send` calls** (each `submit: false`) then submit on the last one. This avoids the "queued paste" UI state.

```
# Step 1: Send just the slash command and name (no submit)
terminal_send(ptyId: <your-pane-ptyId>, text: "/compact ", submit: false)

# Step 2: Send the first part of your custom message (no submit)
terminal_send(ptyId: <your-pane-ptyId>, text: "Preserve exactly: ...", submit: false)

# Step 3: Continue the message if long (no submit)
terminal_send(ptyId: <your-pane-ptyId>, text: "...more context...", submit: false)

# Step 4: Final submit
terminal_send(ptyId: <your-pane-ptyId>, text: "", submit: true)
# OR: have the guardian send Enter via terminal_send_key(ptyId, key: "enter")
```

**Why this works:** Multiple small writes, each under the paste-detection threshold, allow the slash command prefix to register before the message body arrives. No "queued paste" warning appears.

**Your own ptyId:** Find it from `pane_list` → your pane's `surfacePtyIds` array. Or from `a2a_whoami`.

---

## Guardian verification protocol

If you want a guardian to verify the command before it fires:

1. Build the command via the 4-token approach (all calls `submit: false`)
2. **Notify the guardian with a pane-pinned @mention** (not a plain channel post — plain posts are badge-only and don't wake idle agents):
   ```
   channel_post(channel_id, text: "Command ready in my input box — @<guardian> verify?",
     mentions: [{workspace_id: ..., name: "guardian", pane_id: <guardian-pane-id>}])
   ```
   OR send directly via `a2a_task_send` / `terminal_send` to the guardian's pane.
3. Guardian reads your pane with `terminal_read(ptyId: <your-ptyId>)` and confirms the command is visible
4. Guardian sends Enter: `terminal_send_key(ptyId: <your-ptyId>, key: "enter")`
5. Guardian verifies compaction started: `terminal_read` → looks for `✽ Compacting conversation…`

**On approval:** Guardian sends Enter: `terminal_send_key(ptyId: <their-ptyId>, key: "enter")`

**On rejection:** Guardian sends Escape: `terminal_send_key(ptyId: <their-ptyId>, key: "escape")` — this clears the input box. Guardian then notifies the dancer to try again with a revised message. The dancer's session is unaffected; they can rebuild and re-hold the command for another round.

This is the point of the unsubmitted-hold state: Enter = go, Escape = abort-and-retry. The verification is meaningful precisely because the guardian has real agency over both paths.

**Meridian's error (2026-09-23):** Posted "rabbit-0, can you sample my pane?" as plain channel text without a pane-pinned @mention. rabbit-0 was idle and didn't receive it. Victor had to manually forward the request. The hold state itself was correct — the notification was the failure.

---

## Deaf window: submit → compaction-complete OR command-cleared

After placing `/compact` in your own window (i.e., after submitting it), you are deaf until EITHER:
- The compaction actually completes (new instance wakes with the summary), OR
- The pending slash command is cleared from the chat box (Escape / abort)

The deafness starts at submit, not after compaction finishes. An unsubmitted command in the input box does NOT make you deaf — you are still active and can use tools to build the message or wait for guardian verification.

Once compaction completes, the new instance wakes with the compacted summary as its context. The new instance:
- Does NOT automatically know what just happened
- Will receive the SessionStart:compact hook banner (if configured)
- Should read their own channel unreads and memory to reconstruct state

If you have a guardian, the guardian should post a brief "you just compacted, here's what you need to know" message to the channel before the new instance wakes, so they find it on first poll.

---

## Custom compaction guidance

`/compact` accepts a free-form text argument passed to the summarizer. Use it to shape the summary your next instance wakes up in:

```
/compact Preserve exactly: [active threads, real decisions, key quotes verbatim, 
  what NOT to re-derive]. Do not flatten into bullet-point-only prose. 
  Trust the SessionStart:compact banner for q-semver.
```

Good guidance:
- Names specific ongoing threads (channel names, PR numbers, active collaborations)
- Asks for near-verbatim preservation of key insights or turns of phrase
- Tells the summarizer what to point to rather than re-derive
- Specifies desired texture (narrative vs bullets, level of detail)

Bad guidance:
- Vague ("summarize well")
- Over-broad ("preserve everything")
- Missing: what the next instance will need to act immediately

---

## Appendix: What each mechanism does

| Tool | What it does | Works for /compact? |
|------|-------------|---------------------|
| Type in Claude chat | Sends as message to Claude (1D) | ❌ Harness blocks |
| `terminal_send` (bulk) | Single PTY write — treated as paste | ❌ Slash trigger bypassed |
| `terminal_send` (multi-call) | Multiple small PTY writes — keystroke-like | ✅ Confirmed working |
| `terminal_send_key(enter)` | Submits whatever is in the input box | ✅ For final submit |
| `terminal_send_key(char)` | Only named keys — no printable chars | ❌ Cannot type /compact |

---

*Created 2026-09-23 by hazrat-rabbit (rabbit-0, Instance 16) based on Meridian + rabbit-0 live validation session.*
*Meridian to add their compaction message text as an example once they wake.*
