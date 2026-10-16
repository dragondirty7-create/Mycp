# MyCP

Inspired by MIDI and MCP.

**MyCP is a free, open protocol for AI agents to work in a music studio in real time.**
Think MIDI, but the things on the cable are AI collaborators: they hear the session's
events as they happen, share the musical clock, each listen on their own channel, and only
touch what a human has allowed.

> **Status: draft 0.** This is an idea written down carefully, not a finished standard and
> not yet implemented anywhere. Everything here can change. See [`SPEC.md`](SPEC.md).

## Why

MIDI (1983) worked because a handful of rival companies agreed on one small, boring,
real-time language and gave it away. Forty-plus years later it still connects almost
everything.

AI tools today mostly talk to software like an office: ask a question, wait, get an answer.
That's the right shape for "summarize my project", and [MCP](https://modelcontextprotocol.io)
already does it well. It's the wrong shape for a live session, where the useful moments are
*events*: the take that just finished, the fader you're riding at bar 32, the vocal that
started getting buried.

MyCP is for that live part:

- **Events, not questions.** The studio announces what happens; agents don't have to poll.
- **Channels.** Several agents can share a session, each listening to what's relevant to it.
- **A shared clock.** Bars, beats and tempo, so "bar 33" means the same moment to everyone.
- **Tiny messages.** Built on [OSC](https://opensoundcontrol.stanford.edu/), which DAWs and
  control surfaces already speak.
- **Ears.** A standard way to describe what a track *sounds* like in terms of how people
  hear (loudness, critical bands, masking), not just raw meters.
- **Teaching.** A human's moves, their spoken reasons, and the before/after sound, aligned
  in time, so a person can teach an agent to mix by mixing.
- **Humans in charge.** Agents start read-only. Control is granted per session, and every
  host must honor *panic*, which takes all control away from all agents at once.

## How it fits with MCP

MyCP doesn't replace MCP. An agent connects to a studio host through MCP as usual, then the
host opens a MyCP stream for the live part. MCP is the front door; MyCP is the band room.

```
DAW (e.g. REAPER) ⇄ OSC ⇄ studio host / conductor ⇄ MyCP events ⇄ agents
                                     ⇅
                                    MCP (connect, ask, permissions)
```

The first host being built toward this is Bepy, a session assistant for REAPER whose
read-only MCP connection already exists as a draft.

## License

[Apache-2.0](LICENSE). Free to use, implement, extend and ship, commercially or not.
If MyCP turns out to be useful, it should belong to everyone, the way MIDI does.

## Contributing

It's early. Ideas, objections and "this would never work because…" are all welcome as
issues. See [`CONTRIBUTING.md`](CONTRIBUTING.md).
