# RFC: Wizard goes to the top of the funnel 

This RFC is basically half proposal and half update on work that's already in flight.

[Slack thread](https://posthog.slack.com/archives/C09GTQY5RLZ/p1771446043522759) for context.

## TL;DR 

We want to advertise the wizard as the default way to get started with the PostHog platform. 

We've already started, but we want to crank up the volume. Both internally and externally. 

This means:

1. Highlighting the wizard command even more across the website, docs, and in-app
2. Doing a marketing push and potential outbound
3. Creating dev tools or a process for all product eng teams to add their products to the wizard

There's no big launch moment or hard deadline, but the wizard will be "top-of-funnel ready" by **March 17**.

## Overview

The v2 wizard for Next.js has been live for ~3 months. The v2 wizard for 15+ languages and frameworks has been live for a [few weeks](https://posthog.slack.com/archives/C0351B1DMUY/p1770833365394869). 

Here are a few metrics of the business impact so far: 

1. 2x faster conversion to first event
2. 2x conversion in creating first dashboard
3. 2x conversion to first billable event within 1, 7, and 30 days
4. 8-10% overall paid conversion rate for signups
5. Mine calculated some early [GDR and NDR](https://posthog.slack.com/archives/C09GTQY5RLZ/p1771450253709449) numbers

Some dashboards [[1]](https://us.posthog.com/project/2/dashboard/1289703)[[2]](https://us.posthog.com/project/2/dashboard/1317108)

The #team-docs-and-wizard has spent most of Q1 rebuilding the architecture, creating a new [context system](https://github.com/PostHog/context-mill) for creating agent skills, and shipping new features every week. 

Given the promising data and our shipping velocity, we're ready to move the wizard closer to center stage at the top of the funnel.

## Proposal

### Website, docs, and in-app updates

The main idea is to spread the `npx @posthog/wizard` CTA everywhere we can. 

Last week we launched a spiffy new [marketing page](https://posthog.com/wizard) and [docs page](https://posthog.com/docs/ai-engineering/ai-wizard) for the wizard. So we're covered on dedicated landing pages. 

Additional updates would be small and straightforward. 

1. Update the website homepage CTA to show `npx @posthog/wizard` CTA instead of the `Install with AI` button. Something like [this](https://res.cloudinary.com/dmukukwp6/image/upload/q_auto,f_auto/SCR_20260301_nrdi_4d2e62386c.png) @cory to save us a click
2. Update all product installation pages in the docs with the wizard CTA for supported products
3. Update in-app onboarding with the wizard CTA for supported products
4. Update blog posts or tutorials with wizard CTAs to try PostHog
5. Any other places we can think of

### Marketing

I don't want to be prescriptive and will let @joe and the marketing team cook. 

Here are a few general marketing areas where we think the wizard is ready for:

- **External promotion** – This doesn't need to be a formal launch. The story is simple: getting set up with the PostHog platform and multiple products has never been easier. Just one command. The wizard has a built-in sign-up flow as well. Up to the marketing team how we promote this and roll this out.
- **Onboarding and lifecycle** – For users who haven’t activated yet or are stuck in the funnel, we could send an email with the wizard as the CTA.

Other outbound thoughts:

- We could also use the wizard CTA for ads if the demand gen team is interested.
- We're also starting to explore using the wizard for lead enrichment for sales. 

### Onboarding more products

We’ve reworked the Wizard’s architecture to be more modular and composable. With the docs, context mill, wizard, and MCP working together, we now have a scalable foundation to support the majority of products over time.

We're creating a new wizard UX so product engineering teams can plug their products into the wizard more easily. This work is already underway and we’ll formalize it soon, but it's not a blocker. 

## Why not now?

There are a few areas we need shore up. The main concern is making sure we don't introduce regressions or leaks at the top of our funnel if something goes wrong. For example, if there's an Anthropic outage and the wizard goes down, we need fallbacks so users are routed to alternative onboarding flows without disruption. 

We need two weeks to optimize and build contingencies so we don't disrupt signups and activation. If all goes well, we think signups should increase. There's other work, but availability and reliability are the most important things to address.

We've already discussed with @joshua and broken down the tasks we need to complete to be "top-of-funnel ready". 

![figjam of our to-do list](https://res.cloudinary.com/dmukukwp6/image/upload/q_auto,f_auto/pasted_image_2026_03_01_T21_34_53_862_Z_5100e1fdd1.png)

## Work to be done on our end

Anything that isn't a P0 can be safely shipped as a fast-follow. That said, we're planning to ship all P1 items.

- **P0:** Reliability and safeguards
  - LLM gateway and 3rd-party spikes and rate limits
  - CI on merge and cron hourly/daily
  - Incident creation if wizard goes down
  - Runbook for rollback and revert procedures
  - Level up PR evaluator
  - Address rate limit requests to posthog.com
  - Build fallback messaging for wizard outage: “They take this skill with you, sorry the wizard is down”
  - Frontload health check: ping all services
  - Dependent services: MCP, Anthropic, context mill, posthog.com, posthog 

- **P0:** Signups and onboarding
  - Sign-up flow improvements
  - AuthZ screen improvements
  - Analytics improvements
  - Support for FF and experiments in wizard phases
  - FF kill switch for homepage CTA

- **P1:** Product support 
  - LLMA, Logs, Feature Flags, and Experiments
  - Third-party Stripe detection
  - Extensible support for new products via wizard, MCP, or hybrid
  - Skills library sourced from context mill

- **P1:** UX and features
  - TUI overhaul and refresh
  - Monorepo handling 
  - Base languages: web JS, Node.js, Python, Ruby
  - Error messaging improvements
  - Custom events

- **Q2:** Fast follows
  - Anything that misses the deadline
  - More languages: Rust, Go, Elixir, Electron, Unity, Flutter 
  - Non-npx support
  - Product support for Surveys, Endpoints, etc.