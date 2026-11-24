# Muse Is in the Cloud, My Code Is on My Mac


{{< image src="/images/muse-agent-bridge-workflow-en.svg" alt="The Muse × Agent Bridge workflow: I set the spec, Muse writes specs, dispatches and verifies from the cloud, the work travels over MCP and an encrypted relay into a hosted session on my Mac, and the result comes back to me" width="640" caption="I set the spec, Muse dispatches and verifies, the work happens on the Mac, and the result comes back to me — MCP and an always-on listener close the loop." >}}

Quick note on where this post sits, so it doesn't blur into the previous two. In the November 5 post, I was the one driving the agents on my machine. The November 15 post was the Agent Bridge release post, about the tool itself. This one is about the MCP piece that came later: the driver is Muse now, and I'm mostly left with deciding things and looking at results.

Muse doesn't live on my computer. It's Meta's personal agent, it has a cloud machine of its own, it can run programs and stay resident — that detail matters later. Everything of mine is on the Mac. The repos, the files, and a bunch of things that can only be done inside the environment that's already signed in on this machine. It can't reach any of that.

And when it can't reach, what do you do? The dumb way. Copy and paste.

It went like this: I'd talk a task through with Muse, it would write out the instructions, I'd copy them, switch to a Pi session on my Mac, paste, wait. Then copy the result, switch back, paste it in for review. Fine for one task. Two, three? Which session is running which project, which one is waiting on me, whose output is whose — all of it lived in my head. My head is not big enough. Sooner or later you paste into the wrong session, and once that happens everything downstream is quietly wrong, and you don't find out until much later, when you no longer know where it started going wrong.

So MCP, for me, is not some grand concept. It just took over the courier job I was doing by hand. Muse sends the task to my Mac itself, fetches the result itself, and keeps the mapping straight. I don't.

## What's actually in between

On Muse's side there is an MCP server — stdio — plus a small command-line client. Together they are one paired device, named `Muse (MCP)`, connected into Bridge over the existing Noise-encrypted relay. The Mac still only dials out. Nothing new is exposed.

Stdio, in plain words, means two programs have to sit on the same machine and talk through a pipe. Somebody local has to start that server and keep the pairing identity on disk. Muse can do that because it has that VM of its own. Which also fixes the shape of the thing: this is not a "paste a URL into some web page and you're connected" kind of deal. That form doesn't exist. Slightly deflating to write, but it's the truth, and dressing it up would be worse.

Some config trivia, since it cost me thought once: when a session is created, all parameters are passed explicitly. Hosted Pi by default, main model `magpie/group/dsgroup`, thinking level high, subagents also on dsgroup but thinking low. Every task message also gets a default suffix, "use subagents to do it", so the local Pi fans the work out — it's right there in the frontend, not injected behind anyone's back. Why pass everything explicitly? Because Bridge remembers the parameters of the last manually created session, and if you don't, whatever I last clicked by hand leaks into the automated sessions, and when something goes wrong you can't tell whose settings did it. Details like this don't feel important until they've bitten you once.

## One request I made about sessions

Don't keep opening new conversations.

Simple reason: by the third round on a project, if the local agent has forgotten rounds one and two, I have to feed it everything again, and re-feeding is miserable. So the rule is get-or-create: key a session by engine, working directory and purpose, reuse the existing one if there is one, create only when there isn't. New sessions happen when the directory changes, when I explicitly ask, when the old one is archived or gone, or when a one-off task has polluted it beyond use. When it does create one, it says why. No silent pile-ups.

Titles look like `MCP · P011 · Email Sending Setup` — a rule from November 22, project number right after "MCP". Sessions created before the rule can't be renamed; the bridge has no rename. They just run out their lives under the old names. Fine.

There's also a second paired device, `Muse (MCP) Listener`. Why separate — that comes with the notifications part below.

## How one job actually runs — take the blog migration

Process descriptions are boring. Take a real one.

On November 20, the build for switching this blog to the Stack theme went out to a hosted Pi on my Mac, session `MCP · blog-stack-launch`. I remember it because the gates were absurd: a baseline of 582 URLs from the old site, and the new build was not allowed to lose a single one. Titles byte-for-byte identical. Plus a hand-written robots.txt, the live Markdown output format, the og images — each with its own trap.

It built locally first, without touching the publishing repo. Then it checked its own work and found three regressions. Fixed them, checked again, found a second round. These were all the nasty kind: the build succeeds, the pages open, and something somewhere is silently wrong. Green lights everywhere, broken product. Those are the worst bugs there are.

In the end every gate passed. The publishing commit is still sitting on my Mac, unpushed. The date we set was November 25: once the blog's search data has had its full check, I look at it, and only then does it go out. So as of writing, this site still runs the old theme. If you're reading this on the new one, I must have nodded. Approval and verification are two different things, and that line is drawn very deliberately in this setup — mostly because I don't want it getting enthusiastic one day and shipping something on my behalf.

The mechanics, stripped down, are the loop in the diagram at the top. Muse writes the spec — specs are much longer than a normal prompt, red lines, boundaries, acceptance criteria, all in there — sends it into the session, and registers a watch with the listener: which session, which conversation it belongs to, what it's for. Then the local agent works. When it finishes, or gets stuck, the listener wakes Muse up. It reads the result, verifies it itself, and comes back to the original conversation to report. If a session has no next round coming, it archives it in Bridge and mentions it in one line.

