# Request for comments: twig.com – a real company that runs on self-driving & Sarah Sanders

## Problem statement

Self-driving is confusing to the people we built it for, and the confusion is mostly because we explain it instead of show it.

We're working on improving this by providing [use case content](https://posthog.com/pocket-guides), but I think we need a full working, real "example app" to test it on.

**Who we're building for:**
- **Prospective self-driving users:** Our ICP. AI-pilled engineers that review more code than they write by hand at companies of any size, anywhere. They need something real and tangible to look at before they'll point an autonomous loop at their repo.
- **Internal teams:** Positioning of self-driving is doomed to spread across many different teams. Docs, marketing, engineering, editorial, sales, etc. We need one canonical artifact to link to instead of re-explaining all the time.
  - **Sales**: I wanted to call out sales specifically because right now, they work with demo sites. It would be much more powerful to demo on production data without having to use PostHog's.

## The proposal

Today, every explanation we ship feels like we're trying to prove something, none of it is real evidence. The strongest artifact here isn't a better explanation, it's a real company, visibly run on self-driving, with public receipts.

**Let's launch a real company that run on self-driving and ship it on twig.com.**

**Ideas:**

I've considered a few pitches, and landed on one:

- **#1 PICK: daily comp sci puzzles.** A daily-puzzle company with new logic puzzles everyday, NYT format, but instead of focusing on word games, all of the games are rooted in computer science principles. Games are 2-5 minutes each, support streaks, and have share cards to share with friends. We can launch with one flagship game, and build more from there. Maybe we could even expand to physical puzzles ala [The Funday Press](https://www.thefundaypress.com/) one day?
- **PITCH 2 (not picked): sell twigs to birds.** This is the kookier one, but hear me out. People who own bird feeders with cameras (or we can get some people internally to set them up) opt in to share their live feds. We use vision models to figure out which nesting materials birds prefer, then open a store selling bundles of the winners. It's silly, but it's a literal self-driving use case: product decisions are driven end to end by AI watching real users. Those users just happen to be birds lol.
- **PITCH 3 (not picked): we let users choose.** Great suggestion from @mjwarren3. Customer discovery is part of self-driving and I don't think we explain it well. We could let users choose what we build, then use that data to kick off self-driving. _We can still build this into the puzzle idea!!_

**Why puzzles:** a daily update means twig.com ships _every single day on its own_. All the mechanics of it are machine-generatable (scouts generate new puzzles based off user data) AND machine-verifiable. This means the content pipeline can run at agent speed, and the maintenance cost of the whole thing is bare minimum.

_Everything in the rest of this RFC is now scoped with puzzles in mind._

**The requirement:** the company generates REAL recurring signals itself, generates revenue, and is monitored by ONE human (me) so the loop is legible. _It can't just be another boring demo app._

## Sarah, aren't we already the self-driving company?

Yes, we dogfood self-driving. But dogfooding and a separate "micro" company running on its own are different! Dogfooding can tell you it works, but with the amount we ship, it's hard to track.

Twig is on a micro level:

- It is legible. Scout PRs won't ship in the thousands per day (not that PostHog does this either, but you get what I eamn). We need people to be able to isolate the loop. With Twig, every merged PR is likely to be FAR more legibile and trackable.
- No one can make the argument that an engineer could have caught what self-driving did. Twig doesn't have engineers :)
- We don't have to worry about vendor discount bias. Saying "we use our own product" is claimed by every vendor, but people don't get to see _how_ we use it.

## Success criteria

We should let it run, for say, 60 days, then start measuring:
- **At least one daily scout-authored PR merged per day.** Once this thing is running at full speed, it will be machine-generating improvements, new puzzles, etc. All of these PRs should be publicly readable with evidence trails attached to them.
- **The entire company can adopt twig.com as the canonical example app.** We can showcase real artifacts across posthog.com, in sales conversations, at conferences and events, etc. and stop talking in hypotheticals.
- **We score a few viral content moments.** We can have a big launch post and updates along the way.

PostHog's production use of self-driving shows that we ship on average 3.8 scout-authored PRs per day, with 89 of 113 scouts running on a daily basis. This is the PostHog scale, so the success criteria for twig.com feels doable.

