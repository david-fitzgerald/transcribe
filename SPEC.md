---
version: 0.1.0
created: 2026-09-28
updated: 2026-09-28
---

# transcribe — spec

## Objective

A portable agent skill that turns a batch of video/audio URLs or local files into `.txt` + `.srt` transcripts on the user's own machine — human subtitles when the site has them, local whisper.cpp otherwise — so the agent can read and summarise them in-session.

## Non-goals

- Summarising or analysing inside the script — the agent does that from the `.txt`; the script makes no LLM/API calls and needs no keys.
- Cloud transcription of any kind.
- Speaker diarisation (who said what).
- Translation.
- A web UI, server, or MCP server.
- Engines other than whisper.cpp (faster-whisper, mlx-whisper, openai-whisper).
- Installing dependencies — the script detects ffmpeg / whisper-cli / yt-dlp and prints the install command; it never installs.
- Keeping the downloaded video or audio.
- Playlist/channel expansion — one video per input URL.
- A Windows support claim.

## User Interface

**Skill (primary).** The user hands the agent inputs in natural language — "transcribe these: <urls/paths>" — or invokes `/transcribe <inputs>` (`/transcribe:transcribe` when installed as a plugin). The agent runs the script, reads each `.txt`, then does what the user asked; with no further instruction it gives a short per-video summary.

**Script.** `scripts/transcribe.py`, stdlib-only Python ≥ 3.9, executable:

```
transcribe.py [-o DIR] [--model NAME|PATH] [--lang CODE] [--force] INPUT [INPUT ...]

  INPUT          URL (any yt-dlp-supported site) or local media file
  -o, --out      output directory                         default ./transcripts
  --model        whisper.cpp model name or path to a .bin  default large-v3-turbo-q5_0
  --lang         spoken language, or auto                  default auto
  --force        re-transcribe even if outputs exist
```

- stdout: one result line per input — `OK|SKIP|FAIL <tab> <how> <tab> <txt path | error>`.
- stderr: progress.
- Exit: `0` all OK/SKIP · `1` any FAIL · `2` usage error or missing dependency (before any work).

