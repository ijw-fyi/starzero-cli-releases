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
starzero auth login --code <code>       # finishes a --no-browser login with the code the page shows
```

The browser login asks for every scope the CLI uses except deleting, and it renews itself: a command
that starts with under a day left on the token renews it first, so you stay logged in. A login belongs
to one machine: a copy of the credentials file used elsewhere gets the login revoked on both at the
next renewal, so log in on each machine, or use an API key. Machines logged in from one browser
session can share a login, so `auth logout` on one may end the other. A CLI that needs a new scope asks you to
log in again. Over VS Code Remote SSH the plain `auth login` works too: the CLI prints
the address it listens on, VS Code forwards that port, and the login page's redirect reaches it through
the forward. `--no-browser` is for everything else without a browser (plain SSH, containers, an agent session):
it prints the URL and exits. Open the URL anywhere; the page you end on shows a sign-in code with a Copy
button, and `auth login --code <code>` finishes the login. The half-finished login waits in
`~/.starzero/login-pending.json` (owner-only) until then, for up to an hour; the code itself lives ten minutes
once the page shows it. The code is redeemed only together with the verifier kept in that file, and it is spent on
first use, so handing it back to the CLI or agent that started the login through a chat or a ticket is the intended
use. It belongs to that login alone: a login link someone else sends you starts their login, and a code you hand
them logs them in as you.

For scripts and agents, create an API key at https://app.starzero.ai/settings/api-keys and either

```sh
export STARZERO_API_KEY=sk_sz_...          # overrides any stored credential
starzero auth login --api-key sk_sz_...    # or store it
```

The credential is stored in `~/.starzero/credentials` (one JSON line, mode 0600; on macOS and Linux
a file that other users can read is refused). An API key there is a long-lived secret, and so is a browser login, which renews itself;
on a shared machine, prefer `STARZERO_API_KEY` for the session.
`starzero auth status` shows whose credential it is, its type, scopes, expiry and whether it renews itself; `starzero auth logout`
removes it and revokes a browser login on the server, which also ends a chat turn running on it; `starzero credits` shows what the account can spend.

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
stop waiting, and `chat get` shows where it got to. When the chat's context fills up, `chat send`
and `chat get` say so, and `chat compact` folds the history into one summary the agent reads from
then on, billed like a turn.

```sh
starzero chat create --library <lib> --content <mediaId> --title "Report"   # prints the chat id and app link
starzero chat send <chatId> --message "Summarise the interview"   # streams the turn as it happens
starzero chat send <chatId> --message-file brief.md --file notes.md   # message from a file, notes.md attached
starzero chat get <chatId>                                        # transcript (last 20 entries) and whether it is still working
starzero chat get <chatId> --all                                  # the whole transcript
starzero chat compact <chatId>                                    # fold the history into one summary for the agent (billed like a turn)
starzero chat list                                                # your chats, newest first
starzero chat list --query "pricing interview"                    # semantic search over your chats
starzero chat renders <chatId>                                    # videos the agent rendered; ids work with `output url`
```

### Artifacts

Artifacts are the files a chat uses or produces: uploads you attach, documents and data the agent
writes, generated audio and images. Each has a key of the form `<type>/<id>`, for example
`file/66d0…`; `artifact list` prints the keys. `artifact upload` adds files to the account, outside
any chat; `chat send --file` uploads into the chat it sends to. Any key attaches to any chat with
`chat send --artifact`, including one shared from another account.

```sh
starzero artifact upload logo.png brief.pdf                       # account-level artifacts, one row per file with its key
starzero artifact list --chat <chatId>                            # one chat's artifacts
starzero artifact list --type pdf                                 # the whole account, newest first, paged (--limit, --cursor)
starzero artifact url <type>/<id>                                 # presigned URL for one artifact (curl it)
```

### Skills

A skill is a SKILL.md the agent loads when a message names it (`/cold-open`): a slug `name`, a
`description`, the body, and up to 20 text files beside it. Skills live on the account; the CLI
reads and writes them as a folder, one SKILL.md, or the `.skill` archive the app and Claude.ai
exchange, the same layout Claude Code skills use on disk.

```sh
starzero skill list                                               # NAME, IMPORTANT, UPDATED, DESCRIPTION
starzero skill get cold-open                                      # SKILL.md as stored, then its files by path and size
starzero skill get cold-open --path references/style.md           # one file
starzero skill pull cold-open --out skills                        # writes skills/cold-open/SKILL.md and the files beside it
starzero skill pull cold-open --zip                               # writes cold-open.skill, the archive the app imports
starzero skill push skills/cold-open                              # a folder, a SKILL.md or a .skill; exit 8 if the name is taken
starzero skill push skills/cold-open --replace                    # replace it whole: important and the files follow the push
starzero skill set cold-open important on                         # list it with its description in the agent's prompt
```

### Social publishing (experimental)

A render goes out to the accounts linked in the app (TikTok accounts, Facebook Pages, YouTube
channels) as scheduled posts, one per account, each with the exact content its platform takes;
nothing is translated between platforms, and linking or unlinking an account stays in the app.
These commands and their `--json` rows may change in a minor release. The browser login carries
both scopes (a login from before 0.16.0 needs `auth login` again); an API key lists with
`social:read` and schedules, edits, cancels or retries with `social:publish`.

```sh
starzero social account list                                      # ID, PLATFORM, NAME, STATUS; needs_reauth means reconnect it in the app
starzero social account get <connectionId>                        # what it may post right now, read live: privacy levels, limits, or its verdict
starzero social post schedule --render <renderId> --at 2026-11-03T16:00:00Z --targets posts.json   # one post per target, all or nothing
starzero social post list                                         # this calendar month (local time); --from/--to for another window
starzero social post list --render <renderId>                     # that render's posts at any time
starzero social post list --status failed --status needs_attention
starzero social post get <postId>                                 # status, content, attempts, the link once published
starzero social post edit <postId> --at now                       # and/or --content content.json: the whole content, replaced
starzero social post cancel <postId>                              # a held post; it stays listed as cancelled
starzero social post retry <postId>                               # a failed or needs_attention post back to scheduled
```

`posts.json` is an array of targets, or one target as an object: the account id and that
platform's content, in the platform's own vocabulary.

```json
[
  { "connectionId": "66d0…", "content": { "platform": "youtube", "title": "Cold open", "description": "…", "tags": ["clips"], "privacy": "unlisted", "madeForKids": false, "containsSyntheticMedia": false, "notifySubscribers": true } },
  { "connectionId": "66d0…", "content": { "platform": "tiktok", "draft": false, "title": "Cold open", "privacy": "PUBLIC_TO_EVERYONE", "disableComment": false, "aiGenerated": false } },
  { "connectionId": "66d0…", "content": { "platform": "facebook", "kind": "reel", "title": "Cold open", "description": "…", "draft": false } }
]
```

A YouTube post needs `title` and the `madeForKids` declaration; TikTok publishes API posts to
everyone, so `privacy` is `PUBLIC_TO_EVERYONE`, and `draft: true` sends the video to the account's
inbox to finish in the app instead; a Facebook `kind` is `reel` or `video`. `social account get`
reports what the account allows (privacy levels, duration and size limits), and the server checks
every target before scheduling any: a refusal exits 5 naming each refused account and what was
wrong, and nothing is scheduled.

Add `--json` to any command for one JSON document on stdout, or set `STARZERO_JSON=1`. `chat send`
is the exception: its transcript streams as NDJSON events, then the usual `{ "ok": true, ... }` line.

## For agents

Every command is non-interactive and deterministic, with `--json` everywhere. The binary carries its
own documentation: `starzero help "<what you want to do>"` prints the sections and the command help
that match, and `starzero help <topic>` prints one whole document. The topics, and the two sections
after them, are the documentation as the binary carries it.

| Topic | Covers |
| --- | --- |
| `starzero help billing` | Credits: how much does it cost, how many credits do I have, credits left, balance, billing, what bills, confirm before spending, estimate the cost, refused for credits, exit 5, one run at a time, low balance |
| `starzero help chats` | Agent chats: edit a video with the agent, brief the agent, make it 9:16 with captions, cut these interviews into a recap, add a lower-third, apply my brand kit and render, generate a voiceover, add music, scan the whole library, continue my chat, send a message to the agent, the agent asked a question, compact the chat, context is full, attach a file to the chat, make graphics for the video, what did the chat render |
| `starzero help conventions` | Reading the CLI's output: how do I read the output, what does the JSON look like, parse the result, warnings and hints, the next command to run, find the right command, which command does what, human output or --json, the update line on stderr, no prompts, non-interactive, delete something, print-traffic |
| `starzero help environment` | Environment variables: environment variables, set the API key in the environment, config dir, where the credentials file lives, JSON by default, override the API URL, turn off crash reports, turn off the update check, debug an internal error |
| `starzero help exit-codes` | Exit codes: what does exit code 3 mean, the command failed, exit 5, exit 6, exit 7, exit 8, exit 130, what to do after an error, the pipe closed early, usage error |
| `starzero help ids-and-links` | Ids and links: where do I get the id, which id goes where, the link failed with 404, pass the link through, folder path, library id, media id, instance id, render id, chat id, template id, artifact key |
| `starzero help login` | Logging in: log in, sign in, not logged in, exit 3, the login was revoked, needs scopes the login lacks, log in without a browser, sign-in code, API key, which scopes does a key need, credentials file, auth status, log out |
| `starzero help moments` | Finding moments: find where they talk about, search my library for, which clip shows, find the moment when, grab a frame at, download the original, get a link to the original, compile those moments, preview the moments I found, search transcript, search visual, media thumbnail, media download |
| `starzero help publishing` | Social publishing (experimental): schedule a post, publish the render to a social account, linked accounts, which posts are scheduled this month, cancel a post, retry a post, edit a scheduled post, the account was disconnected |
| `starzero help renders` | Renders and outputs: share the run, share the clips, give me a link to the video, download the output, where are my rendered videos, thumbnail of that render, the share link expired, stop sharing the run, make a preview from the moments I found, render id, output url, output share |
| `starzero help skills` | Skills: save this as a StarZero skill, write a skill for the agent, show my StarZero skills, push this skill, pull my skill to disk, edit a stored skill, mark a skill important, replace a skill, SKILL.md, the instructions the agent loads with /name |
| `starzero help upload` | Uploads and imports: upload these files, upload a folder of recordings, put this video online, add this YouTube link to my library, import this playlist, import this channel, import these videos, get a recording into StarZero, is my footage done processing, dry run, credit estimate for an upload, already uploaded, create a library, make a folder, media upload, media import, media resolve |
| `starzero help waiting` | Waiting for work to finish: is my run finished, is my footage done processing, wait for the upload to process, watch a run, how long does it take, the watch ended early, resume waiting, exit 6, exit 7, exit 130, Ctrl-C left the run going |
| `starzero help workflows` | Workflow runs and podcast clips: run the workflow, start an instance of a template, what variables does this template need, is my run finished, what have I run, cancel the run, which media did the run use, shared template, make clips from this podcast, cut this into shorts, vertical clips for TikTok, reels from this interview, caption styles |

## Exit codes

Exit codes decide, stdout informs. Read the code first, then the view on stdout and the error on stderr.

| Code | Meaning | What to do |
| --- | --- | --- |
| 0 | success | continue |
| 1 | runtime or API error | read stderr; a cancelled run also ends here. `INTERNAL` is a bug in the CLI, reported by the CLI itself: relay the message and the event id; a re-run rarely helps. `FILE_ERROR` means the disk refused a read or write of the path it names. `CREDENTIALS_LOCKED` means another process held the login; run the command again |
| 2 | usage error | fix the flags; `--help` shows the accepted values |
| 3 | authentication or missing scope: the browser login was revoked or comes from a CLI before 0.16.0, no credential, a bad credential, or a scope the credential lacks | log in again (`starzero help login`), then re-run |
| 4 | not found | check the id came from a list or view |
| 5 | refused before sending anything (credits, storage, invalid variables, a social post the server refused at preflight, a compaction with nothing new to compact) | report the numbers; nothing was billed; `starzero credits` shows the balance; for `chat compact`, send a message first |
| 6 | the wait ended early: a watch gave up, or a request went unanswered; the operation may still be running | run the resume command from the hint; for `output render` the disconnect cancelled the render |
| 7 | partial success (some files or media failed; details on stdout) | inspect the status column or sessions table; retrying is a separate decision |
| 8 | conflict (a local file exists, a skill name is taken, the agent is busy with a message, a server conflict) | choose another `--out`, or ask before `--force`; for `skill push`, ask before `--replace`; for `chat send` and `chat compact`, `chat get` until `processing` is false |
| 130 | interrupted | the run continues; the hint prints resume and cancel commands |

### When the reader of stdout goes away

When the reader of stdout closes the pipe early (`| head`, a pager quit), the CLI stops at once: a command that had already failed keeps its exit code, one still on its success path exits 0, since the output had no one left to read it. A `chat send` cut off that way keeps running on the server; `chat get` reads the turn afterwards. A command whose result matters runs unpiped.

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
| `DO_NOT_TRACK=1` | never report a crash and never check for a newer release (any value works) |
| `CI` | set by CI systems; turns the update check off |
| `STARZERO_SENTRY_DSN` | where a release binary reports crashes; empty turns reporting off, a DSN of your own replaces ours |

The `--api-url`, `--workflow-api-url` and `--chat-api-url` flags override the three URL variables for one command. The config directory holds the credential (`credentials`), a half-finished `--no-browser` login (`login-pending.json`), the templates this machine has used (`templates.json`) and the day's update check (`update-check.json`).

## Changes

Each release lists what changed on its release page.