_The real bottleneck and risk here is twig's signal supply._

## How long this actually takes

Half-assing this isn't really an option. To ship this and ship it well, we need about two quarters. Yeah, it's a lot.

Scouts need project history and twig has none right now. A brand new product and site has zero signal, so it needs some time to build its signal supply. This means we need to:

1. **Launch in stages, not all at once.** We can't launch everything at once, or we'll never have any signal to work with. We can start with 15 hand-crafted puzzles to get us through 2 weeks and ship those on twig.com in anonymous mode to build traffic. Later, we can implement accounts, social sharing, integrate into posthog.com or app.posthog.com, etc. All of these stages are mini-launches with the opportunity to generate fresh signals. This keeps signal supply fresh and builds a moat around decay.
2. **Good news though, the baseline "waiting" time isn't dead time.** This is the time I spend building twig.com's initial launch version, hand-crafting puzzles, and building the stuff that doesn't need signals.

The rough shape of work might look something like this:

| Sprint | Duration | What ships |
| --- | --- | --- |
| Foundations | 2 weeks | twig.com infra, test suite, PostHog instrumentation, GitHub APP + AI data processing approval, legal/billing approval |
| Alpha launch | 2 weeks | Launch flagship game with 15 hand-crafted puzzles, support streaks in localStorage only, include survey for user feedback |
| Building | 2-3 weeks | Build the signal-powered puzzle generator, difficulty gates, accounts, integrate into posthog surfaces |
| Staged mini launches | 2-3 weeks | Accounts, payments (premium), more puzzle types, more custom scouts as we grow |
| Measure/let it cook | 8 weeks | Company is running: daily triage, PR review, marketing. Work on update series (content, blogs, socials,) |

## Product design

**Flagship game:** merge

Two engineers edited the same picture. Spotting their changes is the game:
- You tap cells in your merge to cycle between the versions that exist for that cell and rebuild the picture so it keeps EVERYTHING both of them did
- Somewhere, they both changed the cell differently, so you have to catch it and flag it
- You get 3 commits, and a failed commit only tells you how many cells are still wrong
- Win, and you get a card to share with your friends

Why merge?
- I created a few prototypes, and merge was the one that survived playtesting. It can progressively get harder through the week, starting with easy puzzles and gradually increasing difficulty.
- It's machine-generatable and machine-readable. A puzzle is just a grid, two edit sets, and a checker. Agents can product and validate the puzzle (I tested this) so our puzzle generation pipeline is reliable.
- Difficulty is scalable: grid size, number of edits, edit subletly, conflict placement. This means scouts can monitor difficulty srift and tune as needed.
- The theme suits our ICP :) it's a merge conflict! It's also visually appealing and engaging for others that aren't engineers.

Mockups from working prototype, using existing twig branding:

Game prototype:
![Game view](images/2026-08-06-twig-is-so-back/gameview.png)

Win card:
![Win view](images/2026-08-06-twig-is-so-back/cardview.png)

**Accessibility is a launch requirement.** A color-grid game locks out colorblind users, so I've designed for this. The prototype has shapes mode where every color carries a distinct glyph.

Shapes mode:
![Shapes view](images/2026-08-06-twig-is-so-back/shapemode.png)

I also need to design for signal supply:
- User-facing errors and replay-able frustration
- Surveys at launch
- Support path, routed to me
- Custom scouts: start with a few tuned for difficulty drift, streak integrity, and puzzle generation

## $$$$$$$$$$$$$

What does this cost to run? This will have to be part of my daily work now, call me the puzzle master. But seriously, it will take at most a few hours per day, on a good day, maybe 1. This work includes:
- Inbox triage
- PR review until test coverage earns trust for autonomy
- QA daily puzzle
- Keep an eye on users and billing

Budget-wise, there's also hosting + LLM spend + self-driving usage, plus some money for distribution. According to Slack, I have $10-25K. This feels reasonable.

## Open questions
- Legal/billing shape for payments
- Budget approved
- Art request for graphics team - how much work should they be expected to do here? How much do they WANT to help with? We can re-use as much original twig.com art as possible imo
