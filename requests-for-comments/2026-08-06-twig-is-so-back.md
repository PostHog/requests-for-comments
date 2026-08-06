# Request for comments: twig.com – a real company that runs on self-driving & Sarah Sanders

## Problem statement

Self-driving is confusing to the people we built it for, and the confusion is mostly because we explain it instead of show it.

We're working on improving this by providing real use case content, but I think we need a full working, real "example app" to test it on.

**Who we're building for:**

- **Prospective self-driving users:** – Our ICP: AI-pilled engineers that review more code than they write by hand at companies of any size, anywhere. They need something real and tangible to look at before they'll point an autonomous loop at their repo.
- **Internal teams:** – Positioning of self-driving is doomed to spread across many different teams. Docs, marketing, engineering, editorial, sales etc. We need one canonical artifact to link to instead of re-explaining all the time.
  - **Sales:** – Sales works with demo sites, but it would be really powerful to demo on production data without using PostHog's.

## The proposal

Today, every explanation we ship feels like we're trying to proove something, none of it is real evidence. The strongest artifact here isn't a better explanation, it's a real company, visibly run this way, with public receipts.

**Let's launch a real company that runs on self-driving and ship it on twig.com.** 

**Ideas:**

I have TWO pitches:

- **PITCH 1: daily comp sci puzzles.** A daily-puzzle company with new logic puzzles everyday, NYT format, but instead of focusing on word games, all of the games are rooted in computer science principles. Games are 2-5 minutes each, support streaks, and have spoiler-free share cards to share with friends. We can launch with one flagship game, and build more from there.

- **PITCH 2: sell twigs to birds.** This is the kookier one, but hear me out. People who own bird feeders with cameras (or we can get some people internally to set some up) opt in to share their live feeds. We use vision models to figure out which nesting materials birds prefer, then open a store selling bundles of the winners. It's silly, but it's a literal self-driving use case: product decisions are driven end to end by AI watching real users. Those users just happen to be birds lol.

- **PITCH 3: we users choose.** Great suggestion from @mjwarren3. Customer discovery is a part of self-driving I don't even think we explain much on the website or docs site. We could let users choose what we build, then use that data to kick off self-driving.

I personally love the bird idea, for the absurdity of it. But I am not married to either... if anyone else has any other ideas, I am all ears.

The only requirement that will really make an idea a good self-driving company is that it generates real recurring signals itself (funnel, tickets, product decisions) and it's run by ONE human so the loop is legible. I will be that human.

It can't just be another boring demo app.

## Sarah, aren't we already the company?

Yes, we dogfood self-driving. But dogfooding and a separate "micro" company running on its own are different! Dogfooding can tell you it works, but with the amount we ship, it's too hard to track.

Twig is on a micro level:
- It is legible. Scout PRs won't ship in the thousands per day. We need outsiders to be able to isolate the loop. With trig, every merged PRs is likely to be FAR more legibile and trackable.
- No one can make the argument that an engineer could have caught what self-driving did. Twig doesn't have engineers :)
- We don't have to worry about vendor discount bias. Saying "we use our own product" is claimed by every vendor, but people don't get to see HOW we use it.

## Success critiera

We should let it run, for say, 60 days, then start measuring:

- **15+ scout-authored PRs merged on real signals**. All of these are publicly readable with evidence trails.
- **Docs can adopt twig as the canoncial example app.** We can showcase real artifacts on every relevant docs page and stop talking in hypotheticals.
- **We have a few content moments.** We can have a big launch post and a 60-day update series.

We are not looking for revenue or daily active users here. The company just has to be real, not big.

## Budget

According to Slack, I have $10-25K. Once we make a decision on company idea, I'll plan budget spend accordingly.

## Open questions

- Decide on a company concept
- Legal/billing shape for payments if we do public payments (bird houses for sale, bird nesting supplies for sale, or puzzle premium accounts)
- Budget plan (allocate spend to different areas)
