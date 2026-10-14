---
name: chatgpt-imagegen
description: >
  Generate PNG images with Codex's built-in $imagegen while signed in with
  ChatGPT (no API key). Takes a direct prompt, or a brief to turn into concept
  prompts, plus optional local reference images; runs one image or a bounded
  parallel batch; saves to <project>/generated-images/*.png. Trigger:
  "generate an image", "make N concepts", "imagegen batch", $chatgpt-imagegen.
---

# ChatGPT imagegen

Thin wrapper around the built-in `$imagegen`. Do not call the Images API, do
not ask for an `OPENAI_API_KEY`, do not install anything.

## Install (once, for humans)

```sh
mkdir -p ~/.codex/skills && cp -R codex-skills/chatgpt-imagegen ~/.codex/skills/
codex login            # choose "Sign in with ChatGPT"
```

## 1. Check sign-in

Run `codex login status`. Continue only if it says it is logged in using
ChatGPT. Otherwise stop and tell the user to run `codex login` and choose
**Sign in with ChatGPT**. Image generation counts against their ChatGPT plan.

## 2. Build the prompt list

- **Direct**: the user gave the exact prompt. Use it verbatim, one job.
- **Delegated**: the user gave a brief ("3 concepts for a spring sale
  banner"). Write one prompt per concept, each a different idea (subject,
  composition, style, palette, text-in-image if any, aspect ratio). Show the
  list and the reason behind each concept in one line. Wait for a go only if
  the user asked to review first.

**Local image inputs**: accept only paths that exist and end in
`.png .jpg .jpeg .webp`. Name any that don't and drop them, don't guess. Pass
the valid ones to every job as references (edit / style / product shot).

## 3. Generate

Output dir: `<project>/generated-images/` (`<project>` = git root, or cwd if
not a repo). Create it if missing. File name: `<slug-of-concept>.png`; if it
exists, append `-2`, `-3`, ... **Never overwrite an existing file.**

- **Single** (1 prompt): call `$imagegen` here with the prompt and reference
  images, and tell it to save to the target path.
- **Batch** (2+ prompts): hard cap 8 images per request, at most 3 running at
  once. Run one `codex exec` per prompt, in a fresh shell each, from the
  project root:

  ```sh
  # jobs.tsv: <target-path>\t<prompt>, one job per line
  export REFS="-i ref1.png -i ref2.png"   # or REFS="" if no local inputs
  # ponytail: fixed concurrency of 3; raise -P only if the plan's rate limit allows
  tr '\n' '\0' < jobs.tsv | xargs -0 -P 3 -I{} sh -c \
    'out=$(printf %s "$1" | cut -f1); p=$(printf %s "$1" | cut -f2-);
     codex exec --full-auto $REFS "\$imagegen $p. Save the PNG to $out." \
       || echo "FAILED $out"' _ {}
  ```

  Ask for more than 8 → do the first 8, say so, and offer the rest as a
  follow-up batch.

## 4. Verify and report

For every job, check the target `.png` exists and is not empty. Report a
table: file path · prompt (shortened) · ok/failed. Re-run failed jobs once on
their own. Don't retry in a loop. If one fails again, say so and include the error.
