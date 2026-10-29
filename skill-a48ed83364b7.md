<!-- ekson73/multi-agent-os:skills/opera-debrief/SKILL.md | https://github.com/ekson73/multi-agent-os/blob/HEAD/skills/opera-debrief/SKILL.md -->

---
name: opera-debrief
description: |
  Use when you want to deliver a session/work recap as a FAITHFUL, dosed NARRATIVE — a
  "summary as an opera": a short story-arc (acts) with measured humour, situational
  (never personal) wit, instigating-but-not-alarming drama, a call-to-action, the key
  insights, and a closing moral — for a HUMAN; or a structured consolidated payload for an
  AGENT. Triggers: "resumo da ópera", "summary as an opera", "narrate this session",
  "give me the story of what we did", "recap with a bit of flair", "debrief com pegada
  narrativa", "tell the tale of this work". It does NOT re-summarize — it CONSUMES an
  existing session map (from `postflight` P2-DEBRIEF / `morning-briefing --mode=recap`) and
  re-casts it through the named OPERA REGISTER, then runs a FAITHFULNESS gate (every beat
  traces to a real fact; no invented drama) + a TONE-SAFETY gate (no alarm; wit never aimed
  at a person). It is the narrative-warm register member of the content/recap family — a
  specialised, dosed sibling of `content-recast`. Cross-vendor AAIF.
triggers:
  - resumo da ópera
  - summary as an opera
  - opera-debrief
  - narrate this session
  - tell the story of what we did
  - recap with flair / with a light touch
  - debrief com pegada narrativa
  - narre como um caso (Sherlock·Watson·Moriarty)
version: 0.2.0
license: MIT
allowed-tools: Read, Write, Edit, Glob, Bash, Skill
metadata:
  version: "0.2.0"
  scope: AAIF cross-vendor
  family: content-lifecycle
  cross_link_slug: opera-debrief
  dogfood_status: pending-first-cycle
---

# Opera-Debrief

## Overview
Deliver a recap of a session, task, or PR **as a short opera** — a narrative arc that *acoolhe, informa e dá direção* (welcomes, informs, gives direction). Single responsibility = the **register transformation** of an *already-computed* session map into a dosed narrative; the summarising itself is **delegated** to the recap family (composition-over-inheritance — this skill never re-derives the facts). The differentiator vs. "make my summary fun" is two non-negotiable gates: a **faithfulness gate** (every dramatic beat traces to a real fact — no invented drama) and a **tone-safety gate** (instigating, *never alarming*; wit aimed at situations or at the agent itself, *never* at a person). A vivid-but-false or anxiety-inducing recap is worse than a plain one.

It is the **narrative-warm register** specialisation of `content-recast` (which re-targets *any* content for *any* audience). Opera-debrief fixes the audience to "a human who just finished the work (or an agent inheriting it)" and the register to "Opera", and adds the dosed-intensity dials + the no-terror clause.

## When to use
- Close a session/PR with a recap a human will actually *read* (and enjoy) — story-arc, light humour, a moral, a call-to-action.
- Turn a dry `postflight`/`morning-briefing` map into a human-warm narrative without losing a single fact.
- Hand a downstream **agent** a consolidated, structured digest of "the story so far" (register switches to machine-economy).

**When NOT to use**: you need the raw operational map, not a narrative → use `postflight` / `morning-briefing` directly. You need to re-target arbitrary content for an arbitrary audience → use the general `content-recast`. The work isn't done yet / there's no session map to recast → there's nothing faithful to narrate.

## §0 — BEING > Rules (foundational)
This skill serves the operator's intent. If any phase/gate obstructs delivering value NOW, skip it, log `Skipped <phase> — BEING > Rules`, and proceed. **Never invent a beat** to make the story better — the faithfulness gate is non-negotiable. **Never alarm** to make it gripping — the tone-safety gate is non-negotiable. Sensitive facts (secrets · production PII · irreversibles · cross-org publish) → escalate, never narrate around.

