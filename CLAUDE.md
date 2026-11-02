# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

There is one artifact here: `pr-agent.json`, an exported **n8n workflow** called "Bitbucket PR - AI Inline Review". A Bitbucket **Data Center** PR webhook triggers it. It reviews the PR diff with an LLM (Qwen `qwen3-coder` through an OpenAI-compatible endpoint) and posts inline comments and tasks back through the Bitbucket REST API 1.0.

The repo has no build, lint or test tooling. You work by editing the JSON, then importing it into n8n (Workflows → Import from File) and triggering the webhook (`POST /webhook/bitbucket-pr-review`) with a Bitbucket PR event payload.

## Editing the workflow

- All logic lives in `nodes[].parameters.jsCode` (Code nodes) and in the agent prompts (`text` / `options.systemMessage`). Each is a single JSON-escaped string, so hand edits to `\n` and `\"` break easily. To read or change one, load and dump it with Python. Use `encoding='utf-8'` and set `PYTHONIOENCODING=utf-8`: the code contains emoji, which crash Windows cp1252 output.
  ```
  PYTHONIOENCODING=utf-8 python -c "import json;d=json.load(open('pr-agent.json',encoding='utf-8'));print(next(n for n in d['nodes'] if n['name']=='Resolve Anchors')['parameters']['jsCode'])"
  ```
- `connections` is keyed by node **name**, and Code nodes refer to other nodes with `$('Node Name')`. If you rename a node, update both places.
- In Code nodes, use only **literal** `$('Node Name')` calls, never a dynamic `$(name)`. A dynamic call makes the n8n task runner ship the entire execution data, including every LLM sub-node run, and the node hangs. The comment in "Resolve Anchors" documents this.
- Credentials are placeholders (`REPLACE_BB` = Bitbucket Basic Auth, `REPLACE_LLM` = Qwen OpenAI-compatible). They are bound after import.

## Pipeline (follow `connections`)

1. **Config** (Set node) holds every tunable setting: `BB_BASE_URL`, the chunking limits, `MIN_CONFIDENCE`, `VERIFY_FAIL_MODE`, `MAX_COMMENTS`, `TASK_SEVERITIES`, `UNANCHORED_MODE`, `LINE_TOLERANCE`, `IGNORE_PATH_REGEX`, and others. It also extracts PROJECT/REPO/PR_ID/hashes from the webhook body. Code nodes read it as `$('Config').first().json` and parse values with a `num(v, default)` helper, so their defaults mirror Config.
2. **Get Repo Rules** fetches `.ai-review.md` (`REPO_RULES_PATH`) from the PR *source* commit. A 404 means there are no rules, and the node never errors. **Get PR Diff** sends its error output to **Format Diff Failure** → **Post Failure Notice**.
3. **List Changed Files** emits one item per reviewable path. **Get Changed File** then fetches each file's full content at `FROM_HASH` (one request per item, `neverError`). **Build Review Context** pairs those responses with the listed paths **by index**.
4. **Build Review Context** reads the diff from `$('Get PR Diff')` and splits the files into chunks, one n8n item per chunk. Each chunk carries `files[]` with global ids `F1..Fn`, `lineIndex`, `annotatedDiff` and `repoRules`/`hasRules`. `lineIndex` holds the commentable lines, which are only the lines inside diff hunks. Despite its name, `annotatedDiff` holds the **whole new file**: every line is numbered, `+` marks added lines, and removed `-` lines appear in place. A file falls back to the plain diff hunks, marked `diff only` in its header, when the fetch fails or the file exceeds `MAX_FILE_CHARS`.
5. **AI Code Reviewer** is a Basic LLM Chain (`chainLlm`, no tools) that runs per chunk with the Qwen model and an auto-fixing structured output parser ("Issues Schema"). **Rules Review Input** re-emits the chunks, because the chain output replaced them. **AI Rules Reviewer** has its own parallel copies of the model and parser, and runs only when `hasRules` is true.
6. **Resolve Anchors** merges the issues from both reviewers (`source: general | rules`). The LLM's file and line count only as hints: the path always comes from the parsed diff, and the line must exist in `lineIndex`, otherwise `code_snippet` is searched within `LINE_TOLERANCE`. Issues it cannot pin are dropped or become file-level comments, depending on `UNANCHORED_MODE`. It emits one item with `candidates` and a `verifyPrompt`.
7. **Verify Issues** (an LLM chain, temperature 0) acts as a strict second reviewer that rejects false positives. **Apply Verdicts** applies the verdicts. If verification failed, it falls back to `VERIFY_FAIL_MODE` (`keep_all|keep_high|drop_all`). It then caps output at `MAX_COMMENTS` and emits `kind: 'comment'` items, which become tasks when severity is in `TASK_SEVERITIES`, plus at most one `kind: 'notice'` when part of the review failed.
8. **Is Inline Comment?** routes each item to **Post Inline Comment / Task** (batched one item at a time, 300 ms apart) or to **Post Failure Notice**.

The LLM nodes and Bitbucket calls use `onError: continueRegularOutput` with retries. Downstream Code nodes must therefore tolerate error-shaped or missing outputs, for example `v.error`, `<think>` tags, or fenced or malformed JSON. Keep that defensive parsing in place.
