# I Built a Cost Simulator for My Agent Runs, Then Killed the Paid Version


## You don't feel it while the money burns

I run agents every day now. Read this code, change that file, run the tests, round after round. It flows so well that money never crosses my mind.

The problem is that an agent's bill doesn't work like "one question, one price." Every extra round makes the context longer, and everything before it gets fed back in. Then there's thinking, which is billed at the output rate — the deeper it thinks, the more it costs. At the end of the month I get one total. Which run burned the most, and why? No idea.

There are plenty of Gemini price calculators out there. I tried them. Most are single-request calculators: tokens in, tokens out, multiply, done. That's fine for a chatbot. It doesn't work for agents — an agent's cost grows. One round and ten rounds are completely different animals, and a single-request calculator can't see that.

So I built one that simulates per turn.

## What it is

[Gemini Cost Simulator](https://geminicost.craftstead.app). Free, runs in the browser. It does one thing: it takes an agent run apart turn by turn — how the context grows, what thinking costs at its rate, roughly where the total lands — so you can do the math before you run, not after.

Two extras came along. One is a side-by-side of the price tiers before and after the change coming on December 1, which I'll get to. The other is an MCP server, so an agent can call the simulator itself and get an estimate before it starts, instead of a human clicking around a web page. Self-test: 16 out of 16. It's listed on the official MCP Registry and on Glama. The pages are in English, Spanish, and Brazilian Portuguese — no deep reason, Gemini's users are scattered everywhere anyway.

It's a small thing. The interesting part is that it almost wasn't free.

## How the paid version died

When I scoped this, I scoped it as a paid product. The plan was ordinary: basic simulation free, saved scenarios and CSV export behind a paywall, payments through the stack I always use. Half the code was written. The products were already created on the payment side.

Then I went and actually looked at the competitors.

Every direct competitor is free. Not one charges.

And our differentiation — per-turn simulation, thinking billed properly — is real, but thin. How thin? A developer with a spreadsheet and the pricing page can get within spitting distance in twenty minutes. You don't build a paywall on that. The wall isn't finished before it's blocked off most of your own users.

There's a more practical account too. Charging isn't a button. Payment integration, key management, order reconciliation, refund arguments — the money chain is the most expensive thing a one-person shop maintains. Keeping an entire chain alive for a difference that can't hold a price doesn't add up.

So the payment PR got closed, unmerged, and the site went fully free. Slightly embarrassing, killing your own business plan. But the earlier you kill it, the cheaper it is. If that PR had shipped and then nobody paid, the sunk cost wouldn't have been half a codebase.

## After launch, the ugly numbers

Once the free version was out, I started doing what every indie developer does: submitting to directories.

One at a time. Findly took it, with a dofollow link — a decent start. Curlie is submitted and waiting. TinyLaunch scheduled it for December 21. Another one wants email verification; sent, waiting. There's no one-click distribution shortcut. It's just dumb grind.

Indexing, since we're being honest: the site has 9 pages. Google has 7. The Bing family is still at 0. I ran an AI citation baseline with a fixed question set — 9 questions, cited 0 times. The competitors are all sitting in those answers.

I'm not going to dress this up as "every beginning is hard, it'll surely get better." The truth is: the tool is done, and distribution has barely started, and distribution is much harder and much slower than building. When you build, a line of code is a line of progress. Distribution isn't like that. You submit to ten directories and maybe hear back from one, two weeks later. Mostly you hear nothing.

## Why December 1

This post is timed because Gemini 2.5 Flash changes its pricing on December 1, and the request routing changes too. Put plainly: my tool does the math on Gemini bills, and the day Gemini raises prices is the day the most people need the math done.

On December 1 I'll post on X and on Reddit, both written in advance. Reddit has rules, so I'm only posting in one subreddit that explicitly allows self-promotion; for the other, I'm asking the mods first and accepting whatever they decide. This post is the long version of the same wave — the posts get seen, the blog tells the whole story.

I have no embarrassment about riding a moment. When a tool product has no natural traffic, events are traffic. The only question is whether your thing is genuinely related to the event. The price rise is real, and the before/after tier comparison in my simulator is a real feature. That's enough.

## The stop-loss line, in writing

Ugly words up front. My line for this site is mid-December. If indexing, citations, and traffic are still exactly where they are now, with no sign of life, I'll shut it down instead of letting it hang.

Free products die of one disease: you can't bear to close them. Server costs are low, the domain is a few dollars a year, leaving it up feels free. But every site left hanging eats maintenance and attention — fix one here, patch one there — until everything is half-dead. Closing something nobody uses isn't failure. It's freeing up your hands.

I'd rather it lived, of course. I still use it myself, every day — before I dispatch a big job, I run the estimate first. That habit has stuck. Even if I'm the only user in the end, the tool wasn't wasted. It just can't have any more of my time.

What about you — do you do the math before you run an agent, or are you like I used to be, waiting for the end-of-month bill?

