<!-- ARCASSystems/FounderOS:skills/content-repurposer/SKILL.md | https://github.com/ARCASSystems/FounderOS/blob/HEAD/skills/content-repurposer/SKILL.md -->

---
name: content-repurposer
description: >
  Reformat one piece of content across many channels. Trigger on "repurpose this", "turn this into", "adapt this for", "convert this to", "make versions of this for", or any variation. Also fires when the user shares content and wants it reformatted for different platforms. Works across LinkedIn, YouTube, Instagram, email newsletters, website copy, Twitter/X, and internal docs. Reads `core/voice-profile.yml` and applies the founder's voice.
why: "Multiplies one piece of thinking across multiple channels without re-writing from scratch for each - saves hours on content execution."
enhance: "Fill core/voice-profile.yml before repurposing - without it the output adapts format but loses your voice, which is what makes the same insight work across platforms."
allowed-tools: ["Read", "Bash"]
mcp_requirements: []
---

# Content Repurposer

Runs on: reasoning - reads your files and reasons; any capable agent can run this.

Apply the voice and writing rules from `core/identity.md`, `rules/writing-style.md`, and `core/voice-profile.yml` (via the `your-voice` skill) to all output. Keep the founder's voice consistent across every channel.

## Voice routing (operator or brand?)

Before any gate or draft, apply the routing rules in `skills/your-voice/SKILL.md` "Voice routing - operator or brand?" section.

- Default to operator voice from `core/voice-profile.yml`.
- If the user named a brand they run, or if the source content is brand-owned, use brand voice from `brands/<slug>/voice.yml`.
- For multi-channel repurposing, the SAME voice applies to every channel adaptation. Do not mix voices across the output.

If `brands/` does not exist or has no entries, this section is a no-op.

## Voice profile readiness

If using operator voice, run: `python scripts/check-voice-ready.py`

If using brand voice, run: `python scripts/check-brand-voice-ready.py --brand <slug>`

If exit code is 1, read the output line and surface it to the user verbatim. Do not produce any draft. Stop.

If the user explicitly chooses to proceed with defaults after seeing that message, repurpose the content using the universal anti-AI baseline from `your-voice` and clearly label that the voice profile was not applied. Do not pretend the outputs are voice-coupled.

Then read the chosen voice profile so the rest of this skill can apply it.

After producing a draft and before returning it, run the anti-examples filter:

1. Read the `anti_examples.pairs` block in `core/voice-profile.yml`.
2. For each line in your draft, scan for matches against any `bad:` pattern (literal substrings, structural markers like negation-contrast, or rule-of-three lists).
3. If a line matches, rewrite it using the `good:` pattern as the model and the `rule:` line as the constraint.
4. Also reject any line that uses an `aesthetic_crimes` phrase or a `red_flags` pattern.
5. Return the cleaned draft.

Do not surface this filter to the user as a separate step. The user sees only the cleaned draft.

## Runtime context

Before drafting, read `brain/.snapshot.md` if it exists. Use the open-flags block to avoid topics that contradict current operator stance. Use the must-do block to lean the draft toward what the operator is actively working on. Use the voice and brand blocks (if present) to set tone. If `brain/.snapshot.md` does not exist, proceed without it - the snapshot is optional context, not a hard prerequisite.

Find the ONE core insight before adapting. A repurpose has one argument across every channel, not a different argument per platform.

## Channel selection (brand-aware)

If brand voice is loaded, read `brands/<slug>/positioning.yml`. Filter the channel list:

- Include channels in `positioning.channels.primary` and `positioning.channels.secondary`.
- Exclude channels in `positioning.channels.off_limits` - never suggest these.
- If the user explicitly asks for a channel that is off-limits, surface the conflict: "Positioning says `<channel>` is off-limits for `<brand>`. Want me to override for this piece, or skip that channel?"

If operator voice is loaded (no brand), include all channels by default but lean toward channels the operator has used recently per `brain/log.md` if that signal is available.

## Channel Specifications

### LinkedIn Post
- 100-300 words, line breaks between thoughts, hook first, no hashtags, arrows for lists

### YouTube Script
- 3-5 minutes, conversational, Hook/Story/Framework/Takeaway, [B-ROLL] markers

### Instagram Caption
- 50-150 words, scroll-stopper first line, short lines, no hashtags

### Email Newsletter
- 200-500 words, lead with the point, one insight, casual direct tone

### Twitter/X Thread
- 3-7 tweets, first tweet standalone value, one idea per tweet

### Website Copy
- Clear hierarchy, scannable, SEO-aware not SEO-driven

### Internal Doc / SOP
- Clear, direct, industry terms when they serve clarity, anti-AI check relaxed

## Repurposing Rules

1. **Don't just shorten.** Each platform has a different angle.
2. **Preserve the core insight.** It should survive every adaptation.
3. **Match the energy.** LinkedIn = thought leadership. Instagram = personal. YouTube = teaching. Email = intimate.
4. **Don't repeat yourself across channels.** Find a different angle for each.
5. **Apply the founder's voice to every output.** Run anti-AI self-check on public content.

## Output Format

```
SOURCE: [Brief description of original content]
CORE INSIGHT: [One sentence]
---
LINKEDIN POST
[content]
---
YOUTUBE SCRIPT
[content]
---
INSTAGRAM CAPTION
[content]
```