What if the local Pi asks a question mid-task — pick option A or B, that kind of thing? There's a rule for that, picked early on, called Mode B: the question gets forwarded to me verbatim, not a word rewritten, I answer item by item, and my answers go back as-is. No summarizing — summaries lose things. And definitely no pre-filling an answer for me. That's not help, that's deciding in my place.

The division of labor settled itself, nobody designed it in a meeting. Short jobs that its own machine can handle, the cloud Pi just does. Long jobs, jobs that need the local repos, go over MCP. And some jobs the cloud simply cannot do: 011's Email Sending setup was one — the API said 403, the cloud browser got stopped by a verification wall, and in the end it was a session on this Mac that did it, with a spec full of red lines: don't touch the existing mail routing, stop if money or a card is involved, stop if not signed in, never ask me for a password. P013's ten pages, the radar lines — same road, nothing special. It's just the daily road now.

One more split, between me and Muse: it doesn't write code. It writes specs, dispatches, reviews, verifies, delivers. The code is written by Pi and Claude Code. If it wrote the code itself, who would review it?

## The dumbest incident so far

Around the 010 days, a hosted session vanished from the MCP session list mid-task.

First thought: it died. Looked into it — seemed gone. Looked again — still gone. It was not gone. After a Bridge update, the app had touched some protected folders, macOS had thrown up a permission dialog, and the Pi process was stuck behind that dialog. Two hours. Alive, unable to move, and from where I sat, through the MCP layer, there was no sign of any of it — not in the list, and certainly no dialog.

One click, if you're sitting at the machine. Maddening, if you're not.

The rules that came out of that: if a long task stops making progress, suspect a local system dialog first, don't start inventing gateway theories. Absence from the list is not a death certificate — somebody has to look at the process list on the Mac itself. Half an hour without progress, treat it as a zombie: kill it locally, re-dispatch, tell the sessions apart by creation time, don't wait it out. Archiving follows the same logic: only archive sessions confirmed finished, with the output already harvested and verified. Running ones, stuck ones, ones that merely can't be found in a list — hands off. Nearly archiving a live process is a lesson you only need once.

## Notifications are polling, slightly embarrassingly

Bridge has no subscribe API. We tried. `session.subscribe` comes back unsupported.

So the listener polls, every 15 seconds, over the sessions it's watching. How does it decide something finished? Two conditions together: the session went from busy to idle, and its message count grew compared to when the task was dispatched. Idle alone proves nothing — maybe it never started. Growth alone proves nothing either. Questions are a separate signal: a needsDecision flag rises, and the question gets forwarded to me right away. That's also why the Listener is its own device with its own connection: the dispatch side connects, speaks and disconnects per call, and a listener that disconnects is just deaf. It has to sit there permanently.

Fifteen seconds means notifications are routinely ten-odd seconds late. I don't care. Nothing I dispatch comes back in under a few minutes anyway.

## Where the money goes — three small accounts

Honestly, this wasn't built to save tokens. The original design note says it plainly: the point was that some work can only happen on this machine. But after running it for a while, the accounts are visible, so here they are.

What actually burns tokens is the coding loop itself: read files, edit, run tests, hit an error, edit again. Dozens of rounds on a big task, context growing the whole time, every round billed. On this setup, all of those rounds happen on the local Pi, burning models on my own magpie gateway. Muse's side is a handful of rounds — the spec, reading the result, the review. Its budget gets spent on judgment, not on typing code. Same task, wildly different bill, depending on whether it does the work itself or stands next to it and watches.

Second account: the hardware. The work runs on this Mac, which has 64 GB of RAM, the repos already on its disk, and it will do big builds, several sessions in parallel, a Docker stack, without breaking a sweat. Muse's cloud machine is a small VM with an order of magnitude less memory — even if it could reach the files, it would lose on hardware first. The Mac is sitting here anyway. Might as well work.

Third one I've basically already said: some things the cloud cannot do at all. Files here, sign-in state here, apps here. Add the three up and this stops being a convenience. Some of the work simply doesn't move without it.

## 005, where all the tests passed and I killed the result anyway

Last story, and it makes me look bad too — I nearly believed the numbers.

Pixel Plunder, the pirate game, second art-correction round. It came back with beautiful stats: 173 tests, all passing. Typecheck clean. Every asset registered, none missing. By any "green means go" standard, that round was done.

Then I opened the images. The ships were wrong, wrong at the root: the sprites themselves had been drawn almost side-on, and no amount of programmatic rotation will fake the diagonal, deck-visible perspective of the locked key visual. The port was wrong too — the prototype is a cove, buildings in one row along the back shore, beach and people in the middle, sea and docks in front; the shipped version had shoved the sea off to one side and clustered the buildings on empty sand.

The numbers hadn't lied. Everything they checked was true. What they couldn't check, they never claimed to check. The last gate was my eyes, and the rebuild direction came out of that rejection: redraw the fleet in the locked pose, rebuild the port as a cove.

So this is where I've ended up: at the front I set the specs, at the back I look at pictures and results and nod. The carrying, the watching, the remembering which conversation belongs to which session — the part that used to wear me down — is gone. The bottleneck is still me; ten sessions reporting back at once will swamp me just like before. But that's a different problem.

That's where this post ends. The next one, whenever it happens, is probably about what to do when you can't review fast enough. I haven't figured that out yet.