**Install.** Plugin: `/plugin marketplace add david-fitzgerald/transcribe` → `/plugin install transcribe@transcribe` (the repo is its own marketplace, also named `transcribe`). Manual: copy `skills/transcribe/` to `~/.claude/skills/` (or any Agent Skills–compatible agent's skills dir).

## Data Model

- **Input** — raw string; kind = `url` (`http://` or `https://`) or `file` (existing path). Any other `<scheme>://` FAILs as unsupported; anything else (including bare domains) FAILs as not found.
- **Result** — input · status (`OK`/`SKIP`/`FAIL`) · how (`subtitles:<lang>` / `whisper:<model>`) · txt path or error.
- **Output pair** — `<out>/<stem>.txt` + `<out>/<stem>.srt`, where `<stem>` = `<slug of title or filename, ≤60 chars>-<id>`; `id` = the site's video id (URL) or the first 8 hex of sha1(absolute path) (file).
- **`.txt` layout** — header block, `---`, then one line per cue (consecutive duplicates collapsed):

  ```
  title: …
  channel: …            (URL only)
  date: YYYY-MM-DD      (upload date, URL only)
  duration: 22m54s
  source: <url | absolute path>
  transcript: subtitles:en | whisper:large-v3-turbo-q5_0
  ---
  ```

- **Model cache** — `${XDG_CACHE_HOME:-~/.cache}/transcribe/models/ggml-<name>.bin`.

## Approach

```
preflight: ffmpeg + ffprobe + whisper-cli required; yt-dlp if any URL; curl if model not cached → else exit 2
model: resolve name → cache path; download via curl to .part, rename on success
for each input, sequentially:
  URL  → yt-dlp -J --flat-playlist --no-playlist (metadata only) → playlist? → FAIL
       → stem → outputs exist & !force → SKIP
       → human subtitle track in the video's language? → yt-dlp --write-subs --convert-subs srt → OK
         (language = --lang, else the site's metadata; if unknown, use the track only when exactly one exists)
       → else yt-dlp -f bestaudio → temp file ─┐
  file → ffprobe duration → stem → outputs exist & !force → SKIP
       → ffmpeg 16 kHz mono PCM wav ◄──────────┘
       → whisper-cli -m model -l lang -osrt → srt (exit status alone is not trusted: the .srt must exist)
  srt → parse (malformed → FAIL; zero cues → FAIL "empty transcript") → header + cue lines
  publish: stage both files in <out> as temps → delete any existing .txt → rename .srt → rename .txt;
           the .txt is published last, so its presence is the completion marker SKIP checks
  every yt-dlp call runs with --ignore-config; each step must yield exactly one expected artifact
  expected external failures (non-zero exit, missing artifact, OSError) → FAIL line; batch continues
  <out> is created up front (failure → exit 2); downloads and conversions live in a system temp dir;
  only publish staging files (.<stem>.*.tmp, same filesystem for atomic rename) sit in <out>, removed on failure
print result lines; exit 0/1
```

The skill's `SKILL.md` carries agent-side behaviour: run the script in the background for anything beyond a couple of short inputs (agent tool calls time out; re-running resumes via SKIP); summarise transcripts one at a time — in subagents where the agent has them — rather than loading a large batch into context at once.

## Decisions

- In the context of the STT engine, facing speed on Apple Silicon and one install path everywhere, we chose whisper.cpp only (Metal on Mac, single binary on Linux) to achieve GPU speed without Python ML dependencies, accepting CPU-only speed on non-Mac machines and no engine choice.
- In the context of the default model, facing download size vs accuracy, we chose `large-v3-turbo-q5_0` (574 MB) to achieve full-turbo speed and near-identical output, accepting ~3.7% word-level divergence from the 1.6 GB full model (measured on a 23-min clip: 117 s vs 119 s).
- In the context of URL inputs, facing auto-captions with no punctuation and rolling duplicate lines, we chose human-uploaded subtitles only, falling back to whisper otherwise, to achieve clean text on every path, accepting that most videos (auto-captions only) take the whisper path.
- In the context of portability, facing pip/venv friction for a public tool, we chose a stdlib-only Python script shelling out to CLIs to achieve zero-install beyond the three binaries, accepting a hand-rolled SRT/VTT parser.
- In the context of summarisation, facing API keys and cost, we chose to leave all language work to the agent reading the `.txt` to achieve a key-free tool, accepting the agent's context cost on large batches.
- In the context of long batches under agent tool timeouts, facing interrupted runs, we chose atomic writes plus skip-if-exists to achieve resumable batches with no half-written transcripts, accepting a `--force` flag to redo.
- In the context of batch throughput, facing a single GPU, we chose sequential processing to achieve predictable resource use, accepting no speed-up from parallelism on CPU-only hosts.
- In the context of distribution, facing Claude Code plugin users, manual installers, and other agents, we chose one repo that is both an Agent Skills–standard skill (spec-only frontmatter; script referenced as `scripts/transcribe.py` relative to the skill root, with `${CLAUDE_SKILL_DIR}` as the Claude Code resolution) and a single-plugin marketplace, to achieve every install path from one tree, accepting the `/transcribe:transcribe` namespace in plugin mode.
- In the context of model download, facing Python 3.14 `urllib` TLS EOF failures on CDN downloads, we chose `curl` to achieve reliable large downloads, accepting curl as a first-run dependency.
- In the context of yt-dlp calls, facing user config files that can add outputs, simulate, or post-process, we chose `--ignore-config` on every call to achieve deterministic artifacts, accepting that config-file cookies and proxies are not honoured in v0.1.
- In the context of batch error handling, facing expected external failures (CLI exit codes, whisper-cli exiting 0 with no output, missing files) versus programming bugs, we chose to translate the former into per-input FAILs at each tool boundary and let everything else propagate, to achieve resilient batches that still crash loudly on defects, accepting that a bug aborts the batch (completed outputs survive).

## Acceptance Criteria

| Criterion | Measurement | Pass |
|-----------|-------------|------|
| Local file transcribes | Run on whisper.cpp's `jfk.wav` sample | `.txt` + `.srt` written; `.txt` contains "ask not what your country can do for you" |
| URL with human subtitles skips whisper | Fake-binary test + one real YouTube smoke | Result `subtitles:<lang>`; whisper-cli never invoked |
| URL without human subtitles uses whisper | Fake-binary test + one real YouTube smoke | Result `whisper:<model>`; `.txt` non-empty |
| Batch isolates failures | 3 inputs, middle one missing | 2 OK + 1 FAIL line; exit 1 |
| Re-run resumes | Second run over the same inputs | All SKIP; no audio/subtitle download, no ffmpeg, no whisper-cli (the URL metadata probe still runs — it yields the stem) |
| No half-written transcripts | Failure before publish (e.g. whisper-cli) and interruption between publish renames, each for a fresh and a `--force` run | Before the old `.txt` is removed (including staging failures): a fresh run leaves nothing; a forced run leaves the previous complete pair untouched. After the old `.txt` is removed and before the new one is published: no `.txt` remains, so the next run redoes the input — never a stale or mismatched pair |
| Missing dependency fails fast | PATH without whisper-cli | Exit 2 before any work; message names the tool + install command |
| Header carries provenance | Inspect a URL-path `.txt` | title, channel, date, duration, source, transcript lines present |
| Speed on Apple Silicon | Default model, 23-min video | ≤ 3 min wall (measured 117 s, M5, Low Power Mode) |
| Model auto-download | First run with empty cache | Model lands at cache path via `.part` → rename; second run downloads nothing |
| Packaging valid | `claude plugin validate` + manual copy install | Validation passes; skill loads in both install modes; frontmatter uses only Agent Skills spec fields |

## Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|------------|
| yt-dlp outdated / site changed (observed: 403 on a 6-month-old yt-dlp) | URL inputs FAIL | Per-input FAIL naming yt-dlp; message says to update it; batch continues |
| Dependency missing | Nothing runs | Preflight exit 2 with the install command for the platform |
| YouTube JS challenge unsolvable (no JS runtime / EJS scripts) | YouTube URLs FAIL | README install recipe includes `deno` (Homebrew `yt-dlp` pulls it in; pipx needs `yt-dlp[default]` + deno); per-input FAIL names yt-dlp |
| whisper-cli exits 0 without writing output (bad audio) | Silent empty transcript | Require the `.srt` to exist and parse to ≥ 1 cue, else FAIL |
| Agent tool timeout kills a long batch | Batch stops mid-way | Atomic writes + SKIP make a re-run resume; SKILL.md says run in background |
| Model download interrupted | Corrupt or partial model | `.part` file, renamed only on curl success |
| whisper-cli defaults to English | Non-English audio mistranscribed | Always pass `-l` (default `auto`) |
| Whisper repeats lines on music/silence | Junk lines in transcript | Consecutive-duplicate collapse; SKILL.md tells the agent to flag remaining repetition; VAD deferred |
| Two inputs share a title or filename | One overwrites or wrongly SKIPs the other | Stem carries the video id or a path hash |
| Many long transcripts at once | Agent context exhausted | SKILL.md: summarise one at a time, subagents where available |
| Playlist URL passed | Unbounded download | Metadata probe detects `_type: playlist` → FAIL with "pass video URLs"; a watch URL carrying a `list=` parameter resolves to the single video (`--no-playlist`, verified) |
