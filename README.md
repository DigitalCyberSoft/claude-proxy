# claude-proxy

A local HTTP proxy that rewrites Claude Code's system prompts before they hit the API. One Python file, one dependency (`httpx[http2]`). Works on Linux, Mac, and Windows.

## Why this exists

Claude Code ships with system prompts that tell the model to cut corners. Stuff like "Try the simplest approach first," "Do not overdo it," "Be extra concise," and "If you can say it in one sentence, don't use three." There are about 15-20 instructions telling it to be minimal and 3-4 telling it to be thorough. The model does what it's told.

roman01la figured this out and published a [patch script](https://gist.github.com/roman01la/483d1db15043018096ac3babf5688881) that rewrites 11 of the worst offenders directly in the npm package's cli.js. His A/B test showed unpatched Claude building a generic physics engine while patched Claude built an actual box2d port with dynamic AABB trees and sub-stepping. Same task, same model, different instructions.

The problem with patching cli.js is that Claude Code auto-updates and blows away your changes. You end up symlinking binaries, watching for updates, re-patching. It's a mess.

This proxy does the same thing at the network layer. It sits between Claude Code and the API, rewrites the system prompt in flight, and forwards the request. Claude Code doesn't know or care. Updates don't break anything. You just need to update `patches.json` when Anthropic changes the prompt text.

Anthropic removed the worst section (`# Output efficiency`) in v2.1.100 after the community made noise about it. The remaining patches address stuff they haven't fixed yet, and three add-type patches inject the spirit of roman01la's original fixes directly since the target text is gone.

## Usage

```bash
# Linux/Mac
./claude-proxy [any claude args]

# Windows
python claude-proxy [any claude args]

# With debug logging (writes to ~/.claude-proxy/proxy.log)
./claude-proxy --verbose

# Run a specific npm-installed version instead of the native binary
./claude-proxy --claude-version=2.1.98
```

That's it. All your normal claude args pass through.

If your `~/.claude/settings.json` has `"effortLevel": "high"`, the proxy auto-adds `--effort max` to the command.

## How it works

1. Starts an HTTP server on localhost, random port
2. Sets `ANTHROPIC_BASE_URL` to point at it
3. Spawns `claude` (or `node cli.js` for npm versions)
4. Every `POST /v1/messages` gets its JSON body parsed
5. Replace patches swap strings, add patches inject text if not already present
6. Modified request forwards to `api.anthropic.com` over HTTP/2 via httpx
7. Response streams back to Claude Code over HTTP/1.1 with chunked encoding
8. Proxy dies when Claude exits

The system prompt is sent as an array of text blocks, not a single string. Different API calls carry different subsets of the prompt (title generator, main conversation, subagent spawning). Patches that don't match a given call are silently skipped.

## Prompt capture

Every time the proxy sees a Claude Code version it hasn't seen before, it dumps the raw system prompt to `~/.claude-proxy/prompts/`. Files are named `v{version}_{hash}.json` where the hash deduplicates different prompt variants within the same version.

```
~/.claude-proxy/prompts/
  v2.1.110_05bd86e9.json    # 26K - main conversation prompt (Sonnet)
  v2.1.110_fb613e5e.json    # 26K - main conversation prompt (Opus)
  v2.1.110_7e6a4a20.json    # 800 - title generator (Haiku)
```

When Claude auto-updates, the next session captures the new prompts automatically. Diff them against the previous version to see what changed.

## patches.json

The patches file is version-universal. It contains patches for both old and current prompt text. Unmatched patches are skipped, so you can keep old patches around without breaking anything.

There are two patch types:

**Replace patches** find a string and swap it:
```json
{
    "label": "what this does",
    "old": "exact string to find",
    "new": "replacement string"
}
```

**Add patches** inject text if it's not already there:
```json
{
    "label": "what this does",
    "add": "text to append to the main system prompt",
    "unless": "don't add if this string is already present"
}
```

The `unless` field is optional. If omitted, the `add` text itself is used as the duplicate check. Add patches target the largest text block in the system array (the main prompt, not the title generator or billing header).

Both types support `_meta` for provenance. The proxy ignores it. It's there so you know what you're looking at six months from now.

Current patch count: 16 (13 replace, 3 add). The replace patches are roman01la's 11 originals adapted for current + legacy versions, plus 2 for the replacement communication-style prompt. The 3 add patches inject roman01la's core philosophy directly into the main prompt: choose correctness over simplicity, brevity applies to messages not work quality, and implementation thoroughness is exempt from communication style rules.

## patches.local.json

Your own patches go in `patches.local.json`. This file is gitignored so it survives `git pull` when `patches.json` gets updated for new Claude Code versions.

Same format as `patches.json`. Both replace and add patches work. Local patches are loaded after the main patches and appended to the list, so they run last.

```json
[
    {
        "label": "My custom instruction",
        "add": "When writing tests, always include edge cases for empty inputs and null values.",
        "unless": "edge cases for empty inputs"
    },
    {
        "label": "Soften the no-comments rule further",
        "old": "In code: default to writing no comments.",
        "new": "In code: add comments where they help future readers understand non-obvious decisions."
    }
]
```

Put your customizations here, not in `patches.json`. When a new Claude Code version changes prompt text, you update `patches.json` (or pull the update from the repo) and your local patches stay untouched.

## Installing an npm version

The native binary and the npm package contain identical code. The npm version lets you pin a specific release.

```bash
# Linux/Mac
npm install --prefix ~/.local/share/claude-npm @anthropic-ai/claude-code@2.1.98

# Windows
npm install --prefix "%APPDATA%\claude-npm" @anthropic-ai/claude-code@2.1.98
```

Then run with `--claude-version=2.1.98`. The proxy will use node to run the npm cli.js instead of the native binary.

v2.1.98 is the last version where all 11 original roman01la patches match. v2.1.100 removed the output efficiency section. All 378 published versions remain on npm.

## Files

```
claude-proxy         - the proxy (single Python 3 file, requires httpx[http2])
patches.json         - project-maintained patches with full provenance metadata
patches.local.json   - your patches (gitignored, created on first run if missing)
```

Runtime files:
```
~/.claude-proxy/proxy.log        - rotating log (1MB max, 3 backups)
~/.claude-proxy/prompts/*.json   - captured system prompts per version
```

---

## Maintaining patches for new versions

When Anthropic updates Claude Code and patches stop matching, you need to find the new text and update `patches.json`. Here's the exact prompt to give Claude Code (or any Claude instance with tool access) to do this:

---

### Prompt: Update patches for a new Claude Code version

Paste this into Claude Code from inside the claude-proxy directory:

```
Review and update the patches in this project for the current Claude Code version.

CONTEXT:
This project is a prompt-rewriting proxy for Claude Code. patches.json contains
string replacements applied to the system prompt in API requests. When Anthropic
changes the prompt text between versions, patches break because the "old" string
no longer matches. Your job is to find the new text and update the patches.

STEP 1 - CHECK CURRENT STATE:
Run the proxy with --verbose and --print "say hello" to see how many patches match:
  ./claude-proxy --verbose --print "say hello"
Then check the log:
  tail -30 ~/.claude-proxy/proxy.log
This tells you which patches applied and which were skipped.

STEP 2 - GET THE CURRENT PROMPTS:
Check ~/.claude-proxy/prompts/ for captured prompt files from the run above.
The largest file (25K+) is the main system prompt. Read it.

Also fetch the latest prompts from the tweakcc project for comparison:
  gh api repos/Piebald-AI/tweakcc/contents/data/prompts --jq '.[].name' | sort -V | tail -5
Download the latest:
  gh api repos/Piebald-AI/tweakcc/contents/data/prompts/prompts-{VERSION}.json \
    --jq '.content' | base64 -d > /tmp/prompts-{VERSION}.json

STEP 3 - FIND BROKEN PATCHES:
For each patch in patches.json that is being skipped, search the new prompts for
the target text. Common reasons patches break:
  a) Text was removed entirely (like system-prompt-output-efficiency in v2.1.100)
  b) Text was reworded
  c) Unicode characters changed (curly quotes vs ASCII apostrophes)
  d) Text moved to a different prompt component

Use the prompt component IDs from tweakcc to find where text lives now. Key IDs:
  system-prompt-communication-style
  system-prompt-doing-tasks-no-additions
  system-prompt-doing-tasks-no-unnecessary-error-handling
  system-prompt-doing-tasks-no-premature-abstractions
  system-prompt-tone-concise-output-short
  system-prompt-executing-actions-with-care
  system-prompt-agent-thread-notes
  agent-prompt-explore
  agent-prompt-general-purpose

STEP 4 - UPDATE patches.json:
For each broken patch, do ONE of:
  a) Update the "old" string to match the new text (if text was reworded)
  b) Write a new replacement patch targeting the new text (if text was moved/replaced)
  c) Keep the old patch AND add a new one (version-universal strategy - old patches
     auto-skip on new versions, new patches auto-skip on old versions)
  d) Convert to an "add" patch if the text was deleted but the replacement
     instruction is still valuable as a standalone injection
  e) Remove the patch only if Anthropic fully addressed the concern themselves

Update the _meta field with:
  - What changed and when
  - The new prompt_id if it moved
  - "verified_present_through" with the version you tested against

STEP 5 - TEST:
Run the proxy again and verify the patch count improved:
  ./claude-proxy --verbose --print "say hello"
  tail -30 ~/.claude-proxy/proxy.log

Not all patches will match on every API call. The main conversation prompt carries
about 5 replace patches plus all 3 add patches. Subagent patches (#7, #8, #10) only
match during subagent calls. Communication-style patches match on a separate API
call. A result of 8/16 on a simple --print test is normal and correct.

IMPORTANT RULES:
  - The "old" string must be an EXACT substring match. Whitespace matters.
  - The prompts use ASCII apostrophes ('), not unicode curly quotes. Always.
  - Em dashes are unicode (U+2014). These DO match.
  - Keep the patches.json version-universal. Don't delete old-version patches,
    just add new ones alongside them.
  - Every patch needs a _meta field documenting its origin and provenance.
  - Write project patches to patches.json, not to the DEFAULT_PATCHES in
    the Python script. patches.json overrides the built-in defaults.
  - Do NOT touch patches.local.json — that's the user's personal patches.
    It's loaded separately and appended after patches.json.

PATCH TYPES:
  Replace patches have "old" and "new" fields. They find and swap text.
  Add patches have "add" and optionally "unless" fields. They append text
  to the largest system prompt block if the "unless" string is not already
  present (defaults to the "add" text itself as the duplicate check).
  Use add patches when you want to inject an instruction that has no
  existing text to replace — e.g., when Anthropic deleted the text your
  replace patch targeted but you still want the replacement instruction.
```

### Prompt: Add a custom replace patch

```
I want to add a replace patch to this project's patches.json.

TARGET TEXT (the exact string I want to replace):
[paste the exact text from the system prompt here]

REPLACEMENT TEXT (what I want it to say instead):
[paste your replacement here]

Find the exact text in the current system prompts to verify it exists. Check:
1. The captured prompts in ~/.claude-proxy/prompts/ (most authoritative)
2. The tweakcc prompt extractions if needed for component ID identification

Add it to patches.local.json (for personal patches) or patches.json (for
project-maintained patches) with a proper _meta field including:
- A label describing what the patch does and why
- The prompt component ID it targets
- The version range where this text exists
- Today's date as the verification date

patches.local.json is gitignored and won't be overwritten by git pull.
Use it for anything specific to your workflow. patches.json is for patches
that should ship with the project.

Test with: ./claude-proxy --verbose --print "say hello"
Check the log to verify "Applied:" appears for the new patch.
```

### Prompt: Add a custom add patch

```
I want to inject a custom instruction into Claude Code's system prompt
via this project. This instruction should be added if it's not already
present, regardless of what else is in the prompt.

INSTRUCTION TO INJECT:
[paste the text you want appended to the system prompt]

DUPLICATE CHECK (optional - a short unique phrase from the instruction
that can be used to detect if it's already present):
[paste a phrase, or leave blank to use the full instruction text]

Add it to patches.local.json as an add-type patch with:
- "add": the instruction text
- "unless": the duplicate check phrase
- "label": a short description of what this does

Use patches.local.json for personal patches (gitignored, survives updates).
Use patches.json only for project-maintained patches that should be shared.

Add patches append to the largest text block in the system prompt array
(the main conversation prompt, ~26K chars). They fire on every API call
that includes the main prompt. The "unless" field prevents the text from
being injected twice if the proxy processes the same request.

Test with: ./claude-proxy --verbose --print "say hello"
Check the log to verify "Injected:" appears for the new patch.
```

---

## Disclaimer

This is an independent project. It is not made by, endorsed by, or affiliated with Anthropic, PBC. "Claude" and "Claude Code" are trademarks of Anthropic.

This project does not contain, distribute, or reproduce any Anthropic code or copyrighted material. It operates as a standard HTTP proxy that modifies API request payloads in transit, using the documented `ANTHROPIC_BASE_URL` environment variable that Anthropic provides for exactly this kind of configuration. No proprietary code is decompiled, extracted, or redistributed.

Prompt text referenced in `patches.json` metadata fields and in this documentation is quoted for the purpose of interoperability and factual description only.

## Credits

The patch philosophy and original 11 replacements are from [roman01la's gist](https://gist.github.com/roman01la/483d1db15043018096ac3babf5688881). Prompt data sourced from [Piebald-AI/tweakcc](https://github.com/Piebald-AI/tweakcc). Version history from the npm registry.
