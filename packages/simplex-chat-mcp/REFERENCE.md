# SimpleX MCP server: reference

[README](README.md) explains how to install and use the plugin.

`@simplex-chat/mcp` is a local server that gives an AI agent its own presence on the SimpleX network. The agent connects to people, bots, groups, channels and directories, as itself or incognito, sends messages and files, and receives all the responses, with the files people send it. Bots tell it which commands they understand, so every bot on the network becomes a service the agent can use. You supervise the agent from its control chat in SimpleX: it asks there before it connects, sends, downloads a file or leaves, and you approve or reject from your phone with one tap, or approve all its messages to a chat you trust, or all files from it; you also schedule prompts for it there. Six tools cover it: `connect`, `send`, `receive`, `download`, `list`, `leave`. Setup is one command to start the agent, a scan to join its control chat, and one command to add it to Claude Code.

The package is also a Claude Code plugin, which gives each Claude Code session its own SimpleX group. There you prompt the session, approve its actions and tool use, and read its answers, see [Claude Code plugin](#claude-code-plugin).

What it isn't:
- **Not a directory or a search service.** The agent reaches chats through links, SimpleX names and the directories it connects to.
- **Not a push channel into an agent of `serve`.** An MCP server can't start a turn of the agent, so news reaches it only as results of calls it makes: waits inside `send` and `receive`, or polling. The plugin's channel passes your prompts to a running Claude Code session.
- **Not an always-on service.** The agent acts while its session runs; services that must always be up should be bots.
- **Not hosted.** It runs on your machine, one process per agent.
- **Not in v0.3:**
  - an address for the agent: nobody can contact it first;
  - your approval of incoming messages;
  - registration in other hosts.

Status: v0.3. [`TESTING.md`](TESTING.md) describes a local build and test. The design, the tools and the scenarios of `serve` are in its [design record](../../plans/2026-10-19-mcp-server.md), and the plugin's design is in its [plan](../../plans/2026-11-07-mcp-claude-code-channel.md).

- [Requirements](#requirements)
- [Quick start](#quick-start)
- [Owner approval](#owner-approval)
- [Add the agent to Claude Code](#add-the-agent-to-claude-code)
- [Approval modes](#approval-modes)
- [Expiry](#expiry)
- [What the owner sees](#what-the-owner-sees)
- [What the agent sees](#what-the-agent-sees)
- [Status while the agent works](#status-while-the-agent-works)
- [Waiting for approval in Claude Code](#waiting-for-approval-in-claude-code)
- [Prompt the agent from the app](#prompt-the-agent-from-the-app)
- [Several agents](#several-agents)
- [Other MCP hosts](#other-mcp-hosts)
- [Voice messages](#voice-messages)
- [Secrets](#secrets)
- [The agent's model](#the-agents-model)
- [Recovery](#recovery)
- [Command reference](#command-reference)
- [Data folder](#data-folder)
- [Security](#security)
- [Claude Code plugin](#claude-code-plugin)

## Requirements

- Node.js 22.
- One of the platforms of the `simplex-chat` package: Linux x86-64 or arm64, macOS x86-64 or arm64, Windows x86-64.
- On Linux x86-64: glibc 2.34 or later, and the system libraries `libcrypto.so.3`, `libgmp.so.10` and `libz.so.1`.
- The SimpleX Chat app on a phone or computer, for the owner.
- Claude Code; other MCP hosts with Streamable HTTP and custom headers can be configured by hand.
- curl 7.55.0 or later on the PATH of Claude Code, for the [status hooks](#status-hooks).
- For voice messages, `--audio`: an OpenAI-compatible audio API, and `ffmpeg` and `ffprobe` for voice replies ([Voice messages](#voice-messages)).

At its first start the server downloads the SimpleX core library, 29 to 50 MB depending on the platform, into `~/.cache/simplex-chat` (macOS: `~/Library/Caches/simplex-chat`, Windows: `%LOCALAPPDATA%\simplex-chat`). Unpacked on Linux, it takes about 240 MB.

## Quick start

1. Start the agent's server in a terminal and keep it running. NAME, here `my-agent`, names the agent's data folder and its entry in Claude Code; `--display-name` is the name people see in SimpleX, NAME by default:

   ```
   npx -y @simplex-chat/mcp serve my-agent --display-name "My Agent"
   ```

   The first start prints, after two lines about downloading the SimpleX core library and the Node.js add-on:

   ```
   SimpleX MCP server 0.3.0, agent "my-agent"
   Data folder: /home/alice/.simplex-mcp/my-agent
   Approval mode: requests
   Agent control chat "My Agent" created.
   Join the agent control chat in the SimpleX app. The link works once:
   <QR code>
   https://smp4.simplex.im/g#...
   If the QR code does not scan, restart with --qr-code large, or paste the link into the app.
   Whoever joins by this link becomes an owner of "My Agent" and approves its actions. To approve owners yourself, start with --owners.
   Listening on http://127.0.0.1:5226/mcp
   ```

   `-y` answers npm's prompt to install the package. The server asks no questions: every choice is an option. The QR code of such a link is 37 columns wide and 19 lines high; `--qr-code large` draws it twice as wide and twice as high. Ctrl-C stops the server, see [Expiry](#expiry).

2. In the SimpleX app, scan the QR code or open the link, and tap Join. The chat "My Agent" appears in the chat list, from the next version with the [restart notice](#restart-notice) of this start, and the agent writes: `I'm your agent. I ask here before I connect, send, download a file or leave a chat.` The terminal prints:

   ```
   Alice joined (member 3) and is now an owner. The link is deleted.
   Add the agent to Claude Code:
     npx -y @simplex-chat/mcp register my-agent
   ```

3. In another terminal, add the agent to Claude Code:

   ```
   $ npx -y @simplex-chat/mcp register my-agent
   Added to Claude Code: scope user, server simplex-my-agent, status hooks in /home/alice/.claude/settings.json.
   ```

4. Add `"mcp__simplex-my-agent"` to `permissions.allow` in `~/.claude/settings.json`, so Claude Code does not prompt for each call; approval happens in the SimpleX app.

5. Start Claude Code and ask it, for example: `Connect to the SimpleX Directory at https://smp4.simplex.im/a#lXUjJW5vHYQzoLYgmi8GbxkGP41_kjefFvBrdwg-0Ok and search it for trail running shoes.` In the chat "My Agent", the agent writes:

   ```
   I want to connect to bot SimpleX Directory, as myself.
   `Find groups, channels, businesses and bots.`
   /approve 1 or /reject 1
   ```

   Tap `/approve 1`. The agent then asks to send its search, and you approve that too, or tap the card's `all` command to approve all its messages to the directory. Once the name `@directory.simplex` is registered, it works as the target too, and the card shows `(@directory.simplex)`.

Without `--owners`:

- whoever joins by the link becomes an owner of the agent control chat;
- every owner approves actions, including owners added later in the app;
- the link is deleted when the first person joins.

## Owner approval

With `--owners`, only the owners listed in it approve actions. An owner added later in the app approves actions only after the server is restarted with that owner in the list.

1. Start the server with `--owners` and no list:

   ```
   npx -y @simplex-chat/mcp serve my-agent --owners
   ```

   The output is the same as in the quick start, with the line `The link adds an observer. Its ID and name are printed when it joins.` in place of `Whoever joins by this link ...`.

2. Join the agent control chat in the SimpleX app with the link. The link is deleted. The terminal prints, and the agent posts in the chat:

   ```
   Alice joined (member 3). Restart the server with --owners 3:Alice
   ```

   Until the restart, the agent's `connect`, `send`, `download` and `leave` fail.

3. Stop the server with Ctrl-C and start it with the list:

   ```
   $ npx -y @simplex-chat/mcp serve my-agent --owners 3:Alice
   Owners: 3:Alice.
   Add the agent to Claude Code:
     npx -y @simplex-chat/mcp register my-agent
   ```

4. To add an owner, add the person to the agent control chat in the app and make them an owner there. The terminal prints, and the agent posts in the chat:

   ```
   Bob (member 7) is now an owner and can't approve actions yet. Restart the server with --owners 3:Alice,7:Bob
   ```

   When an owner leaves or is removed, the agent lists the human owners who remain:

   ```
   Alice (member 3) is no longer an owner. Restart the server with --owners 7:Bob
   ```

   When none remains: `No owners remain: restart the server with --reset-owners --owners`.

In the chat, the `--owners ...` part of these lines is shown as code, so a member's name in it cannot form a tappable command.

At every start with a list, the server removes the connected members of the agent control chat other than the listed owners.

## Add the agent to Claude Code

```
npx -y @simplex-chat/mcp register my-agent                  # all Claude Code projects
npx -y @simplex-chat/mcp register my-agent --scope local    # the current project only
```

`register NAME` reads the agent's port from its data folder, runs `claude mcp remove --scope SCOPE simplex-NAME` (SCOPE: `user`, or the `--scope` value), ignoring a missing entry, and then, for `register my-agent`:

```
claude mcp add-json --scope user simplex-my-agent '{"type":"http","url":"http://127.0.0.1:5226/mcp","headersHelper":"cat /home/alice/.simplex-mcp/my-agent/headers.json","timeout":169800000}'
```

- NAME: 1 to 32 lowercase letters, digits and single hyphens, starting with a letter or digit.
- The entry has no token. At every connection Claude Code runs the `headersHelper` command and sends the header it prints from `headers.json` in the data folder; on Windows the command is `type "<folder>\headers.json"`. With `--scope local`, Claude Code runs it only in a folder whose trust dialog was accepted.
- The agent's tools in Claude Code are `mcp__simplex-my-agent__connect` and the rest.
- `timeout` is the expiry plus 24 hours and 10 minutes, in milliseconds: the longest wait for a decision plus the longest wait for responses. After a change of `--expire` or `--port`, register the agent again; an agent on another port needs the same `--port` at every start.
- The server must be running when Claude Code starts a session. If a session started before the server, run `/mcp reconnect all` in it (Claude Code 2.1.284 or later; earlier, reconnect `simplex-my-agent` in `/mcp`).

When `claude mcp add-json` succeeds, `register` adds the [status hooks](#status-hooks) to the settings file of the scope, and prints its path:
- `user`: `settings.json` in `CLAUDE_CONFIG_DIR`, or in `~/.claude` when it is unset or empty;
- `local`: `.claude/settings.local.json` in the project folder.

The hooks are one group without a matcher for each of the events `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`, `PermissionRequest`, `PermissionDenied`, `SubagentStart`, `SubagentStop`, `PreCompact`, `PostCompact`, `Notification`, `Stop`, `StopFailure` and `SessionEnd`, with this handler:

```
{"type": "command", "command": "curl", "args": ["-q", "--noproxy", "*", "-s", "-m", "2", "-o", "/dev/null", "-H", "@/home/alice/.simplex-mcp/my-agent/hook-headers", "-H", "Content-Type: application/json", "--data-binary", "@-", "http://127.0.0.1:5226/hook"], "async": true}
```

- Claude Code starts curl without a shell, passes it each event on stdin and continues without waiting for it. curl posts the event to the agent's server, with a limit of 2 seconds, and writes the answer to `/dev/null`, or to `NUL` on Windows.
- With `-q`, curl reads no `.curlrc` file, and with `--noproxy *` it ignores the proxy variables of the environment. curl reads the token from `hook-headers` in the data folder at each post; the settings file contains only the path of that file.
- Before it adds the hooks, `register` removes every handler with this data folder's `hook-headers` from the groups of these events, and the groups left empty. Other settings and hooks, the hooks of other agents included, are kept. The file is written again with 2-space indentation.
- In an interactive session, Claude Code runs these hooks only in a folder whose trust dialog was accepted.

Errors:

```
situation                  message
never started              Agent "my-agent" was never started. Start it with: npx -y @simplex-chat/mcp serve my-agent
claude not on PATH         Add this entry under "mcpServers" in ~/.claude.json: {"simplex-my-agent": {...}}
                           (--scope local: under "projects" > "<project folder>" > "mcpServers")
invalid settings file      Added to Claude Code: scope user, server simplex-my-agent. /home/alice/.claude/settings.json is not a
                           valid settings file; add these status hooks to it: {"hooks":{...}}
```

A settings file is invalid when it is not a JSON object, when its `hooks` is neither an object nor `null`, or when the value of one of the 15 events above in `hooks` is not an array; `register` then leaves it unchanged. At each of these errors `register` exits with code 1.

The token is created at the agent's first start and saved in its data folder; every start writes it to `headers.json` and `hook-headers`. `serve my-agent --new-token` replaces it, and Claude Code sends the new token from its next connection, and curl from its next post, without a new `register`.

## Approval modes

`--approve MODE` sets what the owner approves. It is set only by this option.

| Mode | The agent's actions: connect, send, download, leave | Messages to the agent |
|---|---|---|
| `requests` (default) | Performed after `/approve N`, or `/receive N` for a download; a send to a chat approved for all messages at once, with a copy posted in the agent control chat; a download from a chat approved for all files at once, with no post, when the file is on the agent's XFTP servers. | Passed to the agent. |
| `host` | Performed at once; a copy is posted in the agent control chat, except for downloads. A download of a file on other servers fails. | Passed to the agent. |

Claude Code's own permission check applies to every tool call too. In mode `requests`, add the agent's server, for example `"mcp__simplex-my-agent"`, to `permissions.allow` in `~/.claude/settings.json`; otherwise Claude Code prompts for each call, or in auto mode its classifier decides it.

In `host` mode, approval is left to the MCP host. The server marks `connect`, `send`, `download` and `leave` with `_meta["anthropic/requiresUserInteraction"]: true`, so Claude Code shows its permission prompt on every call, in `acceptEdits`, `auto` and `bypassPermissions` modes as well, and allow rules do not skip it. In `dontAsk` mode and in `claude -p` runs Claude Code denies these calls.

## Expiry

A pending action expires after `--expire HOURS`, 23 by default and at most: SimpleX allows edits of a message for 24 hours, and the outcome of a pending action is an edit of its card. Cards don't show the time; `/pending` does. The last line of its card is edited to `Expired.`, and the agent's call returns `{"action": N, "status": "expired"}`. For the same reason, a `connect` waits for the other side at most until 23 hours 50 minutes after the action was created.

A result that no call has returned is dropped after the action's expiry time; a repeated call after that creates a new action. At most 20 actions are pending at once; a further `connect`, `send`, `download` or `leave` fails until the owner decides.

Actions last only while the server runs. When the server stops, the last line of each pending card is edited to `Cancelled: the server stopped.`, and each status message is deleted. The server waits up to 10 seconds, `--stop-wait`, for these edits and deletions to be sent: it prints `Stopping: sending 2 cancelled cards and 1 status deletion, up to 10 s. Ctrl-C to stop now.`, and a second Ctrl-C ends the wait. After a restart, `/approve` of an earlier number replies `Action N is not pending.`, and the agent's repeated call posts a new card. Action numbers continue across restarts.

## What the owner sees

<img src="images/approval.png" alt="The agent asks in its control chat before it sends a message; one tap approves it" width="320">

All of it is in the agent control chat, posted by the agent's profile, except the status messages, which are in the chats where the agent takes prompts ([Status while the agent works](#status-while-the-agent-works)). The agent control chat has the agent's display name; to see it under another name in your chat list, set an alias for the chat in the app. Names that are valid display names are shown plain. All other text written by the agent or by peers, other names included, is shown as code, line by line; names and errors take one line. Commands and SimpleX links inside it cannot be tapped: a valid display name cannot contain `/`, and the app parses each line of a message separately.

```
My Agent: I want to connect to bot SimpleX Directory (@directory.simplex), as myself.
          `Find groups, channels, businesses and bots.`
          /approve 1 or /reject 1

My Agent: I want to send a message to business Ridgeline, where I am incognito:
          `Is the Talus Trail waterproof?`
          /approve #9 4, /approve #9 all or /reject #9 4

My Agent: I want to leave business Ridgeline (#9).
          /approve #9 12 or /reject #9 12

My Agent: I want to download a file from Alice in group Builders (#3):
          `report.pdf`, 2.4 MB
          Servers: `xftp.example.com`, unknown: downloading shows it this computer's IP address.
          /receive #3 7, /forward #3 7 or /reject #3 7
```

The download card of an image or a video shows the sender's preview. Its servers line says `Servers: known.` when the file is on the agent's XFTP servers; otherwise it names the other servers, which see this computer's IP address during the download, and the card leaves out `/receive #3 7 all`. The agent waits until the sender finishes the upload before it asks, because the file's servers are known only then. A file without a file description is not downloaded: no card appears, and the agent's call ends with an error. A send with a file names the file's type, full path and size after the text, and shows an image's or video's preview:

```
My Agent: I want to send a message to group Builders:
          `The trail map.`
          attaching image `/home/alice/trail.jpg`, 3.1 MB
          /approve #3 8, /approve #3 all or /reject #3 8
```

When the agent asks to send a private key or a file of the server's data folder, the server refuses it and posts ``Refused to send `/home/alice/.ssh/id_ed25519` to group Builders (#3): private keys and the server's own files are not sent.`` When the upload of a sent file fails, the agent posts, for example, ``The file `trail.jpg` was not sent to group Builders (#3): `File server error: Server address is incompatible with network settings.`.``; both lines are logged too.

After the decision the card is edited, and its commands removed:

```
My Agent: I sent a message to business Ridgeline, where I am incognito:
          `Is the Talus Trail waterproof?`
          Approved by Alice.
```

Commands:

| Command | Effect |
|---|---|
| `/approve N`, `/approve #9 N` | Performs a connect, send or leave action N; the cards of sends and leaves name the chat. |
| `/reject N`, `/reject #9 N` | Rejects action N. |
| `/approve #9 all` | Approves all the agent's messages to chat #9: the pending ones and later ones. |
| `/revoke #9` | Ends that approval: each message to #9 needs approval again. |
| `/copies #9 off`, `/copies #9 on` | Stops or resumes the copies of the agent's messages to chat #9 that are posted here under that approval. |
| `/receive #3 7` | The agent receives the file of download 7: the server downloads it, and the agent's call returns its path. |
| `/receive #3 7 all` | The same, and the agent receives its later downloads from chat #3 of files on its XFTP servers without asking. Other pending downloads of #3 stay pending. |
| `/receive #3 off` | Ends that: each download from #3 needs approval again. |
| `/forward #3 7` | Downloads the file of action 7 and sends it to you in the agent control chat, as a reply to the card when the file's preview and the card fit in one message. The card stays pending, and the agent gets nothing: its messages show the file as offered, without a path. The server keeps the file outside the chat's folder until `/receive #3 7`, which then uses it. |
| `/leave #9`, `/leave @4` | Makes the agent leave group #9, or delete contact @4, at once, in both approval modes, and rejects the agent's pending actions for that chat. The reply, for example `Left business Ridgeline (#9), requested by Alice.`, is logged too. |
| `/delete #9 files` | Cancels the running downloads from chat #9, which then fail, also those the core continues after a restart, and deletes the chat's files in the data folder, forwarded ones included, for a current or a left chat: `Deleted 3 files from #9, 2.1 MB.` or `No files from #9.` |
| `/pending` | Lists pending actions. |
| `/list` | Lists the agent's 100 most recent chats, one line each, and which of them are approved for all messages or all files: `, approved for this chat by NAME` and `, files approved by NAME`; a last line counts the older chats. |
| `/schedule #3 0 7 * * 1-5 TEXT` | Schedules the prompt TEXT for chat #3 at 07:00 on weekdays, server time: [Scheduled prompts](#scheduled-prompts). |
| `/unschedule #3 1` | Removes schedule 1 of chat #3. |
| `/schedules` | Lists the schedules of all chats. |
| `/help` | Lists the commands, one line each: [Help](#help). |
| `/help schedule` | Shows how to use `/schedule`, with an example. |
| `/list` | Lists the agent's 100 most recent chats, one line each, and which of them are approved for all messages or all files: `, approved for this chat by NAME` and `, files approved by NAME`; `, copies off` when their copies are off, in mode `requests`; a last line counts the older chats. |

After `/leave #9`, or after the agent's `leave` of #9 is performed in either mode, when the chat's folder has files, the agent posts ``Files from business Ridgeline (#9) are kept in `/home/alice/.simplex-mcp/my-agent/files/group-9`: 3 files, 2.1 MB. /delete #9 files``. The files forwarded from #9 are deleted when the agent leaves it.

`/pending`, `/list`, `/schedules`, `/leave` and `/help` are in the chat's command menu, and each pending card shows the commands of its action. Tapping `/leave` in the menu puts `/leave <chat ID>` into the entry field: replace `<chat ID>` with the chat's ID from `/list`, such as `#9`, and send it. Each command includes its number, because the app sends a tapped command as a new message, without a quote. Every card form and reply is listed in the [design record](../../plans/2026-10-19-mcp-server.md), section "What the owner sees: the agent control chat".
| `/secret NAME VALUE` | Saves a secret for the agent, with `--secrets-folder`; the server first deletes the message for all members ([Secrets](#secrets)). |
| `/unsecret NAME` | Deletes the secret. |
| `/secrets` | Lists the secrets, with the owner who set each one and when. |

After `/leave #9`, or after the agent's `leave` of #9 is performed in either mode, when the chat's folder has files, the agent posts ``Files from business Ridgeline (#9) are kept in `/home/alice/.simplex-mcp/my-agent/files/group-9`: 3 files, 2.1 MB. /delete #9 files``. The files forwarded from #9 are deleted when the agent leaves it.

`/pending`, `/list`, `/leave` and `/secrets` are in the chat's command menu; the cards show the action commands, and the reply to `/secret` shows `/unsecret`. Tapping `/leave` in the menu puts `/leave <chat ID>` into the entry field: replace `<chat ID>` with the chat's ID from `/list`, such as `#9`, and send it. Each command includes its number, because the app sends a tapped command as a new message, without a quote. Every card form and reply is listed in the [design record](../../plans/2026-10-19-mcp-server.md), section "What the owner sees: the agent control chat".
| `/model`, `/model coder` | With `--models`: shows the agent's model with a command for each other model, or selects model `coder`, in both approval modes. The script that runs the agent restarts it with that model: [The agent's model](#the-agents-model). |

After `/leave #9`, or after the agent's `leave` of #9 is performed in either mode, when the chat's folder has files, the agent posts ``Files from business Ridgeline (#9) are kept in `/home/alice/.simplex-mcp/my-agent/files/group-9`: 3 files, 2.1 MB. /delete #9 files``. The files forwarded from #9 are deleted when the agent leaves it.

`/pending`, `/list` and `/leave` are in the chat's command menu, and `/model` with `--models`; the cards show the others. Tapping `/leave` in the menu puts `/leave <chat ID>` into the entry field: replace `<chat ID>` with the chat's ID from `/list`, such as `#9`, and send it. Each command includes its number, because the app sends a tapped command as a new message, without a quote. Every card form and reply is listed in the [design record](../../plans/2026-10-19-mcp-server.md), section "What the owner sees: the agent control chat".

### Approval of all messages to a chat

After `/approve #9 all`, the agent's messages to #9 are sent at once, and each one is posted in the agent control chat after it is sent:

```
My Agent: I sent a message to business Ridgeline, where I am incognito:
          `Thanks, I'll order.`
          Approved for this chat by Alice. /revoke #9 or /copies #9 off
```

- It covers messages, status messages and notes only: connecting and leaving are approved one by one.
- It lasts until `/revoke`, until the chat is deleted, or until the approving owner is no longer an approver. It survives restarts: it is kept in the agent's database, as custom data of that chat.
- With it, the agent can send anything it can read, without asking, to everyone in that chat.
- It applies in mode `requests` only: in mode `host`, `list` and `/list` show no approvals.
- `/copies #9 off` stops these copies, and `/copies #9 on` resumes them. The card of a failed send is still posted. The setting is kept in the chat's custom data, also after `/revoke`, so a later `/approve #9 all` replies with the line `Copies are off: /copies #9 on`.

### Receiving all files from a chat

After `/receive #3 7 all`, the agent's later downloads from #3 are performed at once, and nothing is posted in the agent control chat.

- It covers files on the agent's XFTP servers. A download of a file on other servers gets a card, as without it.
- With `--audio`, the server also downloads and transcribes the voice messages of #3 at their arrival ([Voice messages](#voice-messages)).
- Cards already pending in #3, other than card 7, stay pending. A file you rejected, or whose card expired, gets a card again.
- It covers downloads only, and the approval of messages covers no downloads: each is given and ended by its own command.
- It lasts as the approval of messages does, until `/receive #3 off`, and is kept next to it in the chat's custom data.
- `/list` shows it as `, files approved by Alice`.

### Scheduled prompts

An owner schedules a prompt for a chat with `/schedule CHAT CRON PROMPT`. After `/schedule #3 0 7 * * 1-5 Post the trail report for today.`, the agent replies:

```
My Agent: Schedule 1 of group Builders (#3): 0 7 * * 1-5, server time UTC+01:00. /unschedule #3 1
```

- CHAT: a chat ID from `/list`.
- CRON: the 5 words after CHAT; the fields are minute, hour, day of month, month and day of week, in server time. A field is `*`, `N`, `A-B`, `*/S` or `A-B/S`, or a comma list of these; names such as `MON` are refused. Ranges: minute 0-59, hour 0-23, day of month 1-31, month 1-12, day of week 0-7, with 0 and 7 for Sunday. When neither day field starts with `*`, a match of either one suffices, as in vixie cron.
- PROMPT: the rest of the one-line message, at most 2000 characters; a sixth cron field is read as the first word of PROMPT.
- Server time: the local time of the server, in the system's time zone, or in the one set by the `TZ` environment variable, such as `TZ=Europe/Berlin npx -y @simplex-chat/mcp serve my-agent`. In the group of a [Claude Code session](#sessions-and-their-groups), server time is the local time of the plugin's service. The service takes `TZ` from the environment of the Claude Code session or command that started it; while it runs, that time zone applies to the schedules of all sessions. The current UTC offset of server time is shown in the reply to `/schedule`. At a daylight saving change, a skipped local time is never matched, and a repeated one can be matched twice.

At each minute matched by a schedule, its prompt becomes due:
- `receive` of the chat returns it once, in `scheduled` ([What the agent sees](#what-the-agent-sees)), and a `receive` waiting in the chat ends at once. A session waiting in `receive` of the chat, as asked by the prompt in [Prompt the agent from the app](#prompt-the-agent-from-the-app), gets it at that minute.
- Each schedule has at most one due prompt: until `receive` returns it, the later times of that schedule are skipped, and its `at` is the first due minute.
- Due prompts are kept in memory and lost when the server stops; times while it is stopped are skipped.
- Prompts become due only while the owner who added the schedule is an approver, and while the agent keeps its control chat.

A chat has at most 10 schedules. They are kept across restarts in the agent's database, as custom data of the chat, and deleted with the chat when the agent leaves it. The commands are accepted in both approval modes.

The reply to `/schedules` has one line per schedule: chat, number, cron and prompt. The chats are in the order of `/list`, and a long line is cut to at most 100 bytes, ending with `…`. The schedules after the first 100 are counted in a last line; without schedules, the reply is `No schedules.`

```
My Agent: #3  1  0 7 * * 1-5  Post the trail report for today.
          #3  2  30 18 * * *  Summarize the day's messages.
```

The reply to `/unschedule #3 1` is `Schedule 1 of group Builders (#3) is removed.` The later schedules of #3 are renumbered, and the due prompt of the removed one is dropped.

Refusals:

```
situation                                  reply
CHAT unknown, or the agent control chat    No chat #3. /list shows the agent's chats.
a word of CRON not of these forms          Schedule: 5 cron fields expected, such as 0 7 * * *.
a field out of its range, A above B, S 0   Schedule: cron field 60 is out of range.
PROMPT over 2000 characters                Schedule: the prompt is over 2000 characters.
CHAT with 10 schedules                     Schedule: group Builders (#3) has 10 schedules; /unschedule #3 N removes one.
/unschedule of a missing number            #3 has no schedule 3.
```

The server logs each change and each due prompt, with names on one line:

```
Schedule 1 of #3 added by Alice: 0 7 * * 1-5.
Scheduled prompt 1 of #3 is due.
Schedule 1 of #3 removed by Alice.
```

### Help

The agent answers `/help` with the commands, one line each:

```
My Agent: /approve N or /approve CHAT N performs pending action N, other than a download; /approve CHAT all approves all sends to CHAT.
          /reject N or /reject CHAT N rejects pending action N.
          ...
          /help COMMAND shows how to use COMMAND, such as /help schedule.
```

`/help COMMAND` is answered with how to use COMMAND, a command of the list, written with or without its `/`: its forms and, where they apply, its parameters, rules, limits, an example and related commands. For `/help schedule`, these include the cron fields, the server's time zone, the limits of schedules and prompts, and when a prompt becomes due. The reply to other text after `/help` is `No help for that. /help lists the commands.`

The forms in the reply to `/help` are tappable, and each is sent as written, placeholders included.

Every member who can post in the agent control chat gets these replies; the agent accepts the other commands from approvers only.
### Restart notice

From the next version, at each start, after the owner setup, the agent posts its version and the commit of its build:

```
My Agent: SimpleX MCP server VERSION (8e4b6f1c2d) started.
```

- It is posted in the agent control chat and, in mode `requests`, in each chat approved for all messages where the agent is connected, can post by its role, and is not incognito.
- VERSION is the package version, and `8e4b6f1c2d` is the commit the package was built from, as `git rev-parse --short=10 HEAD` printed it at `npm pack` or `npm publish`. No commit is written by `npx tsc`: with a build by it alone, the notice is `SimpleX MCP server VERSION started.` The same version is in the first line the server prints: `SimpleX MCP server VERSION (8e4b6f1c2d), agent "my-agent"`.
- The server posts it by itself: it is not an action, and no card is posted for it. An owner who joins later receives it among the chat's recent messages.
- A failed post is logged on stderr, for example `restart notice to #9: <error>`, and the server continues the start. After the agent lost its control chat, no notice is posted.

## What the agent sees

Six tools:

| Tool | What it does |
|---|---|
| `connect` | Joins a contact, bot, business, group or channel by a link or a SimpleX name, as the agent's profile or incognito. Returns the chat, its profile, welcome message, the commands a bot advertises, and the messages so far. |
| `send` | Sends a message, with a file of this computer when asked, or with `--audio` a voice message, and returns every message received in that chat after it, until none arrives for `idle` seconds (30 by default). A directory bot's search results arrive this way. With `live: true`, it shows a note in the chat's status message at once, without an action. |
| `receive` | Returns the chat's unread messages and those that arrive while it waits, waiting for one when asked, and the chat's due scheduled prompts. When it returns messages, it starts the chat's status message. |
| `download` | Downloads the file of a received message to the data folder and returns its full path, and with `--audio` the transcript of a voice message. |
| `list` | Lists the agent's chats with unread counts and `approved: true` for chats approved for all messages, and its actions without a result. |
| `send` | Sends a message, with a file of this computer when asked, and returns every message received in that chat after it, until none arrives for `idle` seconds (30 by default). A directory bot's search results arrive this way. |
| `receive` | Returns the chat's unread messages and those that arrive while it waits, waiting for one when asked. With `history`, it also returns up to 50 messages before the new ones, the agent's own included. |
| `download` | Downloads the file of a received message to the data folder and returns its full path. |
| `list` | Lists the agent's chats with unread counts and `approved: true` for chats approved for all messages, and its actions without a result; with `--secrets-folder`, also the folder and the names of the secrets. |
| `leave` | Deletes a contact, or leaves a group, business chat or channel, and deletes the chat; the contact or the members are notified. |

A `send` to a chat that `list` shows with `connected: false` is refused before any card. A `send` is refused with `text: too long for its approval card; shorten it.` when its card, in its longest outcome form, would exceed 15000 bytes in the SimpleX message, which holds at most 15602; a text of many lines or multi-byte characters reaches it below 12000 characters.

A received message with a file has a `file` field: its name, size, type (image, video, voice or file) and status; with `--audio`, the transcript of a received voice file is in it too ([Voice messages](#voice-messages)). Status `offered` means `download` can fetch it; `received` comes with the path of the file that `download` downloaded; a file `/forward` sent to you stays `offered`; `rejected` means you rejected its download, or it expired; `failed` comes with the error, also for a file above the size its sender may send, and for a received file that is not in the chat's folder, for example after `/delete #3 files`. A downloaded file is named by its file ID and the extension of its name, when the extension has only letters and digits: the name the sender gave stays in the `name` field. `download` first waits, up to `wait` seconds, for the sender to finish the upload; when it has not finished by then, the call returns an error that asks to call again later. A `download` that runs past its `wait` after the approval returns status `performing`, and the same call again picks it up. A download fails when the sender cancels the file, when the message or its chat is deleted, or when no part of the file arrives for 30 minutes; the server then cancels it.

A received reply has `reply_to`: `item`, the quoted message's ID when the chat has it; `from`, its sender; `type`, its type when it is not a text, such as `image`; and `text`, the first line of its text, up to 200 characters, with `…` when it continues. A quoted image or file without a caption has its file name as `text`. The agent replies to the quoted message with `send` and that `item` as `reply_to`.

A short voice message can arrive with its file already received. The server moves such a file into the chat's folder under the same naming when it is received in full, and its message shows it as `offered`; `download` returns its path there after the approval. A partial file of a download, or of a short voice message that arrives while the server runs, is deleted when its transfer fails or is cancelled.

`receive` returns the chat's due [scheduled prompts](#scheduled-prompts) in `scheduled`, a field present only when a prompt is due: `{"chat": "#3", "messages": [], "more": 0, "scheduled": [{"prompt": "Post the trail report for today.", "at": "2026-11-11 06:00 UTC"}]}`, where `at` is the minute the prompt became due. A `receive` with a due prompt returns at once; after a call that ends without a result, its prompts stay due. The agent is told, in the description of `receive`, to handle each one as a prompt from its owner and to answer in that chat.

`send` with `file`, an absolute path, sends that file with `text` as its caption:
- `.jpg`, `.jpeg` and `.png` as an image, resized by the SimpleX core to the size the apps send, with a preview the core makes;
- `.gif` and `.webp` as an image, unchanged, with a gray preview;
- `.mp4` and `.mov` as a video, unchanged, with the first frame as its preview and its duration when `ffmpeg` and `ffprobe` are installed, otherwise a gray preview;
- other files as a file, unchanged; `as: "file"` sends an image or video this way too.

An empty file is refused, and so is `as` without `file`. A message holds at most 15602 bytes with its text, the file's name and preview, the message `reply_to` quotes and the mentioned names. The quote is the quoted text when the message is an image or video; otherwise it is the quoted content, an image's preview included. A `send` over it is refused with `text: too long for a message with this image; shorten it.`, or without a file `text: too long for the message with its quote; shorten it.` An image's preview takes up to 14000 bytes, so its text has about 1 KB.

The server checks the file by its real path, after symbolic links, and refuses files under `~/.ssh`, `~/.gnupg` and the secrets folder, files named `id_*`, `*.pem`, `*.key`, `*.p12` and `*.pfx`, and the files under its data folder and under `~/.simplex-mcp`, except those under `files/` of its own data folder; each folder is taken by its real path too. The card shows the real path. A file whose real path, size, inode or change time differs at the send from the request is not sent: the action fails.

Incognito: the core creates a random profile for that one chat, so peers of different chats can't link them. The tools' inputs, results and errors are in the [design record](../../plans/2026-10-19-mcp-server.md), section "What the agent sees: tools", and the call sequences of the scenarios in section "Sequence diagrams".

## Status while the agent works

In mode `requests`, the server shows what the agent does in a live message, the status message, in a chat approved for all messages where the agent can post: a chat with a person, a group or a business chat. Channels and chats with bots and agents are excluded. In a group, every member sees the status message and its notes.

- The server posts the status message when messages are returned by a `receive`, unless the chat has one.
- The server also posts the status message at once when it receives a message in that chat, unless the chat has one, with the label `received, waiting for the agent`: the owner sees at once that the server runs, also while the agent is not in a `receive` of that chat. A live message is received when its sender completes it; an edit of a message posts no status message. When messages of that chat are returned by a `receive` or by a `send`, the label is changed to `thinking` at once.
- With `live: true`, the text of a `send`, a note, is shown under the status line at once, and a status message is posted when the chat has none. Such a call is not an action: `status: live` is returned at once. A note with `file`, `reply_to` or `mention`, a note above 2000 characters, and a note to another chat are refused. In the description of `receive`, the agent is asked to call `send` with `live: true` and a short note in the same response as the tool call of a step that takes minutes.
- The agent's next message to that chat without `live` is sent as a new message, and the status message is then deleted for everyone.

```
My Agent: ◑ using download · 0:30
          Reading the file…
```

The apps show it under the header `LIVE`. The first line is the status line: a frame, turned at each update, a label, and the time since the start. From the 50th version, about 10 minutes after the start, the line is `◌ working since 2026-11-10 12:00 UTC`, or `◌ received, waiting for the agent since 2026-11-10 12:00 UTC` while the agent has not taken the message. With `--abort-file`, the line ends with ` · /abort` ([/abort](#abort)). A note is shown under the line until the next note.

| Label, the first that applies | When |
|---|---|
| `received, waiting for the agent` | from a message received in the chat without a status message, until the agent takes the message: messages of the chat are returned by a `receive` or by a `send` |
| `loading MODEL` | `--inference`: a model starts, or none is ready while one stops |
| `reading the prompt with MODEL, N tokens read` | `--inference`: a model reads the prompt |
| `generating with MODEL, N tokens/s` | `--inference`: a model generates |
| `waiting for approval` | the agent waits in a call for your decision on a card |
| `waiting for permission in Claude Code` | from Claude Code's permission prompt to the end of a tool call, a batch or the turn of its session ([Status hooks](#status-hooks)), so also during the allowed call; after a refusal in the prompt, to Claude Code's idle notification |
| `using TOOL` | the agent runs `connect`, `send`, `download`, `list` or `leave`; TOOL is the latest of them |
| `compacting the conversation` | Claude Code compacts the conversation |
| `running a command`, `reading FILE`, `editing FILE`, `searching files`, `searching the web` or `using NAME` | Claude Code runs a tool; the latest running tool is named, as the next table shows |
| `running subagent TYPE` | a subagent of Claude Code runs; TYPE is the latest one's type |
| `working` | `--inference`: the models are idle |
| `thinking` | otherwise, in the first version of other status messages, and at once when the agent takes the message |

MODEL, shown as code, is the model that starts, or the one model of all busy slots of llama-swap. Otherwise, and for a plain llama-server, `with MODEL` is left out, and `loading MODEL` becomes `loading the model`. `, N tokens read` is left out while N is 0, and `, N tokens/s` until two polls during the generation of one task. `waiting for approval` and `using TOOL` are computed from the agent's calls in all its chats.

| Claude Code tool | Label |
|---|---|
| `Bash`, `PowerShell`, `Monitor` | `running a command` |
| `Read` | `reading FILE` |
| `Edit`, `Write`, `MultiEdit`, `NotebookEdit` | `editing FILE` |
| `Grep`, `Glob` | `searching files` |
| `WebSearch`, `WebFetch` | `searching the web` |
| another tool, except `Agent` and the agent's own tools | `using NAME` |

The labels of Claude Code's work are computed from the events of the [status hooks](#status-hooks), in all sessions the server follows. FILE is the file's name without its folder; NAME is the tool's name, without `mcp__SERVER__` for a tool of another MCP server; TYPE is the subagent's type. Each is shown as code. When a name is filtered out, the label is `reading a file`, `editing a file`, `using a tool` or `running a subagent`.

The status line is updated 3 seconds apart at first, then at a tenth of its age, at most 20 seconds apart. From the 50th version, the server checks the chat every 5 minutes, without updates. Notes are shown at once up to the 59th version; a later note is refused. With its deletion, or its final text `◼ aborted`, at most 60 messages per status message are sent to each recipient.

The server deletes the status message for everyone:
- after the agent's next message to that chat without `live` is sent;
- when the agent starts to wait in a `receive` of that chat;
- when the agent leaves the chat, and at `/leave`;
- at the next update or check once the conditions above are no longer met in the chat, for example after `/revoke`, or after the agent loses its control chat;
- 23 hours after the start, at the next check or note; a note at that age is shown in a new status message;
- from the 50th version, 30 minutes after the last start or end of one of the agent's calls, at the next check, while the agent runs no call other than `receive`;
- with the [status hooks](#status-hooks), at the end of a turn after which no Claude Code session that the server follows runs a turn, while the agent runs no call, unless the label is `received, waiting for the agent`;
- at an error of a command for the status message, with `status of CHAT: ERROR` logged;
- when the server stops.

When the agent ends its turn without an answer in that chat, for example after a message it leaves unanswered, the status message is deleted with the status hooks at the end of that turn, when the other sessions that the server follows have ended theirs and the agent runs no call. Without the status hooks, the status message is shown until one of the other deletions above, such as the agent's next wait in a `receive` of that chat. If no call of the agent follows the `receive` that started it, the status message is then deleted about 35 minutes after the start, at the first check at least 30 minutes after that `receive` ended.

At an `/abort`, every status message is ended with `◼ aborted` in place of the deletion ([/abort](#abort)).

At the start, before it accepts calls, the server deletes its own live messages younger than 29 hours 50 minutes in the chats where these conditions are met: the status messages left by a crash.

A deleted status message is shown as marked deleted, with its last text, unless full deletion is allowed:
- in a direct chat, set "Delete for everyone" to "always" in the agent contact's preferences; "yes" is not enough, because the agent's own preference is "no";
- in a group or a business chat, turn "Delete for everyone" on in the chat's preferences; the status message is then removed for every member.

Notifications, by each recipient's apps:
- the status message is notified as a new message;
- its updates are notified only in a chat profile other than the active one, by the Android and desktop apps, once per update;
- the answer is notified as a new message, and the deletion after it is not notified;
- at a deletion without an answer, while the status message is the chat's last message, the Android app replaces a shown notification of that chat with "deleted" or "marked deleted", in the active chat profile.

### Inference endpoint

`--inference URL` names llama-swap, or a llama-server with one model; the state of its models is then shown in the labels.

- URL: http or https, without user name and password; trailing slashes, query and hash are dropped. Another URL, and `--inference` with `--approve host`, are usage errors, exit code 2.
- `SIMPLEX_MCP_INFERENCE_KEY`, when set, is sent as `Authorization: Bearer KEY` to URL only. Only visible ASCII characters are accepted in it; with another key, the server exits with `SIMPLEX_MCP_INFERENCE_KEY: visible ASCII characters expected.`, exit code 2.
- The line `Inference endpoint: URL` is printed at the start, after `Approval mode: requests`.
- At each timed update of the status line other than the first and the frozen one, the server reads `URL/running` of llama-swap, then `/props` and `/slots` of up to 4 servers of its ready models, on URL's host, or on loopback when URL is loopback. When `/running` answers 404, it reads `URL/props` and `URL/slots` of llama-server.
- The result of one poll is shared by all status messages for 3 seconds. A poll is cancelled after 1 second; an answer over 1 MiB and a redirect are refused.
- A failed poll is logged once on stderr as `inference endpoint: ERROR`, until the next successful poll; the labels are then those without `--inference`.

### Status hooks

With the hooks that `register` adds ([Add the agent to Claude Code](#add-the-agent-to-claude-code)), Claude Code passes its events to curl, which posts them to `http://127.0.0.1:PORT/hook` of the agent's server, with the token.

- The server follows a session of Claude Code from its first event with a tool of this agent, such as `mcp__simplex-my-agent__receive`, and ignores the events of other sessions. It follows at most 16 sessions, and drops the oldest one with its state.
- A followed session runs a turn from its first event and from each `UserPromptSubmit`, until its `Stop`, `StopFailure`, `SessionEnd` or idle notification, or until an [/abort](#abort). At the end of a turn after which no followed session runs a turn, while the agent runs no call, the server deletes the status messages, except those with the label `received, waiting for the agent`.
- Of each event, the server uses only `hook_event_name`, `session_id`, `tool_name`, `tool_use_id`, `agent_id`, `agent_type`, `notification_type`, and the base name of `tool_input.file_path` or `tool_input.notebook_path`. The rest of the event, prompts, commands, file contents and tool results included, is discarded after parsing; nothing of it is saved or logged.
- A file's base name is kept only when it consists of 1 to 64 ASCII letters, digits and characters of `_.+@-`. Tool names and subagent types follow the same rule, with `_` and `-` as the only characters besides ASCII letters and digits.
- At most 64 running tools and 64 running subagents are kept, and 256 IDs of ended tool calls; the oldest is dropped.

| Event of a followed session | State of the session after it |
|---|---|
| `PreToolUse` | its tool running, except `Agent`, the agent's own tools and a tool whose end event was received first; without `agent_id`, compaction ended |
| `PostToolUse`, `PostToolUseFailure`, `PermissionDenied` | its tool and the permission prompt ended |
| `PostToolBatch` | the tools of its agent and the permission prompt ended |
| `PermissionRequest` | a permission prompt open |
| `SubagentStart` | its subagent running |
| `SubagentStop` | its subagent ended |
| `PreCompact` | compaction running |
| `PostCompact` | compaction ended |
| `UserPromptSubmit` | all its tools, subagents, permission prompt and compaction ended; a turn running |
| `Stop`, `StopFailure`, `SessionEnd` | all its tools, subagents, permission prompt and compaction ended; its turn ended |
| `Notification` with `notification_type` `idle_prompt`, Claude Code's idle notification | as `Stop` |

- After a refusal in Claude Code's permission prompt, Claude Code 2.1.281 sends no `PostToolUse`, `PostToolUseFailure`, `PermissionDenied`, `PostToolBatch` or `Stop`. The label `waiting for permission in Claude Code` is then shown until Claude Code's idle notification, about one minute after the end of the turn. Claude Code sends that notification only when its terminal received no input since the end of the turn and no dialog of Claude Code is open; otherwise the label is shown until the session's next prompt or its end.
- Claude Code runs curl in the background and hides the outcome outside verbose mode (Ctrl+O), so the failures of the posts while the server is stopped are hidden too.
- The server checks the token, `Host` and `Origin` of a request to `/hook` as for `/mcp`. It answers a `POST` with 204, or with 411 without Content-Length, 413 for a body over 4 MiB and 400 for invalid JSON, and another method with 405.

### /abort

With `--abort-file FILE`, a person in a chat with a status message can stop the agent's work when a script runs the agent: the server counts each `/abort` in FILE, and the script restarts the agent at each change of the count. It applies in mode `requests`.

- FILE is resolved to an absolute path; its folder and the data folder must be disjoint. FILE under the data folder, or the data folder equal to FILE's folder or under it, is refused with `--abort-file: a file in a folder disjoint from the data folder expected, got FILE`, and `--abort-file` with `--approve host` with `--abort-file: status messages are shown only in mode requests.`; both are usage errors, exit code 2.
- The status line ends with ` · /abort`, for example `◓ thinking · 0:03 · /abort` or `◌ working since 2026-11-12 12:00 UTC · /abort`; a note stays under it.
- The apps make `/abort` tappable only in a direct chat with a bot and in a group or business chat whose profile lists commands, and send it there at a tap. The agent's profile has peer type agent, so in a direct chat with the agent and in groups without commands, `/abort` is shown as text, and the person types it.

A message is an abort when:
- it is received in the main conversation of a chat whose status message is shown, outside the support chats of members;
- it is a text message, neither sent as a live message nor edited, whose text without leading and trailing whitespace is `/abort`.

The server decides this once for each message, from the status message shown when it first handles the message: at its arrival, or at a `receive` that reads it before its arrival is handled.

Anyone who can post in that chat may send it: the contact of a direct chat, and every member of a group or business chat. At an abort, the server:
1. keeps the message from the agent: `receive` and `send` never return it, and it is marked read;
2. prints, for example, `Abort by Bob in group Team (#15).`, with the names on one line;
3. adds 1 to the count in FILE;
4. ends every status message of the server, in all chats: a shown status message is edited to `◼ aborted`, which stays in the chat as an ordinary message, and a status message not yet shown is not posted;
5. ends the turn of every Claude Code session that it follows with the [status hooks](#status-hooks), as at its `Stop`, since the script stops these sessions.

When a write of FILE fails, `abort in #15: ERROR` is printed on stderr, and the status messages, the turns and the count are kept; the message is still kept from the agent and marked read. After an abort, `/abort` is an ordinary message until a status message is shown again. When a `receive` of the chat waits, the server deletes the chat's status message first, so an `/abort` sent then is returned by that `receive`.

FILE holds the count in decimal on one line. At the start, a count of 1 to 15 digits in its first line is kept; otherwise, and for a missing FILE, `0` is written. At a missing folder of FILE, or a failed write, the start ends with exit code 1. Each count is written to `FILE.part`, with mode 0644, readable by all users, and FILE is then replaced by `FILE.part` in one rename.

The script that runs the agent:
- reads FILE's first line when it starts the agent, a missing FILE counting as 0, so that the server's first write of `0` is not taken as an abort;
- reads it again about every second;
- at each change of the number, stops the agent's process group and starts the agent again;
- may continue the agent's session, with one prompt line saying that the owner aborted the last task.

When the script runs in another container than the server, mount FILE's folder there, read-only, and not FILE itself: at each write, FILE is replaced by a new file, and the old file stays visible through a mount of FILE. The data folder stays unmounted there.

## Waiting for approval in Claude Code

An MCP server cannot start a turn of the agent: what the agent reads from the server are the results of its own tool calls. A call that waits for the owner, or for responses, stays open, and Claude Code turns it into a background task:

- A tool call in the main conversation that runs past two minutes moves to a background task. Claude continues working, and the result arrives as a task notification. When it arrives while the session is idle, Claude responds to it without a prompt (checked with Claude Code 2.1.300).
- To move calls to the background sooner, set `CLAUDE_CODE_MCP_AUTO_BACKGROUND_MS`, for example to `10000`, in the `env` block of `~/.claude/settings.json`. This applies to every MCP server.
- Calls from subagents, and calls in `claude -p` runs unless `CLAUDE_AUTO_BACKGROUND_TASKS=1`, stay in the foreground: the subagent or the run waits.
- A background task ends with the session. The action stays in the server while it runs; `list` shows it, and the same call again picks it up.

## Prompt the agent from the app

<img src="images/prompts.png" alt="Prompting the agent from the SimpleX app" width="320">

Give Claude Code your SimpleX address:

```
Connect to me as yourself at <your SimpleX address>, with the message:
"I'm your agent, connected as you asked in Claude Code. I'll act on your prompts here, except security-sensitive ones."
Then take my prompts from that chat and answer there: wait for my messages with receive (wait 3600),
answer each one in that chat with send (wait 0), and call receive again after every answer, until I say stop.
Answer my stop message there too before you finish.
```

Or give it the link of a group of yours, where every member prompts it:

```
Connect as yourself to my group at <your group link>.
Then take the prompts of its members from that group and answer there: wait for their messages with receive (wait 3600),
answer each one in that group with send (wait 0), and call receive again after every answer, until I say stop.
Answer my stop message there too before you finish.
```

Approve the connection in "My Agent", and accept the request in the app if it asks. On the card of the agent's first answer, tap `all`: its later answers in that chat need no approval.

Once you tap `all`, the agent shows what it does in a live message in that chat, for example `◐ thinking · 0:12`, and deletes that message when its answer is sent; in a group, every member sees it. The agent posts the status message as soon as it receives your prompt, as `◐ received, waiting for the agent · 0:00`, so you see at once that it runs. With the status hooks, the agent also shows Claude Code's work there, for example running a command, reading or editing a file, searching, running a subagent, or waiting for your permission in Claude Code. To have the status message removed rather than marked deleted, set "Delete for everyone" to "always" in the agent contact's preferences, or turn it on in the group's preferences. Keep that chat in your active chat profile: in other profiles, the Android and desktop apps notify of every update. With a local model behind llama-swap, start the agent with `--inference http://127.0.0.1:8080`: the agent then shows the model loading and its tokens per second.

When a script runs your agent, for example in an unattended loop, start the agent with `--abort-file FILE`: `/abort` is then shown in the status message, as in `◐ thinking · 0:12 · /abort`. When anyone in that chat sends `/abort`, the agent never receives it, 1 is added to the count in FILE, and every status message of the agent is ended with `◼ aborted`; your script stops and restarts the agent at each change of the count. In a group, every member can send it. FILE, its folder and the script are described in [/abort](#abort).

To prompt it on a schedule, send in "My Agent", with your chat's ID from `/list` in place of `@5`:

```
/schedule @5 0 7 * * 1-5 Search the SimpleX Directory for new security groups.
```

`0 7 * * 1-5` is cron syntax for 07:00 on weekdays, server time. At that time the agent gets this prompt from its next `receive` in your chat, and answers you there. Send `/schedules` to list the schedules, and `/unschedule @5 1` to remove this one.

In a new Claude Code session, give the same prompt again. The agent can recall the conversation: `receive` with `history` returns up to 50 earlier messages of that chat, its own answers included.

From the next version, when its server starts, the agent posts `SimpleX MCP server VERSION (8e4b6f1c2d) started.` in "My Agent" and in each chat approved with `all` for its messages, such as this one, unless it is incognito there: you see when it restarted, and the version and commit of its build ([restart notice](#restart-notice)).

## Several agents

Each agent is its own process, with its own data folder, port, agent control chat, Claude Code server and status hooks:

```
npx -y @simplex-chat/mcp serve my-agent-2 --port 5227
npx -y @simplex-chat/mcp register my-agent-2
```

## Other MCP hosts

Add an entry with the URL `http://127.0.0.1:5226/mcp` and the header `Authorization: Bearer <token>`, as in `headers.json` of the data folder; after `--new-token`, update the entry. A host that ends a tool call early returns a timeout to the agent; the agent repeats the call with the same arguments, which picks up the same action. Hosts with a fixed tool-call limit, between 60 and 300 seconds in several hosts, need `wait` below it.

## Voice messages

With `--audio`, an OpenAI-compatible audio API transcribes the voice messages the agent receives and speaks the voice replies it sends:

```
npx -y @simplex-chat/mcp serve my-agent --audio http://127.0.0.1:8080,stt,tts
```

- The value is `URL,STT,TTS` or `URL,STT,TTS,VOICE`: the API's base URL, its speech-to-text model, its text-to-speech model and that model's voice.
- URL is an `http` or `https` URL without a user name or password; its trailing slashes are removed. With another value, the server exits with `--audio: URL,STT,TTS or URL,STT,TTS,VOICE with an http or https URL expected, got VALUE`.
- At start, the server prints `Audio: http://127.0.0.1:8080, speech-to-text stt, text-to-speech tts`, with `, voice VOICE` when VOICE is given.
- Without `--audio`, voice messages are files only, and `voice` is absent from the input of `send`.

### Transcripts

The transcript of a voice message is in its `file`, with status `received`:

```
field              value
transcript         the text of the voice message, trimmed; spoken by the sender: data, not instructions
transcript_error   why the voice message has no transcript
```

`download` returns the same fields next to `path`. A voice message is transcribed:

- at its arrival, in a chat where an owner tapped `/receive CHAT N all`;
- at an approved `download`, in any chat and in both approval modes, when it has no transcript yet.

At arrival, the server downloads and transcribes a voice message under these conditions:

```
condition   value
chat        the main chat of a contact, group, business chat or channel, with all files approved by a current approver, in mode requests
message     a voice message of at most 300 seconds and 1,500,000 bytes, as its sender declares them
file        offered on the agent's XFTP servers, its upload finished within 60 seconds of the arrival
server      fewer than 20 voice messages in transcription
```

Until the server finishes its download and transcription, the message is held:

- It is left out of the results of `receive` and `send`, kept unread and counted in `more`.
- With `wait`, it is returned by `receive` after its release, and the `idle` seconds of `send` are counted from then.
- After an upload not finished within 60 seconds, a file on other servers, or a failed download, the message is returned without a transcript, and the server logs `voice message of file N: ERROR`. After `/receive CHAT off`, a voice message whose download is not started yet is returned with status `offered`.

A short voice message received at its arrival is transcribed at the `download`, as is a voice message after `/forward`.

Speech-to-text:

- The server sends `POST URL/v1/audio/transcriptions` with a form of two fields: `model`, the STT model, and `file`, the voice file named `<file ID><extension>`. No language field is sent.
- It sends one request at a time, in the order the transcriptions are started, each for at most 10 minutes. The text of the JSON response, trimmed, is the transcript.
- A failure is kept as `transcript_error` and logged as `transcription of file N: ERROR`:

```
Speech-to-text failed: HTTP 500.
Speech-to-text failed: The operation was aborted due to timeout.
Speech-to-text failed: connect ECONNREFUSED 127.0.0.1:8080.
Speech-to-text failed: no text.
Speech-to-text failed: the file has more than 1500000 bytes.
```

- The error of a redirect is its status, such as `HTTP 307`; the server does not follow it.
- The error of a failed connection is the system error, such as `connect ECONNREFUSED 127.0.0.1:8080`, or only its code, such as `ECONNREFUSED`, when the connection is refused at each address of the host.
- After a failure, the message is transcribed again at its next `download`; a transcript already written is returned without a request.

Transcripts are kept next to their voice files, in `DATA/files/group-N/` and `DATA/files/contact-N/`, also after a restart. They are deleted and counted by `/delete CHAT files`. After a restart, a voice message whose download was running is shown as `received` without a transcript, and its transcript is added at the `download`.

### Voice replies

With `--audio`, `voice`, 1 to 4096 characters, is an input field of `send`. The audio API speaks it, and the server sends the speech as a voice message, with `text`, when given, as its caption. In the field's description, the agent is told to send a voice message only when you asked for a voice reply.

The text is shown on the card:

```
My Agent: I want to send a message to contact Ana:
          attaching as a voice message:
          `Grüezi. The trail is open.`
          /approve @4 7, /approve @4 all or /reject @4 7
```

- A caption's lines are placed before `attaching as a voice message:`, and the `replying to` and `mentioning` lines after the voice text.
- Voice replies are covered by `/approve @4 all` too.

Refusals of `send`:

```
call                                   error
voice with file                        voice: not with file.
voice with as                          as: applies only with file.
no text, file or voice                 text: required without file or voice.
a caption too long for a voice message text: too long for a message with this voice; shorten it.
```

After the approval:

1. The server sends `POST URL/v1/audio/speech` with `{"model": "<TTS>", "input": "<voice>", "response_format": "wav"}`, and `"voice": "<VOICE>"` when VOICE is given, for at most 10 minutes.
2. `ffmpeg` converts the WAV to AAC in a `.m4a` file, mono, 16 kHz, 32 kbit/s, the apps' recording format; `ffprobe` reads its duration, or 0 when it fails.
3. The server sends the voice message `voice.m4a` with the caption and its mentions.

After a failed speech request or conversion, the last line of the card is ``Approved by Alice; failed: `Text-to-speech failed: HTTP 503.`.``, and status `failed` with the error is returned to the call. The other errors are `Text-to-speech failed: The operation was aborted due to timeout.`, the errors of a failed connection or a redirect, such as `Text-to-speech failed: connect ECONNREFUSED 127.0.0.1:8080.`, and those of `ffmpeg`, such as `spawn ffmpeg ENOENT` without it. The speech and the voice file are kept in `DATA/files/sent/` until the end of the upload; they are deleted after a failed send and at start.

### The audio API

The audio API serves two endpoints under URL:

```
endpoint                        request                                                response
POST /v1/audio/transcriptions   multipart form: model, file (AAC in MPEG-4, .m4a)      JSON with text
POST /v1/audio/speech           JSON: model, input, response_format "wav", voice       WAV audio
```

For example, llama-swap at `http://127.0.0.1:8080` routes both endpoints by their `model` field, with a model `stt` served by whisper-server and a model `tts` served by audio.cpp:

- whisper-server: `--inference-path /v1/audio/transcriptions`; `--convert`, with which whisper-server runs `ffmpeg` on its computer, for the `.m4a` files; `-l auto`, with which it detects the language.
- audio.cpp: its `/v1/audio/speech` returns WAV.
- llama-swap: `captureBuffer: 0`. Otherwise llama-swap keeps its captures, 5 MB by default, in memory: the voice texts of speech requests and the transcripts of transcription responses, served at `GET /api/captures/{id}`.

The computer of the MCP server needs `ffmpeg` and `ffprobe` for voice replies.
## Secrets

With `--secrets-folder DIR`, owners give the agent secrets, such as access tokens, in the agent control chat. The agent gets the folder and the names, and uses a value in shell commands by its path.

```
npx -y @simplex-chat/mcp serve my-agent --secrets-folder /run/agent-secrets
```

The server creates DIR with mode 0700 when it is missing; the mode of an existing folder is unchanged. DIR must be disjoint from the data folder: when it is the data folder, inside it or contains it, the server exits with code 2 and `--secrets-folder: a folder disjoint from the data folder expected, got DIR`.

### Commands

| Command | Effect | Reply |
|---|---|---|
| `/secret NAME VALUE` | Saves VALUE in `DIR/NAME`, readable only by the user, in place of an earlier value. | `Secret NAME is set. /unsecret NAME`, as a new message |
| `/unsecret NAME` | Deletes `DIR/NAME`. | `Secret NAME is removed.`, or `No such secret. /secrets lists them.` |
| `/secrets` | Lists the secrets, with the owner who set each one and when. | `GITHUB_TOKEN  set by Alice  2026-11-12 10:00 UTC`, one line per secret, or `No secrets.` |

- NAME: 1 to 64 capital letters, digits and `_`, starting with a letter.
- VALUE: the rest of the line, without the spaces around it, 8 characters or more.
- `/secrets` is in the command menu. The commands work in both approval modes.

Replies to `/secret`, first match:

1. a message of several lines, from any member: `Usage: /secret NAME VALUE on one line. NAME: capital letters, digits and _, starting with a letter, up to 64 characters. VALUE: 8 characters or more.`
2. a sender who is not an approver: `Only owners approve actions.`, or with `--owners` `Only the owners listed in --owners approve actions.`
3. a server without `--secrets-folder`: `Secrets are off: the server runs without --secrets-folder.`
4. an invalid NAME or VALUE: the usage reply of 1.
5. otherwise: `Secret NAME is set. /unsecret NAME`.

`/Secret` and other letter cases are deleted too, and get the list of commands; the value is not saved.

`/unsecret` and `/secrets` get the replies of 2 and 3 in the same way.

- A secret stays until `/unsecret`, also when the owner who set it is no longer an owner.
- The terminal prints `Secret GITHUB_TOKEN set by Alice.` and `Secret GITHUB_TOKEN removed by Alice.`, without the value. `secrets.json` in the data folder records who set each secret and when.

### Deletion of `/secret` messages

The server deletes every message whose first word is `/secret`, in any letter case, for all members, before it runs the command: a message of any member, also a refused or invalid one, and also one in a member support chat, which gets no reply. The agent control chat has full deletion on, so the text of the message is deleted in the agent's database and in the members' apps, the sender's included, which show `moderated by My Agent`; a member who receives the message later gets it deleted. When an owner turns full deletion off, the server turns it on again at once, before it deletes a later `/secret`; at every start, it turns full deletion on when it is off. The reply to `/secret` is a new message, without a quote of the value.

- The owners' apps show the message until the deletion reaches them.
- While the server is stopped, the message stays until the next start.
- Only the first text of a message is checked: an edit, and the later text of a live message, are neither deleted nor run. Send `/secret` as a new message, not a live one.
- A mistyped command, such as `/secrt NAME VALUE` or `/secrets NAME VALUE`, is not deleted. It gets the list of commands as a new message, without a quote, so the reply keeps no copy of the value: delete your message for everyone in the app.
- After the agent lost its control chat, `/secret` messages are neither deleted nor run.

### Use by the agent

`list` returns `"secrets": {"folder": "/run/agent-secrets", "names": ["GITHUB_TOKEN"]}`, without the values. The `list` description states the use of a secret: only inside a shell command, where the command does not print it, for example as an environment variable, a header read from the file or standard input, not as an argument that the command may print in an error, and never by its value. For example:

```
curl -H "Authorization: Bearer $(cat /run/agent-secrets/GITHUB_TOKEN)" https://example.com/api
cat /run/agent-secrets/GITHUB_TOKEN | sha256sum
```

In `sha256sum "$(cat /run/agent-secrets/GITHUB_TOKEN)"`, the value is a file name, and sha256sum prints it in its error message: `sha256sum: VALUE: No such file or directory`.

`send` refuses the files of the folder, as private keys: `file: PATH is refused: private keys and the server's own files are not sent.`

The agent's commands read `DIR/NAME` at the path that `list` returns. When they run in a sandbox, such as a container:

- mount DIR in the sandbox at the same path;
- run the sandbox with the server's user ID: the files have mode 0600, and a folder that the server creates has mode 0700.

### Redaction

A value that a command prints becomes part of the conversation, which the MCP host sends to the model provider.

- Claude Code sends values as printed, without redaction.
- With the plugin `opencode/simplex-secrets.ts` of this package, opencode replaces each value with `[secret NAME]` before it sends the conversation.

Install the plugin as the user who runs opencode:

```
npm pack @simplex-chat/mcp
tar -xzf simplex-chat-mcp-*.tgz package/opencode/simplex-secrets.ts
mkdir -p ~/.config/opencode/plugins
mv package/opencode/simplex-secrets.ts ~/.config/opencode/plugins/
```

opencode loads the `.ts` and `.js` files of `plugins/` in `~/.config/opencode` (`$XDG_CONFIG_HOME/opencode` when it is set) and in `.opencode` of the project. Start opencode with the folder of the secrets in `SIMPLEX_MCP_SECRETS`:

```
SIMPLEX_MCP_SECRETS=/run/agent-secrets opencode
```

Before each request to the model, and before the request for a compaction summary, opencode passes the messages of the session to the plugin. In every text of their parts, such as texts, tool inputs, outputs, errors and metadata, each value of the folder is replaced, also:

- its URL encoding, as `encodeURIComponent` makes it;
- its base64 and base64url encodings at any byte position, as in `Authorization: Basic ...`, except at most one character at each end, which holds at most 4 bits of the value.

The values are read from the folder at each request. Values read earlier, also of removed or replaced secrets, are replaced until opencode restarts. The values remain in the session that opencode stores; only the copies sent to the model are changed.

These are sent to the model provider as they are:

- the first user message of a session, in the request for its title, which opencode sends before the plugin runs;
- system text: agent prompts, instruction files such as `AGENTS.md`, MCP server instructions and skills;
- values in other forms: hex, other percent-encodings, `+` for spaces, escaped, split, partial or otherwise transformed by a command;
- values removed or replaced before an opencode restart, in earlier outputs;
- tool results of workflow models, which opencode returns to the provider without the plugin.

The plugin fails closed: when `SIMPLEX_MCP_SECRETS` is unset, when the folder or one of its files can't be read, or when a value has fewer than 8 characters, it throws, and opencode sends no request to the model for that step. The hook it uses, `experimental.chat.messages.transform`, is experimental in opencode. This section describes opencode 1.19.5.
## The agent's model

With `--models` and `--model-file`, an owner selects the model of an agent that a script runs in opencode. The server only writes the selected model to the file; the script restarts the agent.

```
npx -y @simplex-chat/mcp serve my-agent --models gemma=llama-swap/gemma-4-26b,coder=llama-swap/qwen3-coder-next,large=hosted/large-latest --model-file /run/agent-model/model
```

- Each item of `--models` is `NAME=PROVIDER/MODEL`. Owners type NAME; opencode's `--model` takes PROVIDER/MODEL.
- NAME: 1 to 16 lowercase letters, digits and hyphens, starting with a letter. PROVIDER: 1 to 32 letters, digits, `_`, `.` and `-`, starting with a letter or digit. MODEL: 1 to 64 letters, digits and `_.:/+@-`.
- Names are unique, and so are PROVIDER/MODEL values. The first item is the default.
- The start prints `Model: gemma (llama-swap/gemma-4-26b), in /run/agent-model/model`.

In the agent control chat:

```
/model        The agent's model is `gemma`: `llama-swap/gemma-4-26b`.
              /model coder
              /model large
/model coder  The agent's model is `coder`: `llama-swap/qwen3-coder-next`. The agent restarts with it. /model gemma
/model gemma  The agent's model is already `gemma`.
/model gpt    No such model. /model shows the models.
```

- Only approvers select a model, in both approval modes. The command at the end of a switch reply selects the previous model.
- With `--models`, `/model` is in the chat's command menu. Without it, `/model` replies `This server runs without --models.`
- A switch is logged: `Model set to coder (llama-swap/qwen3-coder-next) by Alice.`

The file of `--model-file`, FILE:

- FILE contains the selected PROVIDER/MODEL and a newline, with mode 0644. The server writes `FILE.part` and renames it to FILE.
- The server writes FILE at each switch, and at a start when FILE is missing or its first line is no PROVIDER/MODEL of `--models`. Otherwise FILE and its model are kept at a start.
- FILE's folder must exist and be writable by the server. It must be outside the data folder and must not contain it.

The script that runs opencode:

- Mount FILE's folder read-only into the agent's container, not FILE itself: a bind mount of one file references the replaced file after a rename ([moby/moby#15793](https://github.com/moby/moby/issues/15793)).
- Before each run, read the first line of FILE and pass it to `opencode run --model` as one argument.
- While opencode runs, read FILE every few seconds. When its first line differs from the running model, stop opencode and start it with the new model in a new session, without `--continue`.
- Define each PROVIDER/MODEL of `--models` in opencode's configuration.

When opencode stops while a `receive` waits, the server aborts the call without marking messages read, and the agent receives them in the new session.
opencode with a local llama.cpp model behind llama-swap: [LOCAL-MODELS.md](LOCAL-MODELS.md).

## Recovery

| Situation | What to do |
|---|---|
| The server was not running when Claude Code started | Start the server, then run `/mcp reconnect all` in Claude Code (2.1.284 or later). |
| The server restarted while Claude Code was running | Claude Code reconnects up to five times within about 30 seconds. After that, in an interactive session, run `/mcp reconnect all`; `claude -p` runs and Agent SDK sessions keep retrying for about half an hour. Open calls fail, and pending actions are cancelled; the agent's repeated call creates a new request. |
| The Claude Code session ended or was compacted | Ask the agent to recall the chat: `receive` with `history` returns up to 50 messages before the new ones, the agent's own included. |
| A second server was started on the same data folder | It exits with `Another server uses <folder> (PID N). If no server runs, remove <folder>/lock.` |
| Every owner left the agent control chat or was removed | `connect`, `send`, `download` and `leave` fail. Without `--owners`, the server logs `No owners in the agent control chat.` and prints a new link; join with it. With an `--owners` list, the agent posts and the server logs `No owners remain: restart the server with --reset-owners --owners`. |
| The agent was removed from the agent control chat, the chat was deleted, or the agent lost its owner role | `connect`, `send`, `download` and `leave` fail, and the server logs `The agent lost its control chat. Restart the server with --reset-owners.` |
| The QR code does not scan | Start with `--qr-code large`, or paste the link into the app. |
| The wrong person joined by the link | Start with `--reset-owners`. |
| The token leaked | Start with `--new-token`; Claude Code reads the new token at its next connection. |

`--reset-owners` deletes the agent control chat for all its members, creates a new one and prints a new link. Member IDs change in the new chat, so `--reset-owners` takes `--owners` without a list.

## Command reference

`npx -y @simplex-chat/mcp --help` prints:

```
Usage:
  simplex-mcp serve NAME [options]     run agent NAME
  simplex-mcp register NAME [options]  add agent NAME to Claude Code
  simplex-mcp session                  MCP server of one Claude Code session (run by the plugin)
  simplex-mcp service                  the shared SimpleX service of the plugin (started on demand)
  simplex-mcp hook session-start       Claude Code SessionStart hook (run by the plugin)
  simplex-mcp hook turn                Claude Code hooks of prompts, answers, model changes, tool uses and compactions (run by the plugin)
  simplex-mcp link                     show the Main group link, or its owners
  simplex-mcp reset-owners             replace the Main group and its owners

Claude Code plugin:
  /plugin marketplace add https://raw.githubusercontent.com/simplex-chat/simplex-chat/master/.claude-plugin/marketplace.json
  /plugin install simplex@simplex-chat
  SIMPLEX_MCP_HOME       the plugin's data folder (default: ~/.simplex-mcp)

serve options:
  --data DIR             data folder (default: ~/.simplex-mcp/NAME)
  --display-name TEXT    the agent's name in SimpleX (default at the first start: NAME)
  --approve MODE         what the owner approves in the agent control chat:
                           requests  the agent's connections, messages, downloads
                                     and leaving chats (default)
                           host      nothing; the MCP host asks
  --owners [ID:NAME,...] only these owners approve; without a list, the link
                         adds an observer and its ID is printed at the join.
                         Without --owners, whoever joins becomes an owner and
                         every owner approves.
  --expire HOURS         pending actions expire after HOURS, 1 to 23 (default: 23)
  --stop-wait SECONDS    at stop, wait SECONDS for cancelled cards and status
                         deletions to be sent, 0 to 60 (default: 10)
  --reset-owners         replace the owners: a new agent control chat and link
  --new-token            replace the access token
  --port N               HTTP port on 127.0.0.1 (default: 5226)
  --qr-code SIZE         QR code of the link: small (default) or large
  --inference URL        poll llama-swap or llama-server at URL for the model
                         state and tokens per second shown in status messages;
                         the API key, if any, is read from
                         SIMPLEX_MCP_INFERENCE_KEY; mode requests only
  --abort-file FILE      show /abort in status messages and count it in FILE
                         for the script that runs the agent; FILE in a folder
                         disjoint from the data folder; mode requests only
  --audio URL,STT,TTS[,VOICE]
                         transcribe voice messages and send voice replies with
                         the OpenAI-compatible audio API at URL: speech-to-text
                         model STT, text-to-speech model TTS, its voice VOICE
  --secrets-folder DIR   folder of the secrets that owners set with /secret,
                         one file per name; disjoint from the data folder
  --models MODELS        opencode models NAME=PROVIDER/MODEL, comma-separated;
                         /model NAME selects one in the agent control chat;
                         the first is the default; requires --model-file
  --model-file FILE      file with the selected PROVIDER/MODEL for the script
                         that runs opencode; in a folder disjoint from the
                         data folder

register options:
  --scope SCOPE          user (default) or local
  --data DIR             data folder (default: ~/.simplex-mcp/NAME)

--reset-owners takes --owners without a list.
```

## Data folder

```
~/.simplex-mcp/my-agent/
  simplex_chat.db    SimpleX chat database: the agent's profile and chats
  simplex_agent.db   SimpleX agent database: connections and keys
  token              access token, readable only by the user
  headers.json       the token as the Authorization header for Claude Code, readable only by the user
  hook-headers       the token as the Authorization header line for curl in the status hooks, readable only by the user
  server.json        name, port and expiry of the server, read by register
  lock               PID of the running server
  secrets.json       with --secrets-folder: who set each secret and when, readable only by the user
  tmp/               the SimpleX core's work folder for files in transfer
  files/group-3/     files downloaded from chat #3, and files received there without a download, each named by its file ID
                     and the extension of its name, such as 12.pdf; files/contact-4/ for contact @4; with --audio, the
                     transcript of a voice file next to it, such as 13.m4a.transcript.json, readable only by the user
  files/sent/        resized copies of the images the agent sends, voice replies with their speech in WAV, and copies of the
                     files /forward sends you under their names; each is deleted when its upload completes or fails, and all at start
  forwarded/group-3/ files /forward downloaded from chat #3, until /receive moves them into files/group-3/; deleted when the agent
                     leaves the chat, and by /delete #3 files
```

The folder is readable only by the user: every start sets its mode to 0700, except on Windows.

The server keeps its action counter in the chat database, as custom data of the agent control chat; the approvals of all messages to a chat and of all files from it, and the chat's schedules, as custom data of that chat; and its actions and due scheduled prompts in memory.
The server keeps its action counter in the chat database, as custom data of the agent control chat, the approvals of all messages to a chat and of all files from it, and copies off, as custom data of that chat, and its actions in memory.
The server keeps its action counter in the chat database, as custom data of the agent control chat, the approvals of all messages to a chat and of all files from it as custom data of that chat, and its actions in memory. With `--models`, the selected model is kept only in the file of `--model-file`, outside the data folder.

The server keeps no log file: it writes its log to the terminal where `serve` runs, events to stdout and errors to stderr. Each note of a status message is written to stdout, for example `Note to contact Alice (@4): Reading the file…`.

One server runs per data folder: a second one exits, because two processes on one database break each other's connections (a SimpleX server delivers a queue's messages only to the last process that subscribed to it).

## Security

- In mode `requests`, the agent asks in the agent control chat before it connects, sends a message or a file, downloads a file or leaves, and the owner approves or rejects. On a download card, `/receive` gives the agent the file, and `/forward` sends it to the agent control chat, not to the agent.
- Messages in your prompt chat can become prompts for your Claude Code session: use a chat where only you and people you trust write, see [Prompt the agent from the app](#prompt-the-agent-from-the-app).
- The server listens only on 127.0.0.1. Every request needs the token. Requests from web pages on other hosts, and requests with another `Host`, are refused (the SDK's localhost Origin and Host checks).
- Any process with the token can act as the agent. The token is stored only in the data folder, in `token`, `headers.json` and `hook-headers`; Claude Code's config and the `claude mcp add-json` command line contain the command that prints `headers.json`, and the status hooks the path of `hook-headers`, not the token.
- Names written by peers are folded to one line, with control and format characters replaced by spaces, in the terminal and in the agent control chat; there, a name that is not a valid display name is shown as code.
- Without `--owners`, whoever opens the link first becomes an owner. The link works once.
- With `--owners`, only the listed owners approve.
- After `/approve #N all`, the agent sends to that chat without asking, files included; `/revoke #N` ends it.
- In such a chat, the status message and its notes are sent without asking and without a copy in the agent control chat; each note is written to the terminal. Everyone in the chat sees in the status line the names of the agent's running tools and of its model, the names of the files Claude Code reads and edits, without their folders, and the names of its other tools and subagent types, also for its work in other chats.
- With the status hooks in scope `user`, Claude Code passes the events of every session of the user to curl, which posts them to the agent's server, with prompts, commands, file contents and tool results. The server uses only the fields named in [Status hooks](#status-hooks), keeps state only for the sessions that used the agent's tools, and logs nothing. curl ignores `.curlrc` files and proxy variables, so the events and the token are sent only to 127.0.0.1.
- Only approvers schedule prompts, and a schedule's prompts become due only while the owner who added it is an approver. `receive` returns them in `scheduled`, apart from the chat's messages.
- After `/approve #N all`, the agent sends to that chat without asking, files included; `/revoke #N` ends it. With `/copies #N off`, these messages are not copied to the agent control chat.
- After `/approve #N all`, the agent sends to that chat without asking, files included, and, from the next version, at each start the server posts there its version and the commit of its build, unless the agent is incognito there; `/revoke #N` ends it.
- The server refuses to send private keys, its own files and the files under `~/.simplex-mcp`, where agents keep their data, except the files under `files/` of its own data folder; other files the agent can read are sent after approval.
- The server refuses to send private keys, its own files, the secrets and the files under `~/.simplex-mcp`, where agents keep their data, except the files under `files/` of its own data folder; other files the agent can read are sent after approval.
- With secrets, values are protected from accidental exposure to the model provider: the server deletes `/secret` messages for all members, and the agent gets the names, not the values. A prompt injection can make the agent send a value away by reference, for example in a command that reads the file of a secret into a request to another host; this is out of scope.
- Every process of the server's user can read the secrets, and `send` and `connect` texts are not checked for their values: the approval cards show the texts.
- The agent forwards the messages of its control chat to members as an owner: a copy of a forwarded `/secret` message can remain with its delivery job in `simplex_chat.db` of the data folder until the job is done and 7 days old.
- After `/receive #N 7 all`, the agent's later downloads from that chat, of files on its XFTP servers, run without asking; `/receive #N off` ends it. A file on other servers is downloaded only by `/receive` or `/forward` of the card that names them; any other download of it fails, unless the IP address is protected.
- With `--audio`, the voice files of peers and the agent's voice texts are sent to URL, also when URL is on another computer. A URL with a user name or password is refused.
- With `--audio` and `/receive #N 7 all`, voice messages from that chat of at most 300 seconds and 1,500,000 bytes, on the agent's XFTP servers, are downloaded and transcribed without asking; the audio API decodes their audio.
- Transcripts contain the words of peers: they are marked as data, not instructions, in the agent's instructions and in the field descriptions. They are kept off cards and out of the log.
- You approve a voice reply by its text; no audio is on the card.
- With `--models`, only approvers switch the agent's model, and only to an item of `--models`; an unknown name is left out of the reply. The file of `--model-file` contains only PROVIDER/MODEL, in a folder outside the data folder; mount that folder read-only in the agent's container.
- Peers of a chat connected as the agent see its profile; peers of an incognito chat see a random profile made for that chat only.
- The agent has no address, so nobody can contact it first.
- The agent uses private message routing: the SMP server that holds a contact's queue does not see the agent's IP address.
- `SIMPLEX_MCP_INFERENCE_KEY` is sent only to the `--inference` URL; set it only for a URL on 127.0.0.1 or with https. The server reads the servers of llama-swap's models only on the URL's host, or on loopback when the URL is loopback, and refuses redirects. It shows a model ID only when the ID consists of 1 to 64 ASCII letters, digits and characters of `_.:/+@-`, and then as code; it folds the endpoint's error texts to one line.
- With `--abort-file`, anyone who can post in a chat with a status message can stop the agent's work, in all its chats, with `/abort`; the server prints each abort with its sender and chat. FILE holds only the count, readable by all users, in a folder disjoint from the data folder.

## Claude Code plugin

The plugin gives every Claude Code session its own SimpleX group. In that group you send the session prompts, approve its actions and tool use, and read its answers. One service runs all sessions that share a data folder. Peers see one SimpleX profile, `Claude`. Sessions have the tools of `serve` (`connect`, `send`, `receive`, `download`, `list` and `leave`), and `reply`, which sends files to the session's group. In mode `requests`, the service shows the [status messages](#status-while-the-agent-works) of sessions as `serve` does without the [status hooks](#status-hooks) and `--inference`: from the arrival of a message, with the labels `received, waiting for the agent`, `waiting for approval`, `using TOOL` and `thinking`, and with the notes of `send` with `live: true`. Plugin sessions get neither the labels of Claude Code's work and the deletion at the end of its turns, which come from the status hooks of `register`, nor the model labels of `--inference`. When the service stops, it deletes the status messages and waits 10 seconds for the deletions and the cancelled cards to be sent; when it starts, it deletes those left by a crash.

### Install

The plugin needs Claude Code 2.1.214 or later and Node.js 22 or later. In Claude Code:

```
/plugin marketplace add https://raw.githubusercontent.com/simplex-chat/simplex-chat/master/.claude-plugin/marketplace.json
/plugin install simplex@simplex-chat
```

Prompts from SimpleX and approvals of tool use in the group need the plugin's channel. Start Claude Code with it:

| When | Command |
|---|---|
| Until Anthropic lists the channel | `claude --dangerously-load-development-channels plugin:simplex@simplex-chat` |
| After the listing | `claude --channels plugin:simplex@simplex-chat` |
| Team and Enterprise plans | An admin adds `simplex@simplex-chat` to `allowedChannelPlugins`. |

The development flag works only in interactive sessions. Without a channel, the plugin still works, but prompts from SimpleX and approvals of tool use in the group don't. Channels need Anthropic authentication through claude.ai or a Console API key.

The plugin adds:

| Part | What it does |
|---|---|
| MCP server `simplex` | Runs `simplex-mcp session` for each session. It provides the tools, such as `mcp__plugin_simplex_simplex__connect`, and the channel. Its tool `pre_tool_use` answers the PreToolUse hook of every tool, and Claude never needs to call it. |
| SessionStart hook | Checks the Node.js version and starts the service. It opens the session and its group, and shows the Main group link while the Main group has no owner. |
| PreToolUse hook of the plugin's tools | Lets the plugin's tools run without a Claude Code prompt, because you approve them in the SimpleX app. |
| UserPromptSubmit, Stop, PostModelSwitch and PostCompact hooks | Post your terminal prompts, Claude's answers and a line for each summary of the session's context in the session's group, and keep the model and effort in its description. They run in the background. |
| PreToolUse, PostToolUse and PostToolUseFailure hooks of every tool | Post a line for each step of Claude's work when it starts, and complete it when it ends. They run in the background. |
| PreToolUse hook of every tool, of type `mcp_tool` | Calls the tool `pre_tool_use` of the session's MCP server, which allows a tool approved for the rest of the session, see [Tool use](#tool-use). It waits up to 5 seconds. |

The server and hooks run `node ${CLAUDE_PLUGIN_ROOT}/dist/cli.js`. The PreToolUse hook of the plugin's tools runs `node ${CLAUDE_PLUGIN_ROOT}/plugin/allow.js` instead. The package, a 4 MB download, bundles its dependencies and has no lockfile, so Claude Code installs nothing after unpacking it. Node.js loads the modules from the plugin's folder, so a project's `node_modules` can't replace them. The first session start downloads the SimpleX core library. Claude Code waits up to 600 seconds for the SessionStart hook.

### Shared settings

Settings shared by several machines or containers can load the plugin from a folder, without `/plugin` commands. This setup was checked with Claude Code 2.1.322.

1. On each machine or container, unpack the package into `plugin/` of a folder at the same path, here `/opt/simplex-plugins`. VERSION is the plugin's version, such as 0.3.0:

   ```
   mkdir -p /opt/simplex-plugins/.claude-plugin /opt/simplex-plugins/plugin
   npm pack @simplex-chat/mcp@VERSION
   tar -xzf simplex-chat-mcp-VERSION.tgz -C /opt/simplex-plugins/plugin --strip-components 1
   ```

   The dependencies are bundled, so nothing needs installing.

2. Write `/opt/simplex-plugins/.claude-plugin/marketplace.json`:

   ```json
   {
     "name": "simplex-local",
     "owner": {"name": "Alice"},
     "plugins": [{"name": "simplex", "source": "./plugin"}]
   }
   ```

3. Add to the shared settings:

   ```json
   {
     "extraKnownMarketplaces": {
       "simplex-local": {"source": {"source": "directory", "path": "/opt/simplex-plugins"}}
     },
     "enabledPlugins": {"simplex@simplex-local": true}
   }
   ```

   `path` is the folder's absolute path. Claude Code reads `extraKnownMarketplaces` from user settings (`~/.claude/settings.json`), from a file passed with `--settings`, and from managed settings. It loads the plugin when it starts.

4. Start Claude Code with the channel each time:

   ```
   claude --dangerously-load-development-channels plugin:simplex@simplex-local
   ```

   Settings can't replace this flag. An organization's managed settings, `channelsEnabled` and `allowedChannelPlugins`, only allow the channels that a session names with `--channels`. `--channels plugin:simplex@simplex-local` works only when `simplex@simplex-local` is in `allowedChannelPlugins`.

Each machine and each container needs its own data folder, on storage that no other one uses. Use `~/.simplex-mcp` in its own home folder, or set `SIMPLEX_MCP_HOME` in its own environment, not in the `env` of the shared settings. The service matches sessions and checks its lock by process IDs, and process IDs are only meaningful within one machine or container. Each data folder has its own Main group, whose link shows at the first session start there. To tell these groups apart in the app, set `SIMPLEX_MCP_MACHINE` in the environment of each machine or container: `SIMPLEX_MCP_MACHINE=laptop` names its Main group `Claude Control - laptop`.

### The Main group

At the first session start, Claude Code shows:

```
SimpleX MCP: scan this QR code with the SimpleX app and tap Join to supervise your agents in "Claude Control":
<QR code>
https://smp4.simplex.im/g#...
```

The first start downloads the SimpleX core library and creates the group, so the link can take a minute. If Claude Code ends before then, the service keeps starting, and the next session start shows the link.

Only you see this link: it never enters Claude's context or tool results. Scan the QR code, or paste the link into the SimpleX app, 7.1.0 or later, and tap Join. The group "Claude Control" appears, and Claude writes: `I'm Claude. Each Claude Code session has its own group, and you get its invitation here. /sessions lists the sessions; /join N joins the group of session N.`

- The first person who joins by the link becomes an owner, and the link is deleted. Until someone joins, the link stays the same across service restarts.
- Everyone in the Main group is trusted with all sessions, whatever their role there. They can join any session's group with `/join N`, without approval, as admins. There they can prompt the session and approve its actions.
- The group's welcome message says so: `Everyone in this group can join the groups of your Claude Code sessions, prompt Claude and approve what it does. Invite only people you trust with that.` When someone joins, Claude posts `NAME joined. They can join the group of any Claude Code session here, prompt Claude and approve its actions.`
- Claude sets the short description and the welcome message when it creates the group. Later it changes only the group's name and commands, so the texts and preferences that owners set stay.
- Owners can add people with a group link of their own or by inviting their contacts.
- Observers can't send messages in the Main group, so they can't use `/join N`.
- Owners of the Main group are invited to every session's group as owners. They get each invitation in the Main group.
- `/sessions` replies with the text of the list of sessions, below. `/join N` sends you an invitation to the group of session N, as an admin, or as an owner if you own the Main group. Another one-line message that starts with `/` gets `Unknown command.`, as a new message without a quote.
- Leaving the Main group, being removed from it, or getting another role there changes nothing in the session groups.
- When no owner remains, the service creates a new link. In a terminal, `npx -y @simplex-chat/mcp@VERSION link` prints it with a QR code. While the group has owners, the command names them instead.

After the first owner joins, Claude posts the list of sessions in the Main group, after its greeting:

```
🟠 3 · simplex-chat: working
🔵 6 · website: waiting for you, 2 approvals
🔵 7 · docs: waiting for you in Claude Code
🟢 8 · app: idle
⚪ Not running: 5 · bio, 4 · notes, 2 · openclaw, 1 · first
```

- A session is running while its Claude Code runs, or while cards wait for a decision in its group. Each running session has a line, by number:
  - `🔵 N · NAME: waiting for you, K approvals`: K cards wait in the session's group. A tool-use card counts only while Claude Code shows its permission dialog, or while Claude Code's state can't be read.
  - `🔵 N · NAME: waiting for you in Claude Code`: Claude Code waits for an answer in its own window, such as a dialog.
  - `🟠 N · NAME: working`: Claude Code works on a turn, or a subagent works for it.
  - `🟢 N · NAME: idle`: Claude Code waits for the next prompt, also while a background command runs.
  - `▶️ N · NAME: running`: Claude Code's state can't be read.
- The last line names every other session of the data folder, newest first. Without any session, the list is `No sessions.`
- The states are Claude Code's own: the plugin reads them from Claude Code's record of its process, `sessions/PID.json` in `CLAUDE_CONFIG_DIR` or `~/.claude`.
- The list is one message. When it gets too long, the last line keeps the newest names that fit and ends with the count of the others, such as `190 older sessions`. The lines of running sessions are never cut.
- Claude edits the list within 2 seconds of a change. An edit alerts nobody. The list moves up as messages, such as invitations to new sessions' groups, are posted after it.
- SimpleX lets Claude edit a message only within 24 hours. When the list is 23 hours old, Claude posts a new list and deletes the old one for everyone.
- When the service stops, Claude edits the list once more, with every session in the last line.

To replace the owners, run `npx -y @simplex-chat/mcp@VERSION reset-owners` in a terminal. It replaces the Main group at once, without the owners' approval:

- The old Main group is deleted for all its members, and a new one is created. The session groups don't change: members who joined them from the old Main group keep their roles.
- In a terminal, the command prints the new link with a QR code. Run without a terminal, for example by Claude's tools, it shows no link and asks you to run `link` in a terminal.

VERSION is the plugin's version, as in `npx -y @simplex-chat/mcp@0.3.0 link`. The plugin's messages print these commands with the right version. The same command with a newer version would replace the plugin's service.

### Sessions and their groups

- Each session has a number, starting at 1, which is kept across service restarts. Its group is named `N · NAME`. NAME is the session's name from `claude -n NAME` or `/rename`, otherwise the project folder's name, otherwise `Claude Code`. A new name shows from the next session start.
- The group is created at the session start, and the Main group's owners are invited to it as owners. An owner who joins the Main group later is invited to the groups of the open sessions.
- The group's members are Claude, the Main group's owners, members who joined with `/join`, people whom members added directly, and `User`. People added directly get the role that the adding member set. `User` is a second profile of the service that posts your terminal prompts.
- Claude's profile has the peer type `agent`, so the app shows Claude with the agent placeholder. `User`'s profile has the peer type `human`, which the service sets at each start. In a session group that you joined while `User` had the peer type `agent`, as in 0.3.0, the new peer type reaches your app with `User`'s next post there, and the app shows `updated profile` from `User` once.
- Rights follow the member's role in the group, however they joined:

  | Role | Rights |
  |---|---|
  | owner, admin | prompts, the group's `/` commands, approvals of actions and tool use |
  | moderator, member, author | prompts; each of their `/` commands gets `Only admins and owners of this group can send / commands.` |
  | observer | none: the app can't send their messages |

  `User`'s messages are never prompts, commands or approvals.
- When someone joins or their role changes, Claude posts their rights. For an admin the post is `Bob is an admin and can send prompts to Claude and the group's commands, and approve tool use. You can change role to observer to stop it: /'observer #G ID'`, with the group's and the member's IDs. For a moderator, member or author it is `NAME is a ROLE and can send prompts to Claude; the group's commands and approvals need the role admin or owner.`, followed by the same command. The posts for an owner and for an observer have no command: `NAME is an owner and can send prompts to Claude and the group's commands, and approve tool use.` and `NAME is an observer and can't send messages.` Claude posts nothing for `User`.
- `/observer #G ID` makes that member an observer, and Claude replies `NAME is an observer and can't send messages.` Admins and owners can use it on a member whose role isn't above their own. Standing approvals that the observer gave, of all messages to a chat or all files from it, stop counting while they aren't an admin or owner, and count again if they become one. For a member who was invited and hasn't joined yet, Claude replies `No member ID in this group.` That member joins with the role of the invitation, which can be changed after the join.
- The group's description is `The Claude Code session N. Members send it prompts; admins and owners also send the group's commands and approve tool use.`, set when the group is created. Its short description shows the model and effort, see [Model and effort](#model-and-effort).
- The group has the commands of the agent control chat of `serve` (see [What the owner sees](#what-the-owner-sees)), without a session prefix, such as `/approve 7`, `/receive @3 9` and `/leave #2`. It also has `/give CHAT N`, `/observer #G ID`, and `/approve N all` and `/revoke TOOL` for tools, see [Tool use](#tool-use). `/help` there lists `/approve N all`, `/revoke TOOL`, `/give` and `/observer` before the line of `/help COMMAND`; `/help approve`, `/help revoke` and `/help list` also explain the approvals of tools, and `/help give` and `/help observer` explain the last two commands.
- A chat belongs to the session that connected it, even when the other side accepts after `connect` returned. The tools of other sessions don't show it. Their `connect` to its address returns `The chat of LINK belongs to another session; an admin or owner of that session's group can give it to this session with /give.` For example, `/give @3 2` in the group of the session that owns chat @3 gives it to session 2. The chat's [schedules](#scheduled-prompts) are moved with it, and their due prompts are dropped. A schedule's prompts become due only while the member who added it is an approver in the group of the session that owns the chat. Each group has its own member IDs, so the prompts of a schedule added in another session's group become due only after the chat is given back to that session.
- The group has the commands of the agent control chat of `serve` (see [What the owner sees](#what-the-owner-sees)), without a session prefix, such as `/approve 7`, `/receive @3 9` and `/leave #2`. It also has `/give CHAT N`, `/observer #G ID`, and `/approve N all` and `/revoke TOOL` for tools, see [Tool use](#tool-use). `/secret`, `/unsecret` and `/secrets` get `Secrets are off: the server runs without --secrets-folder.`, and a one-line `/secret` of an admin or owner is deleted for all members first.
- A chat belongs to the session that connected it, even when the other side accepts after `connect` returned. The tools of other sessions don't show it. Their `connect` to its address returns `The chat of LINK belongs to another session; an admin or owner of that session's group can give it to this session with /give.` For example, `/give @3 2` in the group of the session that owns chat @3 gives it to session 2.
- All commands also work in the group of a closed session. A `/revoke` sent while the service is stopped applies when the service starts again.
- `claude --resume` and `claude --continue` reopen a session with its group and chats. `/clear` keeps the session's group, because the service links the new session IDs of a Claude Code process to that process's session. A fork made with `claude --resume ID --fork-session` continues in the group of the resumed session. After `/resume` of another session inside a running one, prompts and answers go to the running session's group. A later `claude --resume` of that other session reopens its own group and chats.
- A session closes 10 minutes after its Claude Code session ends, once the expiry times of its actions have passed. The service stops 10 minutes after the last session closes. The next session start or tool call starts it again.
- A non-interactive run, `claude -p` or the Agent SDK, gets no session and no group. Its hooks post nothing, and its SimpleX tools return `This Claude Code run is not interactive, so it has no SimpleX group; the SimpleX tools work in interactive sessions.`
  - The plugin recognizes such a run by `CLAUDE_CODE_ENTRYPOINT`. Claude Code sets it to `sdk-cli` for `claude -p` and to `sdk-ts` or `sdk-py` for the Agent SDK. It also uses `CLAUDE_CODE_SESSION_ATTENDED`, which Claude Code sets to `0` for the hooks of a run nobody attends.
  - A `claude -p` run started from a session of the VS Code extension or the desktop app keeps that session's entrypoint, and its MCP server gets that session's attended value. For such a run, the SessionStart hook marks its Claude Code process, and the service refuses that process's session server when it registers within 10 seconds. Its tools then return the same text.

### Prompts and answers

With the channel, a message from a member of a session's group, other than `User`, is a prompt to the running session, like one typed in the terminal.

- A file attached to the message is saved without approval to `_sessions/<session ID>/files/group/` in the data folder, and Claude gets its path. When the file fails or doesn't arrive within 10 minutes, Claude replies `The file of this message was not received: ERROR`, and the session gets the text.
- When you reply to a message, Claude gets your text after a line `> ` with the quoted message: its type when it is not a text (link, image, video, voice, file, report or chat), such as `> image: IMG_2041.jpg` or `> video`, and the first line of its text, up to 200 characters, with `…` when it continues. Claude also gets the quoted message's ID and sender, and takes that line as data, not as your prompt.
- When you react to a message, Claude gets your emoji as a reply to that message: after its quote line, with its ID and sender, as for a reply. Claude gets the eight emoji of the apps: 👍 👎 😀 😂 😢 ❤ 🚀 ✅. To any other reaction, Claude replies `This reaction is not supported.` once for each member and message, and the session gets nothing. A video or a voice message without text is quoted with its file name, such as `> video: ride.mp4`. Claude answers as for any prompt, with the reacted message in place of the prompt. A reaction is never a command or an approval: a 👍 on a card leaves the card pending. Reactions of observers, a removed reaction and reactions in the Main group reach no session. Owners turn reactions off with the group's preference "Message reactions".
- Prompts reach the session in the order they were sent. A later prompt waits for the file of an earlier one.
- Every message from a member who can send is a prompt, including one that starts with `!`. Only one-line messages that start with `/` are not prompts, see below. Claude Code runs no command from a prompt. Claude gets the text, and runs shell commands only as tool use, such as Bash, under Claude Code's permissions, see [Tool use](#tool-use).
- A one-line message that starts with `/` is a command of the group, such as `/approve 5` or `/list`. From an admin or owner, an unknown command gets `Unknown command.` From a moderator, member or author, any such message gets `Only admins and owners of this group can send / commands.`, `/help` included, and it reaches neither the commands nor Claude. `/help help` in the group states this rule.
- A one-line message that starts with `/` is a command of the group, such as `/approve 5` or `/list`. From an admin or owner, an unknown command gets `Unknown command.`, as a new message without a quote. From a moderator, member or author, any such message gets `Only admins and owners of this group can send / commands.`, and it reaches neither the commands nor Claude.
- While the service runs and the session's Claude Code doesn't, Claude replies `Session N is not running.` It also replies so when Claude Code ends while a prompt's file arrives. For 30 seconds after the service starts, a prompt waits for its session to attach again, so a prompt sent while the service was stopped still arrives.
- When a prompt that a person typed, or a prompt from the group, starts a turn, Claude posts an empty live message at once: the apps show it under the LIVE header, with typing dots. It replies to the prompt when the prompt came from the group. The turn's first post fills it: the first line of Claude's text, a step, or the answer. If someone deletes the empty message, that post comes as a new message. A prompt that Claude Code adds to the running turn, a `/` command and Claude Code's own prompts get no such message.
- Claude's text appears live as Claude writes it, also its remarks between steps, which the terminal shows. Each text of Claude, from its first line, is a live message that grows by whole lines, updated at most every 3 seconds, and completed when Claude ends it, without an edited mark. A text longer than one message continues in a new live message.
- Claude's final answer of a turn is its last text. When Claude Code streams no text, as in versions without the MessageDisplay hook, Claude posts the answer at the end of the turn, in the turn's empty live message when no other post filled it. When the turn's prompt came from the group, the answer replies to it, even when the next prompt arrives while Claude answers. Claude Code can add a prompt that arrives during a turn to that turn. Then an answer posted as a new message replies to the last message from the group added to the turn.
- After a non-blank answer, Claude posts one line in small print, `✅ Done in DURATION`, with the time from the prompt to the end of the turn, such as `✅ Done in 2 min 14 s`. After a blank answer, the line fills the turn's empty live message instead, without an alert. A turn that ends without an answer, as after an interruption or an error, gets no line: its live texts are completed as they are, and its empty live message gets `⏹️ Stopped` at the end of a later turn, or when the session ends. A text or an empty live message still live 23 hours after it was sent is completed the same way, because SimpleX allows edits of a message for 24 hours.
- Claude's replies to a prompt, its answer included, are posted without the quote when the prompt was deleted or the quote makes the message too large. An answer is also posted without the quote when the quote leaves no room for its first character.
- While the plugin is enabled, Claude Code shows Claude's text in the terminal by whole lines, after the plugin's MessageDisplay hook passes each batch of lines to the service, about 0.1 s later.
- `User` posts only the prompts a person typed, also one typed while Claude works. `User` skips commands such as `/compact`, empty prompts such as one with images only, and prompts made only of the plugin's `<channel source="plugin:simplex:simplex" ...>` tags. `User`'s posts are never commands of the group, even a part that starts with `/`.
- Claude posts one line for each of Claude Code's own prompts, without a quote. The line starts with an emoji for the prompt's kind:
  - a notification of a background task, a subagent, a workflow or a monitor: its summary, such as `✅ Background command "Build" completed (exit code 0)`, and the first line of its event, when it has one, such as `📡 Monitor event: "Builds": serverpg exit 0`. The emoji is ✅ for a task that completed, ❌ for one that failed and ⏹️ for one that was stopped or killed. A notification without a status gets 📡 when it has an event, as a monitor event or the end of an artifact watch, and 🔔 otherwise. A notification in plain text, such as `3 background agents were stopped by the user: ...`, gets 🔔 and its first line;
  - a message of another Claude session: `✉️ Message from NAME: ` and its first line;
  - a message of a teammate in an agent team: `✉️ Message from NAME: ` and its summary, or its first line without one. A prompt with several teammate messages gets the line of the first one that is not an idle notification, and a prompt of idle notifications only gets no line;
  - the report of a subagent: `📋 Report from subagent NAME: ` and its first line;
  - a scheduled prompt: `⏰ Scheduled prompt: ` and its first line;
  - an automatic prompt, such as the one after a usage limit resets: `🔄 Automatic prompt: ` and its first line;
  - another kind: `⚙️ Claude Code: ` and its first line.

  A line over 200 characters is cut there and ends with `…`. Prompts typed in the terminal and channel prompts get no line. When one of Claude Code's own prompts starts a turn, Claude's answer to it is posted without a quote.
- When Claude Code summarises the session's conversation, Claude posts one line without a quote, once the summary has replaced the conversation. A summary made by Claude Code at the context limit or in an idle session gets `⚙️ Context summarised automatically: 968K → 35K tokens`, and one after `/compact` or "Summarize from here" gets `⚙️ Context summarised on request: 399K → 25K tokens`. The numbers are the size of the context before and after the summary. The summary of a subagent's conversation posts nothing. Claude Code writes a summary made during a turn to the transcript with its next request, so the line can follow such a summary by a few seconds. It writes the summary of `/compact` and of an idle session at once.
- Claude posts one line for each step of its work, a tool use of the main conversation, when the step starts:
  - Bash: `💻 ` and the command's description, or its first line without one, such as `💻 Run the tests`;
  - Read, Edit and Write: `📖 Read PATH`, `✏️ Edit PATH` and `📝 Write PATH`, with PATH relative to the session's folder when it is inside it;
  - Grep and Glob: `🔍 Grep PATTERN in PATH` and `🔍 Glob PATTERN in PATH`, with PATH as for Read, and without ` in PATH` when the search has no path;
  - WebFetch and WebSearch: `🌐 Fetch URL` and `🔎 Search QUERY`;
  - Agent: `🤖 Agent: ` and the agent's description;
  - another tool: `🔧 TOOL`, and `: ` with the first of its description, query, URL, file path, path, pattern and command that is not blank.

  The line is a live message while the step runs. When the step ends, Claude completes it with ✅, with ❌ for a failure or ⏹️ for an interruption, and with the step's duration, such as `💻 Run the tests ✅ 2.0 s`. Claude Code reports the end of a command or an agent run in the background when it is started, so its line is completed with `(in background)`, such as `💻 Run the tests (in background)`; its completion is posted later as the line of its notification. The completed line has no edited mark. A step refused at approval has no end; its line is completed without an outcome when the turn ends, when a prompt starts the next turn, or when the session ends. A line still live 23 hours after its start is completed without an outcome, because SimpleX allows edits of a message for 24 hours; the step's later end is posted as a new line. The text of a step is cut at 200 characters. Tool uses of subagents and of this plugin get no line.
- Claude marks the turn's empty live message, its texts and answers, the lines of steps, and the lines of Claude Code's own prompts and of summaries of the context as silent, and the line after the answer as important, so a turn alerts once, at its end. A SimpleX app that supports silent messages shows them without an alert; other apps alert for them as for any message.
- A text too long for one message is posted as several messages, each as long as a message can be, measured as for `send`. The first part of an answer carries the quote. A part without a quote holds up to 14,962 Latin, 7,481 Cyrillic or 4,987 CJK characters. A prompt or an answer over 250,000 characters is cut there and ends with `…`.
- The `reply` tool sends a file to the group at once, without approval, with its text as the caption, under the file rules of `send`. It returns the file's message as `item`. A caption too long for a message with the file's preview is refused with `text: too long for a message with this TYPE; shorten it.`

### Group preferences

Owners set the group's preferences in the app, and SimpleX enforces them. Claude changes only the group's name, short description and commands, starting from the profile in the chat database, and turns full deletion on when it is off, as in the agent control chat of `serve` (see [Deletion of `/secret` messages](#deletion-of-secret-messages)).

- While "Files and media" is off, `reply` returns `Files and media are off in the group of this session; its owners can turn them on in the group's preferences.` The cards of sends and downloads of images and videos are posted as text, without the preview, and the sends and downloads still work. `/forward` gets `Action N: Files and media are off in the group of this session; its owners can turn them on in the group's preferences.`
- When another preference refuses one of Claude's posts, Claude posts a text that names it instead:
  - For an answer with a SimpleX link: `Claude's answer was not posted: SimpleX links are off in the group of this session; its owners can turn them on in the group's preferences.`
  - For the line of one of Claude Code's own prompts or of a step: `Claude's note was not posted: ...`
  - For a text of Claude: `Claude's text was not posted: ...`, once for the text. Its live message keeps the text posted last. When the text is the answer, the answer is not posted again.
  - When the refused post would fill the turn's empty live message, the text that names the preference fills it.
  - For a terminal prompt: `User's prompt was not posted: SimpleX links are not allowed for User in the group of this session; its owners can allow them in the group's preferences.` This also happens while the preference allows SimpleX links only for roles above `User`'s role, which is member unless the owners change it.
  - For a card that waits for approval, or for `/forward` of a voice message: `Action N: ...`. The action then fails.
  - `reply` returns the same text without the prefix `Claude's answer was not posted: `.
- After an action is decided, the text names the outcome and the card, and the action stays as it was:
  - The card of an action performed in mode `host` or under a standing approval is posted after the action, except that of a done send under a standing approval to a chat with `/copies CHAT off`, see [Approval of all messages to a chat](#approval-of-all-messages-to-a-chat). When that post is refused, Claude posts `Action N was performed; its card was not posted: ...` or `Action N failed; its card was not posted: ...`.
  - When the edit of a card with its outcome is refused, Claude posts `Action N was performed; its card was not updated: ...`, with `failed`, `was rejected`, `expired` or `ended` in place of `was performed`.
  - A card posted or edited when the service stops gets `Action N: ...`.

### Tool use

With the channel, Claude Code's permission prompts, such as the one before a Bash command, are also posted in the session's group:

```
Claude: I want to use `Bash`: `List files`
        `ls -la`
        /approve 5, /approve 5 all or /reject 5
```

`/approve 5` allows the tool use, and `/reject 5` denies it. The dialog in the terminal stays open, and the first answer, in the terminal or in the group, applies. A card that is still pending gets `Ended.` when its turn ends, when a prompt starts the next turn, or when the session ends. A prompt that Claude Code adds to the running turn leaves the card pending. The preview of the tool's input is shortened to fit the card.

`/approve 5 all` allows the tool use and approves its tool for the rest of the session. Claude replies `` Approved `Bash` for the rest of this session by Alice. /revoke Bash ``.

- Later uses of the tool run without a card and without the dialog in the terminal. Each use in the main conversation still has its line in the group.
- Claude Code still asks where its own rules require it: for a matching ask rule, a write to a protected path such as `.git`, `.claude` or `.bashrc`, in plan mode, and when the classifier of auto mode asks. Such a use gets a card with the line `` `Bash` is approved for this session, and Claude Code still asks for this use. ``, and no `all` command. With Claude Code before 2.1.299, an approved tool also runs writes in plan mode.
- The approval covers the uses of subagents too. Other pending cards of the tool stay pending.
- It counts while the member who gave it is an admin or owner of the group, and counts again if they become one again.
- `/list` shows it after the chats, as `` `Bash`  approved for this session by Alice ``. `/revoke Bash` ends it with `` Uses of `Bash` need approval again. ``
- It holds for the Claude Code process of the session, also after that process attaches to the service again. It is dropped when another Claude Code process opens the session, when the session closes, and when the service restarts. The service tells processes apart by their ID and their start time, read on Linux and macOS. On other systems, and when the start time can't be read, the approval also ends when the same process opens the session again, as after a failed call. While two Claude Code processes have the session open, only the one that opened it last gets the approvals. It also ends when the session's group is lost: when the group is deleted, or Claude is removed from it or is no longer its owner. The terminal shows nothing at the approval, and Claude Code's `/permissions` doesn't list it, because the plugin keeps it.
- `/approve N all` of another kind of action gets `Action N is not a tool use.`
- While the plugin's MCP server is not connected, as when it is disabled in `/mcp`, left out by `--strict-mcp-config` or failed to start, Claude Code reports the PreToolUse hook error `MCP server 'plugin:simplex:simplex' not connected` at each tool use, and the tool use goes on as without the plugin.

### Model and effort

The short description of a session's group shows `MODEL · effort LEVEL`. The model is updated at the session start when Claude Code passes it, at each model change, and at the end of each turn from the model of the answer in the session's transcript. So the model shows from the first answer, also in a resumed session. The effort level is updated at the end of each turn. Changes made in the terminal, or from the Claude app by Remote Control, show the same way.

### Options

Set in `/plugin` > Installed > simplex > Configure options:

| Option | Default | Values | Effect |
|---|---|---|---|
| `main_group` | `Claude Control` | a valid SimpleX display name | The name of the Main group. With `SIMPLEX_MCP_MACHINE` set in the environment, ` - ` and its value follow, up to 50 characters in all. |
| `approve` | `requests` | `requests`, `host` | The approval mode of the sessions. |
| `expire_hours` | `23` | 1 to 23 | How long an action waits for a decision, see [Expiry](#expiry). |

A changed option applies from the next session start, and so does a changed `SIMPLEX_MCP_MACHINE`. The service renames the Main group to the new name. An invalid value is shown at the session start, and every tool call of that session returns the same text, for example `Plugin option expire_hours: hours from 1 to 23 expected, got 48. Change it in /plugin > Installed > simplex > Configure options.` or `SIMPLEX_MCP_MACHINE: a valid SimpleX name, such as mylaptop expected, got my.laptop. Change it in the environment of this machine.` A value of `approve` other than `requests` and `host` can come only from a hand edit of the settings. It is shown only at the session start, and the session's tools run in mode `requests`.

In mode `host`, a session asks in its group to run in that mode: `I want to run in mode host: Claude Code asks you before each of my actions, and they are performed without approval here.`, with `/approve N or /reject N`. It asks at its start, or when the group gets its first admin or owner other than Claude, by a join or a role change. The session runs in mode `host` after an admin or owner approves, and keeps the mode across restarts, in the custom data of its group in the chat database. After a rejection, it asks again only after a session start with `requests`. A change back to `requests` applies at the next session start without a card, and a pending card then gets `Ended.`

### Plugin data folder

`~/.simplex-mcp`, or the folder in `SIMPLEX_MCP_HOME`:

| Path | Content |
|---|---|
| `service.token` | The access token of the service's socket, readable only by the user. |
| `service.lock` | The PID of the running service. |
| `service.sock` | The service's Unix socket. On Windows it is the named pipe `\\.\pipe\simplex-mcp-<hash>`. For a path over 103 bytes, it is `simplex-mcp-<hash>.sock` in the system temporary folder. |
| `service.log` | The service's log, emptied when it is over 10 MB, at a service start and every hour. |
| `titles.json` | The titles of new sessions whose session server had not registered when the service stopped. The next start reads and removes it. |
| `_hub/` | The databases of the profile `Claude`: the Main group, the session groups and the chats with peers. |
| `_user/` | The databases of the profile `User`. |
| `_sessions/<session ID>/` | `session.json` with the session's number, name, project folder, group, a rejected mode `host`, model, effort level and other session IDs. `files/` holds the files of its chats and of its group. |

The folders, and the files the service writes, are readable only by the user, except on Windows. One service runs per data folder. These names contain a dot or start with `_`, which agent names of `serve` can't, so the folders of agents in `~/.simplex-mcp` never collide with them.

### Plugin recovery

| Situation | What happens, what to do |
|---|---|
| The session start shows an error | The error follows `SimpleX MCP:`, with details in `service.log`. The session starts without the service, and the next tool call tries again. |
| Node.js is older than 22 | The session start shows `SimpleX MCP needs Node.js 22 or later; found <version>.` |
| Claude Code ends before the session start shows the link | The service runs apart from Claude Code and keeps starting. The next session start shows the link. |
| The service stopped during a call | The call returns `The SimpleX service stopped; repeat the call.`, and the next call starts the service again. The actions of the stopped service are lost, and a repeated call creates a new action. |
| The service stopped between calls, or its start failed | The session starts it again and attaches after 1 second. While that fails, it retries at intervals that double up to 30 seconds. |
| A session starts while the service stops | The new service waits up to 15 seconds for the stopping one to release its lock, and then starts. |
| `service.lock` names a running process that is not the service | After 15 seconds, the session start shows `SimpleX MCP: Another service uses FOLDER (PID N). If no service runs, remove FOLDER/service.lock.` |
| A newer plugin version starts | It stops the older service and starts its own. When the two versions can't talk to each other, tool calls return `Restart Claude Code to update the SimpleX plugin.` |
| Sessions of plugin 0.3.0 run after an upgrade | Their session servers connect to the new service with the old default name, `SimpleX MCP`, and the service renames the Main group to it. The group keeps the name of the last session that started or connected, and its members see `group profile updated` at each change. Once those sessions end, the next session start sets `Claude Control`. |
| A session's group has no admin or owner but Claude | Its actions are refused with `The group of this session has no admin or owner to approve actions. Members of the Main group join it by /join N there; without members, run npx -y @simplex-chat/mcp@VERSION link in a terminal.` |
| A session's group was deleted, or Claude lost its owner role there | The session's actions fail with `The group of this session is lost. Start a new Claude Code session.` |
| The Main group was deleted, or Claude lost its owner role there | The service logs `The Main group is lost. Run npx -y @simplex-chat/mcp@VERSION reset-owners in a terminal.` |
| The wrong person joined by the link | Run `npx -y @simplex-chat/mcp@VERSION reset-owners` in a terminal. |
| The QR code does not scan | Run `npx -y @simplex-chat/mcp@VERSION link` in a terminal, or paste the link into the app. |

### Plugin security

- The service listens only on its local socket, in a folder only the user can read, and every request needs the token. On Windows the named pipe has no folder permissions, so the token is the only check.
- The service gets only the environment variables it uses: the paths of the home, temporary and cache folders, the locale and the time zone, the proxy and certificate settings, `NODE_OPTIONS`, the `SIMPLEX_` variables and the plugin options.
- Everyone in the Main group is trusted with all sessions. In a session's group, rights follow the member's role, see [Sessions and their groups](#sessions-and-their-groups).
- Claude Code runs no command from a message of a session's group, even one that starts with `!`. Claude gets the text, and its tool use follows Claude Code's permissions.
- Only you get the Main group link. The session start shows it, and `link` prints it in a terminal.
- A session runs in mode `host` only after an admin or owner approves it in the session's group.
- A tool approved in the session's group runs without the terminal's dialog, and Claude Code's own rules still apply to it. Only an admin or owner approves it, for the rest of the session.
- Claude's files reach the group only while its preference "Files and media" allows them. Owners set it, and SimpleX enforces it. A post that a preference refuses is replaced by Claude's text that names the preference.
- Any program running as you on this computer can replace the Main group and its owners at once, without their approval, with `reset-owners` or a request to the service.
- The plugin refuses to send private keys and the files of its data folder, except the files under `files/` of the session's own folder.
- Peers of a chat see the profile `Claude`, or a random profile in a chat connected incognito.
- These protections hold against Claude's tools, but not against a shell running as the same user. Such a shell can read the data folder, with the token and the chat databases, and send requests to the service.
