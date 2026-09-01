# Engagement autocapture - concept doc

## **Problem statement**

When a customer is doing product analytics, they often want to know how “engaged” a user is with a part of the product. But the definition of “engaged” is difficult to pin down. Sometimes it can be defined by a simple click, but often it’s less clear. It might mean the customer:

- Is on a certain page for a given amount of time
- Has a specific element is in the viewport for some time
- Is hovering their mouse over something for a bit
- They’re clicking around within a certain element
- They’re typing on the page
- Or something else

With today's analytics products, these are difficult events to capture because they aren't based on a single action that the user took, but rather, it's how the user engages over time.

As evidence, in our own instrumentation of PostHog, we use timers to trigger some of our most useful events - `insight analyzed` and `recording analyzed` (the basis of our discoveries metric). Both of these are based on a 10 sec timer of the elements being open on a page, but the implementation is messy, and the metric isn't that reliable. As one example, `insight analyzed` would fire if the user opens the insight in a new tab, never looks at it, but leaves the tab open for 10 seconds. Ideally, our `insight analyzed` would be based on the chart being in the viewport while the user is focused on it and moving the mouse around.

Outside of PostHog, I’ve also seen a similar pattern of core metrics being based on “time something was in the viewport”. This is particularly popular in the content and e-commerce industries (e.g. how long was this product in the viewport before it was clicked). Additionally, I had a conversation with a friend who was excited about using PostHog and thought this was what "autocapture" meant.

## **Design**

This feature needs to be dead simple for customers to implement. One idea for what it could look like is (meant to spur discussion more than be the “correct way”):

- Basic engagement metrics are auto-added to the $pageleave event:
    - Time on page
    - Time focused on page
    - Time engaged on page (window focused, mouse moving etc.)
    - Number of clicks on page
    - Maybe more
- A data attr, `data-ph-engage`, is added to any HTML element that should be tracked. If this attr is set, we will send up events:
    - ElementEnteredViewport
        - All autocapture properties
        - Any element metadata added in the `data-attr`
    - ElementLeftViewport
        - All autocapture properties
        - Any element metadata added in the `data-attr`
        - Time 50% of element is in viewport
            - Total time
            - Time when user is “engaged” (window focued, mouse moving etc)
        - Time 100% in viewport
            - Total time
            - Time when user is “engaged” (window focued, mouse moving etc)
        - Click count inside element
        - Key stroke count inside element
        - Time mouse hovered in element


If we added these features to PostHog-js, then instrumenting our insights page would be as simple as adding a `data-ph-track-engagement` to the insight container, and then any insights about how useful a useful feature is could be based on this.

## **Success criteria**

We will have succeeded if we can make it super simple for users to track “engagement metrics”, and they find those metrics useful.

We can measure this by:

- The number of customers using tracking engagement metrics
- The number of customers with insights based on engagement metrics