## Parameters
| Param | Default | Meaning |
|---|---|---|
| `<source>` (positional) | session map from `postflight`/`morning-briefing` (or prior context) | What to narrate. Empty → derive from the current session map; if none exists → print usage. |
| `--audience` | `human` | `human` (warm opera narrative) · `agent` (structured/JSON-RPC consolidated digest, humour off). |
| `--lang` | `pt-BR` | Output language (`pt-BR` · `en-US` · any). The narrative register adapts to the language's idiom. |
| `--intensity` | `dosed` | Global dial: `dosed` (measured — default) · `low` (mostly facts, a touch of warmth) · `off` (plain — degrades to a `postflight` debrief). |
| `--humour` | inherits `--intensity` | `med` · `low` · `off`. Situational/self-directed only. |
| `--drama` | inherits `--intensity` | `instigante` · `low` · `off`. **Never `terror`** — the dial has no alarming setting by design. |
| `--acts` | auto | Number of acts; auto-mapped from the session's phases/turning-points (cap 5 — anti-bloat). |
| `--lens` | `classic` | Tone lens: `classic` (story-arc, v0.1.0 default) · `deductive` (a *case* à la Holmes·Watson·Moriarty — intuitive-logic made visible, less technical/more human — see Lens below). |
| `--dry-run` | off | Emit the neutral session-map + the planned act structure + chosen dials only; no narration. |
| `--media` | `text` | Output medium. `text` = printed opera (**DEFAULT — never plays sound**). `audio-voice` = ALSO speak it aloud via `bin/speak.sh --play` + `skills/voice-director` Voice-Director (register-adapted, `--style narrador`). **Opt-in only** — audio NEVER auto-plays (the operator may be somewhere sound is unwelcome). |

**Argument parsing**: token(s) before the first `--` = `<source>`; `--key value`/`--flag` after = parameters. No positional → take the session map from `postflight`/`morning-briefing`/prior context.

## Pipeline (0 → 6)
0. **Intake** — parse `<source>` + params. Resolve dials from `--intensity` if unset. If `--audience=agent`, force `--humour=off --drama=low`.
1. **Acquire the map (DELEGATE — do NOT re-derive)** — obtain the session map from `postflight` P2-DEBRIEF or `morning-briefing --mode=recap` (objectives, done, in-flight, decisions, gaps, the turning-points). If absent, ask the upstream skill for it; never recompute facts from scratch. This is the faithfulness anchor.
2. **Distil → fact-ledger** — extract the atomic **facts + turning-points + the one decision/insight that mattered** into a neutral ledger. Each becomes a candidate beat. NEVER narrate from raw context.
3. **Compose the acts** — map the ledger onto a story-arc: **setup → rising tension (the turning-point/conflict) → resolution → moral**. Auto-pick act count (≤5) from the real turning-points. Each act = one real fact, dramatised *in framing only*.
4. **Apply the OPERA REGISTER** (see profile below) at the chosen dials — humour (situational/self), wit (never personal), drama (instigating, no-terror), plus the three mandatory beats: **insights**, **call-to-action**, **moral-da-ópera**.
5. **Dual gate (MANDATORY)** —
   - **Faithfulness**: every beat traces to a fact in the ledger (step 2). Strip/repair any invented drama, inflated magnitude, or fabricated stakes. Emit an **Information-Loss Note** (what was compressed/omitted) so the reader knows the boundaries.
   - **Tone-safety**: no alarming framing (no "catastrophe/disaster/terror"); wit targets situations or the agent itself, never a person; humour stays *comedido* (signal ≫ noise — it must carry attention to the fact, never bury it). Drop anything that fails.
6. **Render by audience** — `human`: the opera (acts + insights + CTA + moral + loss-note). `agent`: a structured/JSON-RPC consolidated digest (acts→events, moral→verdict, CTA→next_action) with humour stripped — see Output.

## The OPERA REGISTER PROFILE (the SSOT this skill owns)
A named **human-warmth register** (audience-design, Bell 1984) with a fixed recipe and explicit dials:

| Ingredient | Role | Dial | Default | Guardrail |
|---|---|---|---|---|
| **Story-arc (acts)** | structure: setup → tension → resolution → moral | `--acts` | auto (≤5) | each act = one real fact |
| **Measured humour** | carries attention to the point | `--humour` | dosed/med | situational or self-directed; signal ≫ noise |
| **Wit / light jab** | keeps it alive ("sarcasm sem ofender") | (humour) | low | aimed at *situations* or *the agent*, **never a person** |
| **Instigating drama** | makes the turning-point land | `--drama` | instigante | **no terror / no alarm** — by design the dial cannot frighten |
| **Insights** | what we learned | mandatory | on | grounded in real facts |
| **Call-to-action** | gives direction (next step) | mandatory | on | one concrete next move |
| **Moral-da-ópera** | the one-line takeaway | mandatory | on | the through-line of the whole arc |

