# Every Drill Passed. The First Real Order Broke Everything: PopHonest's Payment Chain, As It Happened


## First, what this is

[PopHonest](https://pophonest.com) is a WordPress popup plugin I made. Popups have a bad name — fake countdowns, fake stock levels, "only 2 left!" — so the plugin's whole pitch is the word in its name: Honest. Popups when they should pop, no dark patterns. Core is $149 a year, Pro is $249 a year, payments run through Waffo, and activation is a licence key.

The product itself isn't much to talk about. This post is about its payment chain, because the thing that taught me a lesson wasn't building the plugin. It was taking money.

## In the drills, everything was green

My flow looks like this: Waffo pushes order events to my Worker over webhooks. The Worker verifies the signature, checks the amount, issues a licence key, emails it to the buyer, and the buyer activates Pro in their WordPress admin.

Before launch, I drilled this whole flow for a full round. The payloads were signed by me, in exactly the format Waffo's documentation describes — indistinguishable from real callbacks. Everything passed. A wrong-amount order ($19) was correctly rejected, with the reason logged. The $149 and $249 orders issued keys normally. A renewal issued a new key. A resend by order number returned the same key — idempotency fine. A forged signature got a 401. In Docker, a key signed during the drills really did activate Pro.

My feeling at the time: solid. Ready to launch.

Looking back, what "solid" actually meant was: I had read the documentation correctly. That was all.

## The first real order, and everything the docs didn't say

On November 23 I ran the first real TEST order. Test card, real Waffo checkout page, real callbacks, no step skipped. The order went through and the ledger balanced in the end — 20 out of 20 checks. But what surfaced in the middle was in none of the drills:

**First, subscription purchases don't send order.completed at all.** My ledger had 8 rows of order.completed, every one from my own self-signed drills. The real order delivered exactly two events: subscription.payment_succeeded and subscription.activated. The event flow I had assumed during the drills was not the event flow of the real world.

**Second, the two events split the work differently than I thought.** payment_succeeded arrived and the Worker only recorded it — no key issued. Money arriving is not the same as a subscription being active; that distinction is Waffo's, and it's fixed. The actual trigger for issuing the key is activated, which comes later. Lucky the issuing logic was hanging on the right hook — otherwise it's either taking money and handing out no key, or handing out two.

**Third, there is no separate subscription ID.** I had always assumed a subscription gets its own number, and renewals and cancellations anchor to it. They don't. The orderId is the subscription anchor. Renewals recognize that.

**Fourth, you don't compute the expiry date yourself.** The activated payload carries currentPeriodEnd directly — for this order, November 23, 2027. The ledger records that, not my prepared default of "issue date plus 400 days." That default nearly got used.

One small tail: the licence email really did reach the buyer's inbox about a second after ordering, but it was sent through another domain of mine that already has email sending set up. PopHonest's own production sending domain still isn't enabled as of today, so that result doesn't carry over to production. I'm writing it here as an IOU.

## A rule that came out of it

PopHonest was also the first product under my "one project, one store" rule. The earlier products shared one Waffo store; this time I opened a separate store. Then I found out Waffo reviews per store: the merchant account was long approved, and the new store still went through its own review, with production checkout sitting at STORE_NOT_APPROVED until it cleared, unable to move.

The lesson is concrete: opening the store belongs at the front of the launch checklist, on the same level as the domain and Search Console — one to three business days early. Open it on launch day and the store is in review while you stand there waiting, and the whole release rhythm falls apart. This is a standing rule for me now.

## Last thing

My current view of payment chains: passing your drills only proves you understood the documentation. A real order going through proves it can actually take money. What lives between those two things is never fully written in the docs — not because the docs are bad, but because some behavior only shows itself when real traffic runs over it.

So my launch checklist has one more hard rule: before any paid product launches, one real TEST order runs end to end — buy, pay, use, all three stages verified. No matter how green the drills are, they don't count.

What did your last product's first real order dig up that the docs never mentioned? I bet something.

