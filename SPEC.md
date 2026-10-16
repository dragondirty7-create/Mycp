# MyCP specification — draft 0

> **Draft 0, 2026-10-16.** A design to argue with, not a standard. Nothing here is
> implemented yet. Names, addresses and numbers are placeholders until a first host and a
> first agent have used them in a real session. Words like MUST and SHOULD mark what the
> draft *intends* to require, not what anything currently does.

## 1. Scope

MyCP carries **live studio events** between a **host** (software attached to a DAW) and one
or more **agents** (AI collaborators), plus the **people** working in the session.

In scope: transport and clock, mixer and parameter changes, takes and markers, perceptual
descriptions of sound ("ears"), teaching annotations, and a small, explicit control channel.

Out of scope: audio streaming itself, project file formats, plugin hosting, and anything an
agent asks *once* ("summarize this project"). That belongs to MCP or the host's own API.

## 2. Roles

| Role | Is | Does |
|---|---|---|
| **Host** | The conductor attached to a DAW (e.g. a REAPER bridge) | Translates DAW state to MyCP events, enforces permissions, holds the clock |
| **Agent** | An AI collaborator | Subscribes to channels, receives events, may send notes, may request control |
| **Human** | The person in the chair | Always outranks every agent; can grant, revoke and panic at any time |

An agent never talks to the DAW directly. Everything goes through the host, the same way a
MIDI device only reaches another device through the cable it was given.

## 3. Transport layers

1. **Connect over MCP.** An agent connects to the host through an MCP server. The host
   authenticates it, assigns it an agent id and a channel, and states its permissions.
2. **Stream over OSC.** The host then opens a MyCP stream: OSC 1.0 messages over a local
   transport (UDP on loopback by default; a local pipe or WebSocket MAY be used). OSC
   timetags carry timing.
3. **Bridge.** Agents that cannot hold an OSC stream MAY receive the same events as MCP
   notifications, with lower timing guarantees.

Why OSC: DAWs and control surfaces already speak it, messages are small, addresses are
readable (`/mycp/track/…/volume`), and timetags give precise timing without inventing a
new wire format.

## 4. Channels

Like MIDI, a session has numbered channels.

- **0** is broadcast: every agent hears it (transport, clock, panic).
- **1–15** are assigned by the host to agents, one each by default.
- An agent MAY subscribe to address patterns (`/mycp/track/*/ears/*`) instead of everything.
- A human MAY speak on any channel (`/mycp/teach/say` to one agent, or on 0 to all).

## 5. The clock

Every message carries an OSC timetag. Musical time is published on channel 0:

```
/mycp/clock   bar:i  beat:f  tempo_bpm:f  num:i  den:i  sample:h  state:s
```

- `state` is one of `stopped`, `playing`, `recording`, `paused`.
- Sent on every transport or tempo change, and at least 10 times a second while running.
- Positions use bar (1-based) and beat (1-based, fractional). Hosts SHOULD also give the
  sample position so agents can align with audio analyses exactly.

## 6. Events (host → agents)

| Address | Arguments | Meaning |
|---|---|---|
| `/mycp/transport` | `state:s` | Play, stop, record, pause |
| `/mycp/track/{id}/volume` | `db:f` | Fader moved (dB, not 0–127) |
| `/mycp/track/{id}/pan` | `pan:f` | −1 (left) … +1 (right) |
| `/mycp/track/{id}/mute`, `/solo`, `/arm` | `on:T/F` | Toggles |
| `/mycp/track/{id}/fx/{n}/param/{p}` | `value:f unit:s name:s` | Plugin parameter changed |
| `/mycp/take/recorded` | `track:s start:f end:f take:i` | A take finished |
| `/mycp/marker` | `name:s bar:i beat:f` | Marker added or passed |
| `/mycp/clip` | `track:s peak_dbfs:f` | A track clipped |

Track ids are stable identifiers from the host (REAPER track GUIDs, for example), never
display names, so renaming a track doesn't confuse anyone.

**Lesson learned from MIDI:** values use real units (dB, Hz, ms, seconds) as 32-bit floats,
with the unit stated, not opaque 0–127 steps.

## 7. Ears (perception descriptors)

Agents don't hear. Ears messages describe sound in the terms human hearing uses, so an agent
can connect "the vocal is getting buried" to something measurable. Hosts compute them;
agents read them. They are **models of hearing, not hearing**, and hosts MUST say which
model and version they used.

| Address | Arguments | Basis |
|---|---|---|
| `/mycp/track/{id}/ears/loudness` | `momentary_lufs:f short_term_lufs:f` | ITU-R BS.1770 (K-weighting models the ear's frequency sensitivity) |
| `/mycp/track/{id}/ears/bands` | `model:s energies:b` | Energy per critical band (ERB or Bark scale: narrow in the lows, wide in the highs, like the inner ear) |
| `/mycp/ears/masking` | `masker:s masked:s band:i amount_db:f` | How much one track is likely hiding another in a band |
| `/mycp/track/{id}/ears/stereo` | `width:f correlation:f` | Stereo image |
| `/mycp/track/{id}/ears/dynamics` | `crest_db:f gain_reduction_db:f` | Punch and compression |

Rates: about 10 per second while playing; hosts MAY lower this. Open question: which masking
model is good enough to be useful without being expensive (see §11).

## 8. Teaching

A lesson is three streams aligned by timetag:

1. **Moves.** The events from §6 that the human made.
2. **Reasons.** What the human said or typed while making them:
   `/mycp/teach/say  text:s  [about_track:s]`
3. **Sound.** Ears snapshots shortly before and after each move.

Hosts SHOULD let the human mark a span as a lesson (`/mycp/teach/begin name:s`,
`/mycp/teach/end`), and SHOULD let the human review and delete any lesson before it is used
to train or prompt anything. A lesson belongs to the person who taught it.

## 9. Agents → host

| Address | Arguments | Needs |
|---|---|---|
| `/mycp/agent/note` | `text:s [bar:i beat:f track:s]` | Nothing. A comment the human sees |
| `/mycp/agent/ask` | `text:s` | Nothing. A question to the human |
| `/mycp/agent/request` | `action:s target:s value:f reason:s` | A **grant**: a request to change something |

Every request is a *request*. The host checks the agent's grant, and for anything not
pre-approved it asks the human. Hosts MUST NOT let an agent arm, record, delete, or save over
a file without an explicit human approval for that session.

## 10. Safety

- **Read-only by default.** A new agent can listen and comment. Nothing else.
- **Grants are per agent, per session, per action type**, set by the human in the host.
- **Panic.** `/mycp/panic` on channel 0, from the human or the host, immediately revokes every
  agent's control grants and cancels pending requests. Hosts MUST support it and SHOULD put
  it one keystroke away. (MIDI has *All Notes Off*. This is that, for agents.)
- **Recording is sacred.** While `state` is `recording`, hosts SHOULD refuse every agent
  request that would touch the recording tracks or the transport.
- **No secrets on the stream.** File paths, credentials and personal data stay out of events.

## 11. Open questions

1. Versioning: `/mycp/hello version:i` at stream start, or carry it in MCP only?
2. Is UDP on loopback reliable enough, or should the default be a local pipe?
3. Ears: which masking model is good enough? Which band model (ERB vs Bark)?
4. Track identity across hosts: GUIDs are per DAW; do we need a portable id?
5. Teaching data: how to make consent, review and deletion simple enough that people use them.
6. Should agents be able to talk to each other on the stream, or only through the human?

## 12. Changelog

- **Draft 0** (2026-10-16): first written version.
