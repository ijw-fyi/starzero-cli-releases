# starzero

`starzero` puts a StarZero library on the command line: create libraries, upload media, wait for
processing, search by transcript or by what is on screen, pull originals and thumbnails back,
run workflow templates to completion, cut a recording into podcast clips, and talk to the StarZero
agent about a library, files and renders included.
It is built for coding agents first: no prompts, no spinners, stable exit codes, and `--json` on
every command.

This repository holds the releases. The source is developed in a private repository; questions and
bug reports are welcome in the issues here.

## Install

Download the archive for your platform from the [latest release](https://github.com/ijw-fyi/starzero-cli-releases/releases/latest),
put the binary on your `PATH` as `starzero`, and check it with `starzero --version`.

| Platform | Archive |
| --- | --- |
| Linux x64 | `starzero-linux-x64.tar.gz` |
| Linux arm64 | `starzero-linux-arm64.tar.gz` |
| macOS Apple Silicon | `starzero-darwin-arm64.tar.gz` |
| Windows x64 | `starzero-windows-x64.zip` |

```sh
curl -fsSLO https://github.com/ijw-fyi/starzero-cli-releases/releases/latest/download/starzero-linux-x64.tar.gz
tar xzf starzero-linux-x64.tar.gz
sudo mv starzero /usr/local/bin/
starzero --version
```

Every release also carries `SHA256SUMS` for checking the download. Optional: an `ffprobe` on `PATH`
(from ffmpeg) lets `media upload` estimate credits and skip files that are not media.

## Authenticate

At a keyboard, log in through the browser:

```sh
starzero auth login                     # opens the browser; the CLI waits for the login to finish
starzero auth login --no-browser        # prints the URL instead and stops
starzero auth login --callback "http://127.0.0.1:47831/callback?code=..."   # finishes a --no-browser login
```

The browser login asks for every scope the CLI uses except deleting, and the token it stores lasts
5 days; run `auth login` again after that. Over VS Code Remote SSH the plain `auth login` works too: the CLI prints
the address it listens on, VS Code forwards that port, and the login page's redirect reaches it through
the forward. `--no-browser` is for everything else without a browser (plain SSH, containers, an agent session):
it prints the URL and exits. Open the URL anywhere, and when the browser lands on a page that cannot
load (`http://127.0.0.1:47831/callback?...`), run `auth login --callback` with that page's full
address. The half-finished login waits in `~/.starzero/login-pending.json` (owner-only) until then,
for up to an hour.

For scripts and agents, create an API key at https://app.starzero.ai/settings/api-keys and either

```sh
export STARZERO_API_KEY=sk_sz_...          # overrides any stored credential
starzero auth login --api-key sk_sz_...    # or store it
```

The credential is stored in `~/.starzero/credentials` (one JSON line, mode 0600; on macOS and Linux
a file that other users can read is refused). An API key there is a long-lived secret, a browser login expires after
five days; on a shared machine, prefer `STARZERO_API_KEY` for the session.
`starzero auth status` shows whose credential it is, its type, scopes and expiry; `starzero auth logout`
removes it and revokes a browser login token; `starzero credits` shows what the account can spend.

## Quick tour

### Credits

```sh
starzero credits
```

Prints the balance, the plan and when its term ends, and one row per credit note that still holds
credits (origin, credits left of the original, expiry). Uploads, workflow runs, podcast runs, chat
turns and clean renders spend from these. `media upload` checks the balance itself and refuses an
upload it cannot cover (exit 5); the other commands report the server's refusal.

### Libraries and media

```sh
starzero library list
starzero library create --name "Interviews"
starzero folder create --library <lib> --path /raw
starzero media upload --library <lib> --folder /raw *.mp4        # returns when the bytes are stored
starzero media import --library <lib> https://www.youtube.com/watch?v=...   # a video URL the server fetches itself
starzero media resolve --library <lib> https://www.youtube.com/playlist?list=...   # the videos behind a playlist, channel or folder share, minus those already in the library; nothing imported
starzero media watch  --library <lib> <mediaId>...               # blocks until processing finishes
starzero media list   --library <lib> --status completed
starzero search transcript --library <lib> --query "pricing"     # each hit prints its row, then the matched text
starzero search visual     --library <lib> --query "whiteboard"
starzero media thumbnail   --library <lib> <mediaId> --at 12.5
starzero media download    --library <lib> <mediaId> --out clip.mp4
starzero output render     --library <lib> --clip <mediaId>:12.5-30 --clip <mediaId>:41-58   # the moments as one temporary video
```

### Workflows

Run a workflow template and collect its outputs:

```sh
starzero workflow template list                                   # yours, plus shared ones used before
starzero workflow template describe <templateId>                  # the variables it expects
starzero workflow instance create --template <templateId> --library <lib> --media <mediaId> --variables vars.json
starzero workflow instance watch <instanceId>                     # silent until done; exit 0 only on completed
starzero workflow instance get <instanceId>                       # app link, chat links per branch, render ids
starzero workflow instance get <instanceId> --library --media     # plus the library name and the selected media (name, origin)
starzero workflow instance share <instanceId>                     # public page for the run's outputs; --off takes it back
starzero output url <renderId>                                    # signed private mp4 URL (curl it)
starzero output share <renderId>                                  # public share page + direct mp4 link
```

### Renders from moments

`output render` cuts moments of a library's media (the ranges `search` prints) into one video and
prints a signed URL for it. The render is temporary and watermarked by default, which is free;
`--no-watermark` gives a clean render and uses credits. A temporary render is a file behind that
URL and nothing else: download it while the URL is valid (a day), since `output url` and `output
share` only know renders that workflows and chats keep.

```sh
starzero output render --library <lib> --clip <mediaId>:12.5-30 --clip <mediaId>:41-58        # 1080p, source aspect ratio
starzero output render --library <lib> --clip <mediaId>:12.5-30 --aspect 9:16 --height 720 --no-watermark
```

### Podcasts

A podcast run cuts one media item into short clips. It is a workflow run started through a packaged
template, so once started it is followed, inspected and cancelled with the `workflow instance`
commands; `podcast-clips create` prints the instance it started plus every option it sent, defaults included.

```sh
starzero podcast-clips create --library <lib> --media <mediaId>                 # 3 clips, 1-2 min, 4:5, captions on
starzero podcast-clips create --library <lib> --media <mediaId> --clips 5 --duration mix --aspect 9:16 --caption kamua/kamua-purple --music --watch
starzero podcast-clips create --library <lib> --media <mediaId> --auto --instructions "keep the product demo"   # the agent picks count, length and pacing
starzero podcast-clips create --help                                    # every flag, its default, and the caption types
starzero podcast-clips list                                             # your podcast runs with their options; ids work with `workflow instance`
starzero workflow instance watch <instanceId>                     # then `workflow instance get` for the render ids
```

### Chat

A chat is a conversation with the StarZero agent about one library, optionally focused on a few
media items. Each `chat send` is one turn: the CLI streams the agent's reply, tool activity and
questions as they happen and returns when the turn ends. The chat keeps working server-side if you
stop waiting, and `chat get` shows where it got to.

```sh
starzero chat create --library <lib> --content <mediaId> --title "Report"   # prints the chat id and app link
starzero chat send <chatId> --message "Summarise the interview"   # streams the turn as it happens
starzero chat send <chatId> --message-file brief.md --file notes.md   # message from a file, notes.md attached
starzero chat get <chatId>                                        # transcript (last 20 entries) and whether it is still working
starzero chat get <chatId> --all                                  # the whole transcript
starzero chat list                                                # your chats, newest first
starzero chat list --query "pricing interview"                    # semantic search over your chats
starzero chat renders <chatId>                                    # videos the agent rendered; ids work with `output url`
```

### Artifacts

Artifacts are the files a chat uses or produces: uploads you attach, documents and data the agent
writes, generated audio and images. Each has a key of the form `<type>/<id>`, for example
`file/66d0…`; `artifact list` prints the keys. Uploading happens through `chat send --file`, which
ties the file to that chat.

```sh
starzero artifact list --chat <chatId>                            # one chat's artifacts
starzero artifact list --type pdf                                 # the whole account, newest first, paged (--limit, --cursor)
starzero artifact url <type>/<id>                                 # presigned URL for one artifact (curl it)
```

Add `--json` to any command for one JSON document on stdout, or set `STARZERO_JSON=1`. `chat send`
is the exception: its transcript streams as NDJSON events, then the usual `{ "ok": true, ... }` line.

## For agents

- Every command is non-interactive, with one exception: `auth login` without `--api-key` waits for a
  browser login. Agents use `STARZERO_API_KEY` or `auth login --api-key`. Everywhere else a missing
  flag is a usage error (exit 2), never a prompt.
- Human output is deterministic and free of colour or cursor movement. `--json` gives
  `{ "ok": true, "data": ..., "warnings"?: [...], "next"?: { "watch": "starzero ..." } }`.
- Errors go to stderr as `CODE: message (hint)`, or with `--json` as
  `{ "ok": false, "code", "message", "hint", "exitCode", "httpStatus"?, "apiCode"?, "requestId"?, "reported"?, "eventId"? }`.
- An `INTERNAL` error is a bug. A release binary reports it (the error message, the command name,
  the version and platform; never arguments, keys or tokens) and prints the event id; `reported` and
  `eventId` carry it with `--json`. Set `DO_NOT_TRACK=1` or `STARZERO_SENTRY_DSN=` to turn that off.
- `media upload` is idempotent: re-running it skips files already in the library (matched by
  content fingerprint) and reports them as `already-uploaded`.
- `media import` takes video URLs (YouTube, Google Drive, Frame.io, Facebook, most sites yt-dlp
  reads), up to 100 per request, and prints one row per URL: `imported`, `already-imported` (the
  server matched the platform's video id, so re-running is safe), or `failed` when the server could
  not resolve it (a playlist or channel URL, a private or removed video, the same video under
  another URL in the same call; the API gives no reason).
  There is no estimate before sending: the server refuses the batch with exit 5 when credits or
  storage fall short, and a refusal on a later batch still prints the rows already imported.
- `media resolve <url>` lists the videos behind a video, playlist, channel or folder share (Google
  Drive, Box, Frame.io, Shade, Facebook) without importing any, so an agent can choose before paying:
  one row per video with its platform id, title, duration and, for folder shares, path; `--json` adds
  the provider's other columns under `extra`. With `--library` the videos that library already holds
  are hidden (`--all` shows them with their media id), the count line says how many, and the hint is
  the `media import` command for the rest. A URL with nothing behind it exits 1.
- Every wait runs until the server finishes: `media upload --watch`, `media import --watch`, `media watch`,
  `workflow instance create --watch`, `workflow instance watch`, `podcast-clips create --watch`,
  `output render` and `chat send` have no default limit (`media resolve` is the exception: its
  `--timeout` defaults to the server's own 600 s cap). Pass `--timeout <seconds>` to cap one;
  a timeout exits 6 and, where the work continues on the server, prints the command to resume waiting.
- `--events` on `media upload`, `media import --watch` and `media watch` streams one JSON progress event per line to stderr.
  Processing a clip takes minutes server-side even when the clip is seconds long; a silent wait is normal.
- `media url` prints a presigned storage URL that carries no API key.
- `workflow instance create` bills from the first second and is never retried. Variables are checked
  against `template describe` first; a mismatch refuses the run (exit 5) unless `--force`.
- `workflow instance watch` prints nothing until the run ends (`--progress` adds one stderr line per
  30 s poll). Exit 0 means completed, 7 partially completed, 1 failed or cancelled, 6 timed out with
  the run still going. Ctrl-C leaves the run on the server and prints the resume and cancel commands.
- `workflow instance share` makes the whole run public at `https://share.starzero.ai/i/<instanceId>`
  (no signature, no expiry) until `--off`; `output share` is the signed, expiring page of one render.
  Every run view carries `shared` and `sharePage`, and a finished unshared run with outputs has `next.shareRun`.
- `workflow instance get --library --media` adds `library` (name, size) and `media` (name, status,
  duration, origin) to the view through one assets call for the library and one per selected media;
  the ids stay in `libraryId` and `mediaIds`. A media or library deleted since the run is a warning,
  and a whole-library run resolves to an empty `media`.
- Shared templates never appear in the API's list; the CLI remembers every template it described or
  started in `~/.starzero/templates.json` (id and name only) and shows them under `template list`.
- `podcast-clips create` bills from the first second and is never retried. It sends the app's defaults for
  every option you leave out and echoes the full set under `options`, so the JSON says exactly what ran.
  `--caption` takes one of the types `--help` lists; `--caption-preset` and `--caption-style` pass any
  other pair through, and the server answers a wrong one with a 400 naming the preset.
- `chat send` blocks until the agent's turn ends, however long that takes unless `--timeout <seconds>`
  is passed, and prints the turn as
  it happens: the reply text, delegations, and a digest of finished tool calls
  (`[tools 1:18 · content_frame_bounding_boxes x9 · view_media x2 · running: render_video]`) whenever
  the agent speaks again or every 30 s, so a turn with hundreds of calls stays a few lines a minute.
  Failed calls print at once with their error, and when the agent stops to ask questions
  (`ask_questions`) they print in full and the summary's `next.answer` says to reply with another
  `chat send`. `--tools` prints every finished call with its arguments instead. The footer and
  `data.context` give the size of the last call (`12,345 tokens in context`): watch it grow and start
  a new chat when it gets large, since `data.usage` is what the whole turn cost, not the context.
  Exit 6 on timeout
  and 130 on Ctrl-C both leave the turn running on the server; `chat get` reads the outcome. Exit 8
  means the agent was still busy with the previous message.
- `chat send --file <path>` uploads the file as an artifact of that chat first and prints
  `[uploaded <path> as artifact "<type>/<id>"]`, so a retry can pass `--artifact <type>/<id>` instead.
- Chats created by the CLI run gated tools without asking (yolo mode), because nobody is there to approve.
- `chat renders` reads the render ids out of the chat's render tool calls (`render_video`,
  `render_video_portrait_from_project`); a 24-hex id is a render id for `output url` and `output share`.
- `starzero credits` is the balance check before a billed command: `data.creditsLeft` is what can be
  spent now, `data.plan` is null on a free account (clean renders need a plan), and `data.notes`
  lists the credit notes still holding credits, soonest expiry first.
- `output render` blocks, silently, until the render is done (`--timeout <seconds>` to cap it) and is
  never retried. A timeout (exit 6) or Ctrl-C drops the connection, which cancels the render; there
  is no job to resume and no progress to show. The result is a file behind a presigned URL (valid
  for `urlExpiresInSeconds`) that the service removes after a while; the API keeps no record of it,
  so download it, or make a workflow or chat render for anything to keep. `--no-watermark` uses
  credits (currently 250 per rendered minute), and the server refuses it without a subscription.

## Exit codes

| Code | Meaning |
| --- | --- |
| 0 | success |
| 1 | runtime or API error |
| 2 | usage error |
| 3 | authentication or missing scope |
| 4 | not found |
| 5 | refused before sending anything (credits, storage) |
| 6 | timeout while waiting; the operation may still be running |
| 7 | partial success (some files or media failed; details on stdout) |
| 8 | conflict (file exists locally, server conflict) |
| 130 | interrupted |

## Environment variables

| Variable | Purpose |
| --- | --- |
| `STARZERO_API_KEY` | API key; overrides the stored credential |
| `STARZERO_CONFIG_DIR` | where `credentials` and the other local files live (default `~/.starzero`) |
| `STARZERO_JSON=1` | JSON output by default |
| `STARZERO_API_URL` | assets API base URL override |
| `STARZERO_WORKFLOW_API_URL` | workflow API base URL override |
| `STARZERO_CHAT_API_URL` | chat (agents) API base URL override |
| `STARZERO_DEBUG=1` | print stack traces for internal errors |
| `DO_NOT_TRACK=1` | never report a crash (any value works) |
| `STARZERO_SENTRY_DSN` | where a release binary reports crashes; empty turns reporting off, a DSN of your own replaces ours |

## Changes

Each release lists what changed on its release page.
