# Grok Bot, Claude Code, and DeepSeek Harness: one result file that saves most of the tokens



I keep a handful of Grok bots across a few projects: a game, some small SEO sites, a few MCP tools of my own. They plan and schedule. The actual work, writing code, running tests, clicking through a browser, goes to two agents on my Mac: dsh in a sandbox, and Claude Code via `claude -p`.

Heavy work on the laptop, coordination in the cloud. The seam between them was the problem. Bots bill by the token, and a long context is both expensive and dumber. The first version of the handoff was me copy-pasting: I would run dsh or Claude Code myself and paste the terminal output back into the chat. The delay was tolerable. The cost was not. That block of output sat in every later turn, and the bot would read logs and think and slowly lose track of what it had said ten minutes earlier.

So we settled one rule: whoever does the work writes a result file, and nobody carries logs around. The file sits on disk, so the bot reads the parts it needs and the rest stays out of the context, and the executor does not have to wait for someone to ask.

{{< image src="/images/agent-handoff-loop.svg" alt="The handoff flow between the cloud bot and the Mac agents" width="480" caption="The job goes down, the result goes to disk, `dsh-wait` prints three lines, and only `ok=` and `summary=` travel back to the bot." >}}

## The result file

Fixed field order: `ok`, `summary`, `task`, `finished_at`, `changed`, `next`, `detail`. Failures write one too, with `ok` false and a summary of where it got stuck. No secrets in the file, ever.

dsh writes to `.dsh-results/<task-id>.json` inside the workspace. I first tried `~/.dsh-results` and learned the sandbox cannot write outside the workspace; that idea died on the spot. Claude Code writes to `.claude-results/<task-id>.json`, and that directory is in `.gitignore`. Both write a temp file and `mv` it into place, so the coordinator never reads half a JSON document.

With a file in place, the next thought was obvious: let the bot wait for it.

## The wake-up that never came

I put a small script on the Mac, `dsh-wait`. Every 15 seconds it checks whether the result file exists. When it does, it prints three lines, `ok=`, `summary=`, `file=`, and exits. Otherwise it times out. The plan was that the bot would push the script into the background and end its turn, and the Mac would wake it when the work finished. I believed it because cloud background commands do send that callback. I carried the same assumption over to a local shell without checking whether the Mac has any such mechanism.

It does not. A background shell on the Mac does not wake the bot when it exits. Only the cloud's own background commands get that callback. I learned this twice. During a competitor research task the result file had been sitting there for half a day while the bot slept, and I only found out because I asked. Same story on a playtest run. The work was done. Nobody reported it. Whatever tokens the background wait saved were paid back with interest.

So the protocol followed the local rule: if a shell job on the Mac does not call back, no step may wait for a callback.

## Stay in the turn and wait

The fix was to not end the turn. The bot starts `dsh-wait` and then blocks in AwaitShell, matching `^ok=`. A round runs about 13 minutes. If nothing has landed, it waits another round. When the line appears it reports `ok` and `summary` and nothing else. Waiting adds nothing to the context.

Then the second ceiling: a Grok bot turn gets cut at roughly 50 minutes. Any long task killed the bot mid-wait, and the previous thirteen minutes were gone.

## Hand off every 40 minutes

The workaround is a relay. At around 40 minutes with no result, the bot sets a one-shot reminder for about 5 minutes out and ends the turn quietly. When the reminder fires it deletes the reminder first, then keeps waiting. One small wake every 40 minutes stays clear of the 50-minute hard stop, so a long job can keep going for hours, one relay at a time. The extra cost is one more wake per relay.

One parameter worth changing: `dsh-wait` defaults to a 30-minute timeout, which is too short for real work. Long jobs get 2 hours.

## The expensive way

For a while another bot took the other road. It ran `nohup claude -p`, then read the progress file every half hour and posted progress with screenshots. It looked busy. It was the most expensive thing in the system: every half hour, a large file got dragged back into the context.

An audit later made the difference plain. Under the same skill, the reference bot (006) followed the chain end to end. Another (005) got almost every step wrong, starting with nohup plus progress polling. So the rules became blunt: no polling progress files after nohup, one `dsh-wait` per task, no scheduled reads of progress.

## Only ok and summary

The context discipline came down to one sentence: `detail` stays on disk. Handoffs between bots, reports to me, and returns from the executor carry `ok`, `summary`, and the path to the result file. `changed` can come along if it is short. If I need specifics, I pull them out with `plutil` and `rg`. Nobody `cat`s the whole JSON, and nobody pastes a log into the chat.

`summary` later got a hard limit of 200 characters. One task had stuffed five hundred-odd characters in there, and reading it was the same as pasting a log: the summary was a report. Two hundred characters is enough to say whether the job succeeded without making anyone scroll. Overflow goes in `detail`.

## The Notion ledger

