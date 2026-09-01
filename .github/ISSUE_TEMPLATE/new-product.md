---
name: New Product
about: Steps you need to take when starting a new product here at PostHog
title: '[New Product] <PRODUCT NAME>'
labels: new product
assignees:
    - PostHog/team-growth
---

Team Lead: _Insert team lead @_
Team Members: _Insert team members @_
Exec: _Insert exec who team lead reports into @_

> The team can decide who's responsible for completing these. Most of it makes sense to fall under the team lead figure but none of it requires any administrator privilege

## Setting up a product

-   [ ] As soon as you start seriously planning a new product, add it as an [early access feature](https://us.posthog.com/early_access_features) as a `concept`. You can read more about it [here](https://posthog.com/docs/feature-flags/early-access-feature-management)
-   [ ] Inform the Marketing team a new roadmap item is available via the [#team-marketing](https://app.slack.com/client/TSS5W8YQZ/C08CG24E3SR) channel
-   [ ] Create a new folder in the `products/` folder in the PostHog repo. You should include at least a `manifest.tsx` file defining the base structure of your product.
    -   [ ] Category should be `"Unreleased"` to avoid showing up in customer's sidebars too early.
-   [ ] Create an alpha feature flag in `constants.tsx` (and [in PostHog](https://us.posthog.com/feature_flags)) to hide your product behind it. This should NOT be the same feature flag as the one in the preview roadmap. You'll wanna change it to the preview roadmap one only once you get close to releasing it to customers.

## Moving the product to Alpha

-   [ ] Slowly add customers you've talked into using your product to the feature flag
-   [ ] Update `manifest.tsx` to include your product under an accurate category other than `"Unreleased"`

## Moving the product to Beta

Beta is when you open up the product to all users who want to opt-in. See [releasing as beta](https://posthog.com/handbook/product/releasing-as-beta) for detailed guidance.

-   [ ] Update your code to use this feature flag rather than the old one
    -   [ ] Make sure you've moved over alpha users to the new feature flag before switching the flag
-   [ ] Ensure your roadmap item has a feedback link and docs link
-   [ ] Make sure you're capturing [product intent](https://posthog.com/handbook/growth/growth-engineering/product-intents) for your product already. You should keep iterating on what's intent and what's not.
-   [ ] Ensure your roadmap item has a payload with the `ProductKey` you defined in your `manifest.tsx` file: `{ "product_intent": "your_product_key" }` or else people won't see your product in their sidebar
-   [ ] Move your roadmap item from `concept` to `beta`
-   [ ] Inform the marketing teams a new beta is available via the `#team-marketing` channel
-   [ ] Let `#team-growth` know about the move to get your product added to user's sidebars - product should be added to everyone who already has a product from the same category as the new product in their own sidebar

## Launching a new product (GA)

**If you're planning to launch your product on a specific quarter, you MUST let the Marketing team know about it at the start of the Quarter.**

-   [ ] [Create a Pricing RFC](https://github.com/PostHog/product-internal/blob/main/requests-for-comments/templates/request-for-comments-new-pricing.md?plain=1) to coordinate with the Billing team
-   [ ] Inform the `#team-marketing` channel that a new Pricing RFC has been created, they'll create a [Launch Plan](./launch-plan.md) for you
-   [ ] Continue to communicate timelines in the `#team-marketing` channel
-   [ ] Add your product to the Onboarding flow, `#team-growth` can help with that but you can simply copy it from other products
-   [ ] Update [`posthog.com`](https://github.com/PostHog/posthog.com/blob/master/src/lib/productInterest.ts) to capture page views on your docs to redirect people to your product during onboarding
-   [ ] Include your product to the "Quick Start" section with some basic
-   [ ] Figure out what your [activation metric](https://posthog.com/product-engineers/activation-metrics) is, you can get a PM to help you in case your team doesn't have a PM yet, ask in the `#pm` channel for help
