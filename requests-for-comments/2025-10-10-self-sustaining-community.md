# [OST] The PostHog social network: Creating a self-sustaining community

_Don't judge this concept by the title. Even @andyvan-ph said this proposal was better than it sounded by the name._ 😉

Despite multiple attempts over the years, “PostHog community” is something we have yet to nail. While we have passionate customers, we lack a place for them to gather online.

(This is in comparison to communities like Figma where users can contribute templates and plugins, or Shopify where users can create themes and build apps on the platform. There’s also an aspect where users can make money from their contributions to both of these communities.)

### What hasn’t worked

| **Things we’ve tried** | **Goal**                                                                                                                                                                       | **Why they failed**                                                                                                                                                                                      |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Community Slack        | A thriving community of people within our ICP who share ideas, help, and learn – in the same way there are industry-standard Slack communities                                 | Became a de-facto support forum which team members didn’t monitor. Discussions also disappeared after 90 days as Slack is cost-prohibitive for a large community.                                        |
| Online forums          | Reduce support tickets, scale knowledge by having common questions indexed.<br /><br />Create a place for our customers to hang out and discuss things outside of our product. | Treated as another support channel. Engineers aren’t proactive in monitoring them for questions they can answer.<br /><br />We built it, but have no internal champion to dogfood it and drive adoption. |
| Events (historically)  | Increase buzz/word of mouth                                                                                                                                                    | We were too small at the time / couldn’t pull enough of a crowd / didn’t execute in the right physical location                                                                                          |
| Post upvoting          | Run our own HN-style leaderboard of content interesting to product engineers, both authored by us and submitted by the community                                               | Never got critical mass, didn’t fully execute, nor have the hook to get people to add to their bookmarks bar next to HN                                                                                  |

### What _has_ worked (for growing our audience)

-   **Content** has always been a strong reason for the growth of our brand. With over 100,000 subscribers to the newsletter, there is a clear interest in topics we cover.
-   **Building in public** - we’ve grown our audience by people being curious in how we do things
-   **Thought leadership** - similar to above, we turn our learnings into content. We share about what has/hasn’t worked for us, in the form of online posts, speaking at events, and doing interviews. People want to know more about how PostHog does things because we’re different and it’s a success story that stand out.
-   **Events** _not_ centered around PostHog - the recent stuff Zaltsman has been doing is showing early signs of working.

The first three bullets are generally one-way (we talk, people listen), but the last point is the most important in my mind since it doesn’t require our direct involvement.

As an analogy, the best dinner parties are the ones where the host could disappear and everyone else still has a great time. An unsuccessful dinner party would be where no one has fun unless the host is there (likely because they don’t know anyone/have anything in common).

For a PostHog community to be successful, it has to scale, and the only way for it to scale is to have people outside of the company driving it.

## One goal, many initiatives

We have many people/groups within PostHog working on various things to foster community or general adoption, but they’re scattered between people/departments/initiatives:

-   **Community forums** live on the website, but most of the topics are just about specific products
    -   We’ve built a community karma system to reward users for being helpful in the forums, but we haven’t launched it yet as it needs some tuning, and also needs to be connected to a reward system to actually make the karma valuable
-   **Newsletter** is doing well, but is one-way
-   **Content hubs** have great content, but aren’t organized in a way where they’re referenced as the go-to resource for what they cover
-   **Events** (so far) have a small network effort (since they’re localized) and take a ton of effort to make into a success
-   **Sponsorships** are good for brand awareness and creating goodwill in the community
-   **Online posts/shit-posting** are great for the brand, and we see moderate engagement here
-   **Merch** has worked as a brand awareness/loyalty play
-   Our **Cool Tech Jobs** board provides mutual benefit: it brings an audience to us, and also sends candidates to the companies who list on it

## Untapped potential

### Opportunities

| **Opportunity**                            | **Problem**                                                                                                                                                                                                                         | **Idea**                                                                                                                                                                                                                                                   |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Co-opting the “Build in public” community  | (No “problem” here, just that the community seems to be on X, which makes sense as it’s the best opportunity for distribution)                                                                                                      | Give them a place to get eyeballs on what they’re building                                                                                                                                                                                                 |
| Internal team cohesion                     | The team is growing and it’s impossible to keep up with Slack. The signal to noise ratio between important announcements or demos and general discussion makes it hard to follow anyone outside your small team and a few channels. | A place to share demos where the team can consume them on their own time, and as an added bonus, our customers would find a lot of this interesting, too.<br /><br />The `/wip` page could be resurrected if we can get people sharing what they’re doing. |
| Connect offline events to online community | PostHog-sponsored meetups happen, but they tend to end there, outside from sharing photos on X                                                                                                                                      | A way to signify in a community profile that a person did a thing, like: led or attended an event, won a hackathon, etc. Reward with exclusive merch/other benefits.                                                                                       |