Enough tasks and you need a place to see progress. Before starting, create or reuse a row in Notion, set it to in progress, and record the source bot, the project number, and the full prompt. On success, flip it to done and put the summary in the notes. On failure or timeout, leave the status alone and write the failure reason in the notes. Leaving it alone means the next time I open the table I can see it never finished. Marking it done would bury it.

Two traps. Source and number are select columns, and a value that is not in the table cannot be written at all, so the schema has to change first. And there is a column called `note` which is my own progress journal. The bots may not touch it. They get the notes property.

## Small permissions

Claude Code runs with `--permission-mode acceptEdits` by default: it may edit files, not run arbitrary commands. When it needs to build, test, run git or drive the browser, it gets a minimal `--allowedTools` list. `--dangerously-skip-permissions` and bypassPermissions are out unless I say so in the conversation.

At launch the full command line, permission mode and allowed tools included, goes into the first line of `.claude-results/<task-id>.log`, so that later I can answer what was switched on. The prompt itself lives in `.prompt.md`, which keeps the command line from becoming a novel.

## One browser

Browser testing on the Mac is ego lite (`/Applications/ego lite.app`, CLI `~/.local/bin/ego-browser`). The prompt spells out the steps: run `ego-browser skill` first, name the session after the task id, prefer the local preview, close the page when done. No Playwright, no Puppeteer, no headless Chromium, and no falling back to them when ego fails. On the Claude side, `Bash(ego-browser:*)` is the only extra permission it needs.

## When the Mac is unreachable

When the VPN is flaky the bot cannot reach the Mac. One of them decided to check every 15 minutes, which meant dozens of wake-ups overnight, nearly all of them wasted. The rule now: if it cannot connect, check at most every 2 hours. If I get impatient I poke it myself. Once it reconnects, it goes back to one `dsh-wait` per task.

## Two skills

The rules drift when they live in a chat, so they ended up as two skill files: `dsh result json` and `claude result json`. The second opens by saying the result file, the waiting, and the Notion row all follow the first; only the launch differs.

Three templates get copied out of them most often. The result file block goes at the end of every task prompt:

```text
When done, write `.dsh-results/<task-id>.json` at the
workspace root with exactly these top-level keys, in this
order:
- `ok`: true if the task fully succeeded, false otherwise
- `summary`: one sentence, hard limit 200 characters, what
happened or why it failed
- `task`: the task id
- `finished_at`: ISO 8601 time with offset
- `changed`: list of files, commits, or resources changed
- `next`: what a human or caller should do next, or empty
- `detail`: the full output, logs, diffs, and reasoning
Write it to a temp file in `.dsh-results/` first, then `mv`
it into place. Write the file even on failure. Never put
secrets, tokens, or keys in it.
```

For Claude Code the path becomes `.claude-results/<task-id>.json`.

If the task touches a browser, one more block:

```text
Browser testing: use only ego lite through the `ego-browser`
CLI.
First run `ego-browser skill` and follow the guide it
prints.
Do not install, import, or launch Playwright, Puppeteer,
headless Chromium, or Chrome.
Use one TaskSpace named after the task id, open only the
URLs the task needs, close the pages you opened and call
`finish()` at the end.
If ego-browser fails, report ok false with the error; do not
fall back to another browser.
```

And the launch command for Claude Code. `$FLAGS` holds `--permission-mode acceptEdits` plus this task's tool whitelist, and the whole string goes into the first line of the log so the permissions can be checked afterwards:

```bash
{
  echo "# cmd: claude -p <prompt.md> $FLAGS"
  claude -p "$(cat <prompt.md>)" $FLAGS
} > .claude-results/<task-id>.log 2>&1
```

## The short version

If I were wiring this up again:

- two shared skills, `dsh result json` and `claude result json`
- seven fields, `summary` under 200 characters
- `~/.local/bin/dsh-wait <result-file> [interval=15] [timeout]` plus AwaitShell, with a handoff at 40 minutes
- read `ok` and `summary` first, slice `detail` on demand
- Notion: in progress, then done; failures stay in progress
- Claude: acceptEdits plus a minimal allowedTools list, command line in the log header
- ego lite only
- never: end a turn and hope the Mac wakes you, poll a progress file after nohup, paste `detail` into chat, or idle-check every 15 minutes

## Why I moved the work back

The chain worked. It also cost the orchestrator's own token budget and stability: a relay every 40 minutes, backoff when the Mac is unreachable, reading results and deciding what to do next, all of it lands on the bot. The arithmetic was simple. Writing code and running tests myself in dsh keeps the context in front of me, where I can see it and change it. Directing it from the cloud, one hop away from the machine, meant several rounds just to agree on what state we were in.

So the split changed. Code, dsh runs, Claude Code runs: mine. Grok bots: research and SEO, which they are better at, since the information is scattered across the web and they can read it in parallel without me watching.

I kept the result files, and the two skill docs are still there, `dsh result json` and `claude result json`, for anyone who needs to wire this up. The expensive part was pulling large results back into the orchestrator's context over and over, and a result file with a 200-character summary keeps a gate between the two sides. The gate is still there. Fewer jobs go through it now.

