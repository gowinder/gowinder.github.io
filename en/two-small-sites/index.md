# Two small sites went live: an Old English dictionary and a batch renamer


# Two small sites went live: an Old English dictionary and a batch renamer

I wanted to test one question: can one person, with no budget and no ads, get anywhere by taking a very narrow need all the way to the end?

Rather than argue about it, I built two things. They sit at opposite ends of the spectrum — one is pure content, one is a pure tool — which makes them a clean control group.

## eldertongue.com — an Old English dictionary

The first is [eldertongue.com](https://eldertongue.com), a dictionary for Old English.

The reason is boring and practical. Looking up an Old English word today means landing on a university site that looks like 2003, or digging through PDFs and forum threads. The people who need it are not rare: students reading Beowulf, writers building a fantasy world, anyone curious about where a word came from.

So I reorganised the entries as form → part of speech → gloss → example → cognates, and generated close to twenty thousand pages. Technically it is very light: static output on Cloudflare's edge, fast everywhere, and cheap enough that hosting is not a line item.

This one is my experiment in whether content can grow on its own. Half a conclusion: making the content is not the hard part. Getting it found is.

## massrename.com — a batch renamer

The second is [massrename.com](https://massrename.com), a tool for renaming many files at once.

It came out of my own annoyance. When I clean up scans, TV episodes or photos, the built-in rename tools stop at "add a prefix" or "change the extension". Anything real — reorder by capture date, regex replace, zero-padded counters — means opening a terminal. People who can write the script do not need a tool. People who cannot write it, cannot write it. The middle is empty.

So it is a web app: stack rules, preview every result live, and only run it once the preview is right. One-time purchase, no subscription.

This one is my experiment in whether a tool can sell. It launched days ago, so there is nothing meaningful to report yet.

## Two things I am now sure of

**Building has become almost free.** Static site, edge deploy, one domain: both sites together cost less than dinner. The expensive part is not making the thing. It is deciding, before you make it, whether it deserves to exist.

**Code is not the bottleneck. Distribution is.** Both sites forced me to re-rank the tidy technical answers I had collected over the years. Performance, accessibility, structured data — you should still do them, but their ceiling is "does not get in the way". Being found, and being trusted, is a separate problem with a separate set of rules.

## What is next

I will write down what happens with these two — including the calls I got wrong. Not a success story, just: what I thought at the time, and where that turned out to be wrong.

If you happen to need either of these, it is there. Questions are welcome in the comments, or find me on X ([@craftstead](https://x.com/craftstead)).

---

*Both sites are built by me alone. Stack: Astro, Cloudflare Workers, Cloudflare Pages.*