We’ve also had some hackathon ideas over the years that we haven’t gotten to yet, but I’d still like to see happen – plus also some newer ones:

_Internal-focused, but also interesting to our community_

-   **Offsite/small team offsite pages** - a place to see a rundown of each meetup we do: who attended, location, what was discussed, hackathon demos
-   City guides: recommendations on good places to stay, work, and insights for team leads about how to make the best use of time and put on a successful small team offsite – based on our personal experiences

_External-facing ideas_

-   **Cool Co-working Groups (another fine Zaltsman idea)** - a way to find in-person communities for like-minded people, similar to what [buildspace.so](http://buildspace.so) was
-   **Company-wide AMA** (a Vandervell idea) - ask PostHog anything and the relevant person can respond about how we did a thing
-   **Sharing cool customer side projects** – not to compete with ProductHunt, but a way to highlight things our users are building to our 100k+ audience
-   **Video library** - from talks and things

## The opportunity

With all these cool initiatives we have going on (plus things we’ll do in the future), we have the chance to bring them all under a single umbrella online that would create an opportunity to discover other initiative you might not otherwise discover. Imagine:

-   reading content in our newsletter (in the future, hosted on our website instead of a third-party) where you can also see events or group meetups in your area
-   online discussions around our hub topics, ie: growth engineering, design for product engineers, or marketing or fundraising for founders
-   visiting our changelog where you can also see other demos of WIP work
-   the untapped potential for cross-sell

## The proposal

The cool part about PostHog is that people are genuinely interested in following our story. We have an opportunity to dogfood a thriving, public community if we put in the effort. But we need to dogfood it.

So I’m proposing:

-   we repurpose the `/community` section of our website to feel more like a social network, combining all of these various community-tangential elements into a feed
-   when you visit posthog.com, if your project ID is in local storage or you have a community account, the `/community` page opens by default (instead of `home.mdx`)
-   we slice up all-hands demos and share the ones we can post publicly, “authored” by the person sharing
    -   we encourage team members to share more demos here – both for our team to enjoy, but also for our audience, so we can build hype about new features/products
        -   (we already have a Slack shortcut to copy internal posts to various forum topics)

## How it fails

| **Problem**                                                                                                                                                                                                                                                                                                                                                                                                                                                       | **Solutions**                                                                                                                                                                                                                                                                                                                                                      |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| No one owns it                                                                                                                                                                                                                                                                                                                                                                                                                                                    | We’re hiring a community manager and this could fall under their purview                                                                                                                                                                                                                                                                                           |
| No internal buy-in / team isn’t incentivized to use it.<br /><br />A problem we have with engineers is that they want to build products, not deal with customer support or marketing. Communication tends to happen on the platform with the least friction: Slack. Getting people to change their workflow is hard. This even includes social channels like #where-in-the-world, #devel-random, or various topic/interest channels like #food, #{city.name}, etc | 1. We mandate that this is where WIP work gets shared by default (unless it’s sensitive/needs to stay private) – similar to how we used to do Thing of the Week in all hands.<br /><br />2. We use our (existing) Slack shortcut to copy internal posts to various forum topics<br /><br />3. We give kudos / shout outs / social validation to encourage adoption |
| Content isn’t interesting                                                                                                                                                                                                                                                                                                                                                                                                                                         | I don’t think this will be as much of an issue, but it may need to be curated so it always feels fresh and interesting.<br /><br />Unified auth isn’t the solution in and of itself, but the more we know about the user, the more we can show relevant content (things like role, length of timing using PostHog, products used, etc) |

## Why it’s worth a shot

Even though this sounds like a big endeavor, most of the pieces are already built. It really comes down to a UI play, along with a strategy to make sure it stays fresh, unlike the existing `/community` page.

But by combining so much content into a single place, I’m not too worried we won’t be able to make it interesting.

## Old wireframe

I put this together a long time ago (before we launched the new site), so I can envision the social network/community being an entire "app", maybe something along these lines:

<img width="2208" height="2388" alt="image" src="https://github.com/user-attachments/assets/6f29be81-95da-4ef9-90ed-bc973bf799b8" />