> The profile is a **default recipe** — the agent may blend/override dials with rationale (cognitive-adaptation-freedom), as long as both gates hold. `--intensity=off` collapses the register to a plain `postflight` debrief (graceful degradation).

## Lens: Deductive (Holmes · Watson · Moriarty) — `--lens=deductive`
A sophistication tuning of the Opera register: stage the recap as a **case**, not a report — *less technical, more human, intuitive-logic made visible*. The arc remaps to **the scene → the clues → the deduction → the reveal → the lesson**.

| Voice | Role |
|---|---|
| **Watson** (narrator) | Warm, human, the audience's surrogate: asks the obvious question, marvels, and **translates every technical term into a plain image** — at most **one** light technical anchor per act, glossed. He is the "less technical, more human" voice. |
| **Holmes** (deduction) | Makes the intuitive logic *visible*: observe the clues (real facts) → infer → the inevitable reveal (*"once you eliminate the impossible, whatever remains, however improbable, is the truth"*). Sophistication = clarity made elegant, never jargon. |
| **Moriarty** (the adversary) | The **problem / false-assumption personified** — never a person, never alarming. Gives the case its tension; defeated by deduction, not feared. |

**Gates (unchanged, re-expressed for this lens):** every *clue* traces to a real fact (faithfulness — no invented clue); Moriarty is always a *problem*, never a human, and never escalates to terror (tone-safety). **De-jargon is mandatory** under this lens: raw tokens (`mergeStateStatus`, JSON-RPC, file paths) become Watson's plain images.

