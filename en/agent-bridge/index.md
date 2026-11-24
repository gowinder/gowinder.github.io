# Away From the Desk: I Gave My Mac Agents a Phone Entrance, and Why "Yes, Merge It" on a Phone Doesn't Count


The last post was about the result file, how the coordinator and the executor hand work over. That setup works, but I skipped past a precondition at the time: the executor runs on my machine, so the human has to be at my machine too. Later I moved the heavy work back onto my own laptop, and that precondition turned into a daily problem — the work lives on the Mac, and I am often not in front of the Mac.

I use Claude Code and pi, both wired to my own magpie gateway. The everyday model is DeepSeek v4.1 Flash; it handles coding and chores. For design review I bring in something stronger (gpt 6.1 sol — I used Opus 5.5 before, and switched after the account I'd had for years got banned). Most of the work is coding, but I also use them to collect material online, and lately to look for jobs and send resumes. How many sessions I open depends on the day: one or two when things are quiet, more than ten when it's busy, each one holding a different thread. When an agent finishes a round, or asks a question only I can answer, it just sits there waiting. At my desk that's fine. The problem is I'm not always at my desk — I take the dog out, and that session waits half an hour.

I tried remote desktop. Opening a whole desktop session to answer one line is too heavy. Tunnels and the like mean leaving an inbound hole in my home network, and I don't want that hole. There is an official Remote Control, and I'll explain below why it doesn't work for me. So I looked around, found nothing, and built one myself: Agent Bridge. Repo created on November 10, 0.11.0 shipped on the 14th. The scope is deliberately narrow: on a phone browser I can see the agent sessions running on my Mac and reply to them. That's it.

## Five rounds of validation before writing any code

I didn't start with code this time. I wrote a plan on November 9 and spent a few days on pre-validation, mostly out of fear of a few assumptions. Some things sound self-evident and turn out to be nothing like that in practice. The biggest trap in this project was never "can I write it", it was "did I get the model of the world wrong".

Validation said the relay path works: a Cloudflare Durable Object can host WebSocket rooms, Noise has standard implementations, and end-to-end encryption doesn't need to be invented. For Claude Desktop's Code sessions, you can push messages in while the desktop app is running: every session process has a socket under `/tmp/cc-socks/`, and messages sent in that way landed at the end of the same record, visible in the UI, without forking the record. ChatGPT's Codex threads can go through its own IPC, but only into threads that are already open in the desktop app; threads that aren't open can't receive anything.

Some options got crossed out on the spot. Claude's regular chat has no local record on the machine at all — the version I have installed doesn't even ship a chat UI — so that line was dead. The honest conclusion was "partially works", and I scoped the project to exactly that: whatever path verified, build it; whatever didn't, skip it.

## The architecture is nothing fancy

A daemon on the Mac, plus a small menu bar app. It only dials out, connecting to the Cloudflare relay and staying connected. Nothing inbound is opened on my home network — no port forwarding, no firewall changes. I care about that one: opening a hole in the router just to answer a message remotely is not a trade I'll make.

The relay can't see content. Browser and Mac speak Noise end-to-end encrypted; the relay forwards ciphertext and at most knows the room id, when there's traffic, and how big the packets are. Pairing doesn't use accounts either. The Mac generates a one-time token, only its hash goes to the relay, and the token itself sits after the `#` in the link — the part a browser never sends to the server. During pairing, the phone and the Mac each display a 6-digit code; if they match, pairing proceeds. The token is single-use and expires in a few minutes.

The painful part is adapters. Every agent stores sessions and receives messages differently, so each one has to be wired up individually: Claude Desktop's Code, the Claude Code CLI, ChatGPT's Codex, the Codex CLI, Antigravity, DeepSeek Harness, and later pi. And after wiring each one up, capabilities have to be labeled honestly — if reading is incomplete, mark it partial. None of these agents has ever offered a stable interface; claiming full support would be lying to you and to myself.

## Every feature came out of use

The first version only mirrored: move the desktop session onto the phone, read and reply. That was enough for a few days, until I got stuck on approval — I say "yes, merge it" on the phone and it doesn't count. That deserves its own section, further down.

The breakthrough was hosted sessions. Injecting text into someone else's session is semantically relaying, so I stopped injecting. Instead Bridge starts a Claude Code process of its own and owns the input side. Sessions created from the web are all like this: tool calls that need approval and questions the agent asks travel back over the protocol to the browser, and tapping approve or reject on the phone actually takes effect. The cost is that these sessions are invisible in the desktop app — two separate worlds. After 0.1.7 shipped, my habits drifted over almost entirely.

Everything after that was chased by small annoyances. When Bridge restarted, it took the processes it had started down with it, killing whatever round was in flight. So I added a resident daemon that owns those child processes: Bridge restarting now only drops the connection, the round keeps running, and the missing output is filled in when you come back. An agent finishing at 3 a.m. with nobody watching led to push notifications. Later pi became hostable too — pi is what I use most, so the web UI can switch its model and set a separate model and thinking level for subagents.

## The three small features I use most

Todo was added in 0.11. Before it existed I didn't think anything was missing; after it landed I can't work without it. Mid-conversation I suddenly remember something — maybe a feature, maybe a bug — and the current conversation is busy, and I don't want to start a new conversation and pull a new branch just for that. So I drop it into todo and tag it as feature or bug. When the conversation finishes and things free up, I tap that item and it lands in the input box, ready to send. Items live in Bridge's own database, so clearing browser cache doesn't lose them; a different machine means a different database, which is worth remembering.

The sidebar is barely noticeable on a phone but is the workhorse in a desktop browser. The left column can stay open, with the structure tree and history; at a glance I can see which session is running, which one is waiting on me, and which one has unread replies. Same for the info bar inside a session: how much context is used, which background tasks are attached, all laid out. Compared with scrolling around a TUI in a terminal, I'd rather keep the web page open — the information is spread out, nothing to memorize.

Then there's quick switching: one button in the top bar that jumps back and forth between the two most recent conversations. It sounds tiny and it's impossible to stop using, because I discovered that I only ever follow two conversations deeply at a time. Ten may be open, but my attention rotates between two; any more and I can't keep up. In the end, however well the agents run, the queue terminates at me. The bottleneck in the AI era is the human brain. That's just how it is.

These features now sit on the path of my ordinary day. On the commute, walking the dog, on the couch with the TV on, lying in bed at night, even sitting on the toilet — whenever I remember that some session needs a reply, I pull out the phone and read it, answer it, queue the next thing. The reason that works is that hosted mode is basically full-featured: not a read-only mirror of the desktop app, but the session itself running completely on the phone.

## Not being able to approve — I thought about this for a long time

There's only one path for injecting text into a desktop session, the peer socket, and injected text arrives wrapped:

```
<cross-session-message from="uds:/tmp/cc-socks/<pid>.sock" from-name="Agent Bridge · <device name>">
what you said
</cross-session-message>
```

The receiving agent can tell the content from the person. For editing code or looking things up, it just does it. But for merging, releasing, deleting — it replies that this came from another session, so it doesn't count as me saying it. At first I saw that as an obstacle. Later I decided it's correct. If Bridge stripped that wrapper and disguised the phone's words as something I typed into the desktop input box, then any forwarded piece of text could impersonate me and approve a release. That hole can't be opened; once it is, the whole security model of the tool collapses.

So there are three crude workarounds. Either I come back to the computer and click the last step myself; or I tell the desktop session up front that once all checks pass it can merge PRs on its own, and a later "merge it" from the phone falls under that prior authorization; or I have it print the command and I copy it into a terminal myself. None of them is elegant, but the semantics are clean. If I really need to approve from the phone, I open a hosted session — that problem doesn't exist there.

I looked carefully at the official Remote Control. Messages sent from the official app do count as your own turn, without that wrapper. But three things didn't line up. It requires a claude.ai subscription login and accepts neither API keys nor third-party gateways, while my machine goes through magpie with no login, so it just errors out. It exposes no interface for local programs, so Bridge has no way to hand it the phone's input. And its records live on Anthropic's side, while the premise of this whole thing is that content never leaves my machine and the ciphertext channel. It's a good feature that solves a different problem than mine.

I also experimented with `claude --resume -p`, appending to a session by id. It really executes — it will even make the commit for you — but it executes in a newly spawned process, not the one open on the desktop. The desktop process doesn't read the record off disk, so the new words never enter its head, and two processes writing the same record file fork it; the appended lines become orphans nobody claims. The receipt says success while the actual state disagrees. That's worse than not being able to do it. I stopped and dropped it.

Sessions running directly in a terminal have one more small problem: when a multiple-choice prompt pops up, the web page can't answer it. I tried several delivery methods — the channel ignores it, or the place I write to isn't the input queue, or permissions aren't enough. There's only a way when the session runs inside a herdr panel: that panel can send real keystrokes, so it first reads which option the cursor is on, then presses arrows and Enter. I only risk single-select; I don't touch multi-select. Without herdr, I go back to the terminal and answer honestly.

The rest of the limits, all at once. The web UI can only create hosted sessions; every other adapter attaches to a session that already exists. Reading is generally incomplete — missing tool output, missing attachments, missing early conversation, any of these. Antigravity's desktop build is read-only. Some methods in the protocol are defined but not implemented. The web gets new messages by polling: the chat page refreshes every five seconds, so it isn't real-time. The UI is Chinese only. Record files sit on disk unencrypted, protected only by the system's directory permissions. One room binds to exactly one Mac. That's about it; the changelog has the rest.

## Where it stands now

0.11.0, MIT licensed, on GitHub at [gowinder/agent-bridge](https://github.com/gowinder/agent-bridge), DMG available for download, web client at [agent-bridge.craftstead.app](https://agent-bridge.craftstead.app). Signing and notarization require an Apple certificate I haven't configured, so the unsigned build needs a manual allow in System Settings the first time you open it. It's in the README. Nothing to gloss over.

My own usage has changed three times. At first I used Claude Desktop's sessions; mirroring looked convenient, but the moment I needed to commit code or open a PR I got stuck — no approval, so the last step always meant going back to the computer. Then I switched to Bridge-hosted Claude Code, which runs the whole flow. Now I mostly use hosted pi: it's more complete and more open, and models, subagents and thinking levels can all be set directly from the web page.

One more thing that sounds a bit circular: most of Agent Bridge's later development was Agent Bridge developing itself through hosted pi. When inspiration hits, I drop it into its own todo first, and when one item is done I line up the next one by hand — I can push it forward from anywhere. Once the auto-update toggle in the app's settings does everything by itself — detects a new build and installs it without me clicking "Check for updates" and "Update now" — I can basically use it without the Mac as the host machine. A tool that can leave its own development machine and keep developing itself: that line is closed.

Nothing urgent next by way of new agents. Push notifications for subscriptions are owed — build them and polling can retire, and that's the biggest experience gap. A few corners of hosted pi still need polishing; I'll finish those first.

One question to leave open, and I genuinely want to hear how others think about it: a tool that reaches your development machine straight from your phone — how far does it have to go before you'd dare leave it on all the time? My answer is everything above: outbound only, the relay sees no plaintext, approval rights are never lent out. If any one of those goes slack, I'd rather not use it.