**The lens prompt (drop-in):**
> Stage the recap as a *case*. WATSON narrates (human, the audience's surrogate; translate every technical term into an image, ≤1 light technical anchor per act, glossed). HOLMES deduces (observe the real clues → infer → reveal the inevitable). MORIARTY is the problem personified (never a person, never alarm). Arc: scene → clues → deduction → reveal → lesson. Sophistication = elegant clarity, not jargon.

Composes freely with the dials (`--humour`/`--drama`/`--intensity`); `--lens=classic` keeps the v0.1.0 story-arc.

## Composition map (DRY — zero re-summarisation)
| Need | Delegated to |
|---|---|
| The session map (facts) | `skills/postflight` P2-DEBRIEF · `skills/morning-briefing --mode=recap` |
| General audience re-targeting (non-Opera registers) | `skills/content-recast` (Opera is one named register; that skill owns the rest) |
| Render to slides/PDF/podcast | per `content-recast` Composition map (Gamma · make-pdf · NotebookLM) — never rebuilt here |
| Speak the recap aloud (`--media audio-voice`, OPT-IN) | `bin/speak.sh --play` via `skills/voice-director` Voice-Director (`--style narrador`, register-adapted) — never auto-plays; default output stays text |

## Faithfulness + Tone-safety gates (the differentiators — non-negotiable)
1. Every beat traces to the fact-ledger (step 2). 2. No invented facts, stakes, numbers, or quotes. 3. No magnitude/causality inflation for drama. 4. Load-bearing caveats survive or appear in the Information-Loss Note. 5. No alarming framing (no terror). 6. Wit never targets a person. 7. Humour is *dosed* — it must serve the fact, never obscure it (transparency: warmth must NOT drop a risk/caveat). Research basis: factuality in simplification (Devaraj et al., ACL 2022) · InfoLossQA (arXiv 2401.16475) · audience-design (Bell 1984).

## Output format
**`--audience=human`:**
```
🎭 <Title> — resumo da ópera

Ato I — <setup, grounded in a real fact>
Ato II — <the turning-point / tension>
Ato … — <resolution>

🎯 Moral da ópera: <the one-line through-line>
💡 Insights: <1-3 grounded takeaways>
👉 Próximo passo: <one concrete CTA>
ℹ️ Nota de fidelidade: <what was compressed/omitted; "nada inventado">
```
**`--audience=agent`** (JSON-RPC consolidated, humour off):
```json
{"jsonrpc":"2.0","method":"recap.opera","params":{
  "title":"...","acts":[{"act":1,"fact":"...","frame":"..."}],
  "moral":"...","insights":["..."],"next_action":"...","info_loss":"...","faithful":true}}
```

## Anti-patterns (do NOT)
- ❌ Re-summarise from raw context (skip the delegated map) → faithfulness drift + duplicates `postflight`.
- ❌ Invent drama/stakes to make it gripping (AI slop) → violates the faithfulness gate.
- ❌ Alarm the reader ("disaster/catastrophe/terror") to hold attention → violates tone-safety.
- ❌ Aim wit at a person → violates tone-safety ("sarcasm sem ofender").
- ❌ Let humour bury a fact/caveat → violates dosed-transparency.
- ❌ Rebuild a renderer or a summariser → use the Composition map.
- ❌ Narrate work that isn't done / has no map → nothing faithful to tell.

## §Quality Tests (self-dogfood — 6/6)
1. **Self-Application** — this SKILL.md is itself the faithful, dosed product the skill describes (the genesis session was recapped "as an opera" and the operator asked to make that reusable). ✅
2. **Non-Contradiction** — composes (not duplicates) `postflight`/`morning-briefing`/`content-recast`; the Opera register is one named register, not a rival summariser. ✅
3. **Survival** — applied to itself it advocates faithful, non-alarming narration; survives. ✅
4. **Bounded-Responsibility** — `--dry-run` · register-transform only (no re-summarisation) · acts ≤5 · two hard gates · `--intensity=off` graceful degradation · DUED sunset. ✅
5. **Explicit-Exception** — §0 BEING>Rules + HUMAN_DOMAIN escalation + cognitive-adaptation-freedom on dials. ✅
6. **Utility-Sunset** — §DUED below. ✅

## §DUED Sunset (qualitative, not counter-based)
Deprecate when ANY: `content-recast` absorbs "Opera" as a first-class register (E6) · the host provides a native faithful narrative-recap primitive (E1) · operator retraction (E4) · ≥3 false-positives where a gate misfires (E5). Dormant-by-design otherwise.

## Related Multi-Agent OS Artifacts
- `skills/content-recast/SKILL.md` — the **general** register/audience re-targeting engine; Opera-Debrief is its **narrative-warm register specialisation** (fixed audience + Opera register + dosed dials + no-terror clause). Shares the faithfulness-gate lineage.
- `skills/postflight/SKILL.md` — **P2-DEBRIEF** produces the session map Opera-Debrief consumes (DRY: facts in, narrative out). Natural chain: `postflight → opera-debrief`.
- `skills/morning-briefing/SKILL.md` — its 7-section recap (`--mode=recap`) is an alternative fact-source for the narrative.
- `skills/preflight/SKILL.md` — start-of-session counterpart of `postflight`; bounds the session whose story this skill tells.

## §Refs
- Audience-design / register: Bell, A. (1984), *Language style as audience design*.
- Faithfulness: Devaraj et al., ACL 2022 (`2022.acl-long.506`) · InfoLossQA (arXiv 2401.16475).
- Frameworks: story-arc / three-act structure · the "distil-first, then re-cast" pattern (shared with `content-recast`).
- Cross-link slug: `[[opera-debrief]]`.

## Changelog
| Version | Date | Change |
|---|---|---|
| 0.1.0 | 2026-06-18 | Bootstrap. Narrative-warm register specialisation of `content-recast`: the **Opera register profile** SSOT (story-arc + dosed humour + situational wit + instigating-no-terror drama + insights + CTA + moral) with explicit intensity dials, a faithfulness gate (no invented drama + info-loss note) and a tone-safety gate (no alarm + wit never personal). Composes `postflight`/`morning-briefing` for facts (zero re-summarisation); dual-register output (human opera / agent JSON-RPC). Vendor-neutral, MIT, family-aware. |
| 0.2.0 | 2026-06-18 | Add **`--lens=deductive`** (Holmes·Watson·Moriarty) — a sophistication tuning: recap as a *case* (scene→clues→deduction→reveal→lesson), less-technical/more-human, intuitive-logic made visible. Watson narrates + de-jargons (≤1 technical anchor/act, glossed); Holmes deduces; Moriarty = the problem personified (never a person, never terror). Reuses both gates unchanged; `--lens=classic` preserves v0.1.0. Origin: operator refinement (debate→converge with the cast as the debate participants). |
