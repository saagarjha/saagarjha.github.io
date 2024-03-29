---
layout: post
title: "Why we at $FAMOUS_COMPANY Switched to $HYPED_TECHNOLOGY"
---

When $FAMOUS_COMPANY launched in 2010, it ran on a single server in $TECHBRO_FOUNDER's garage. Since then, we've experienced explosive VC-funded growth and today we have hundreds of millions of daily active users (DAUs) from all around the globe accessing our products from our mobile apps and on $famouscompany.com. We've since made a couple of panic-induced changes to our backend to manage our technical debt (usually right after a high-profile outage) to keep our servers from keeling over. Our existing technology stack has served us well for all these years, but as we seek to grow further it's clear that a complete rewrite of our application is something which will somehow prevent us from losing two billion dollars a year on customer acquisition.

## Why switch?

As we've mentioned in previous blog posts, the $FAMOUS_COMPANY backend has historically been developed in $UNREMARKABLE_LANGUAGE and architected on top of $PRACTICAL_OPEN_SOURCE_FRAMEWORK. To suit our unique needs, we designed and open-sourced $AN_ENGINEER_TOOK_A_MYTHOLOGY_CLASS, a highly-available, just-in-time compiler for $UNREMARKABLE_LANGUAGE. Even with our custom runtime, however, we eventually began seeing sporadic spikes in our 99th percentile latency statistics, which grew ever more pronounced as we scaled up to handle our increasing DAU count. Luckily, all of our software is designed from the ground up for introspectability, and using ~~some BPF scripts we copied from Brendan Gregg's website~~ our in-house profiling tools $FAMOUS_COMPANY engineers determined that the performance bottlenecks were a result of time spent in the garbage collector.

Initially, we tried messing with some garbage collector parameters we didn't really understand, but to our surprise that didn't magically solve our problems so instead we disabled garbage collection altogether. This increased our memory usage, but our automatic on-demand scaler handled this for us, as the graph below shows:

![The inevitable conclusion of scalable architecture](MoarAWS.png)

Ultimately, however, our decision to switch was driven by our difficulty in hiring new talent for $UNREMARKABLE_LANGUAGE, despite it being taught in dozens of universities across the United States. Our blog posts on $PRACTICAL_OPEN_SOURCE_FRAMEWORK seemed to get fewer upvotes when posted on Reddit as well, cementing our conviction that our technology stack was now legacy code.

## Pivoting to a new stack

We knew we needed to find something that could keep up with us at $FAMOUS_COMPANY scale. We evaluated a number of promising alternatives that we selected and ranked based on the how many bullet points they had on their websites, how often they'd appear on the front page of Hacker News, and a spreadsheet of important language characteristics (performance, efficiency, community, ease-of-use) that we had people in the office fill out.

{% include aside.html type="Fun fact" content="As our spreadsheet met strong statistical guarantees of randomness, we were able to reuse it to replace our application's CSPRNG." %}

After careful consideration, we settled on rearchitecting our platform to use $FLASHY_LANGUAGE and $HYPED_TECHNOLOGY. Not only is $FLASHY_LANGUAGE popular according to the Stack Overflow developer survey, it's also cross platform; we're using it to reimplement our mobile apps as well. Rewriting our core infrastructure was fairly straightforward: as we have more engineers than we could possibly ever need or even know what to do with, we simply put a freeze on handling bug reports and shifted our effort to $HYPED_TECHNOLOGY instead. We originally had some trouble with adapting to some of $FLASHY_LANGUAGE's quirks, and ran into a couple of bugs with $HYPED_TECHNOLOGY, but overall their powerful new features let us remove some of the complexity that our previous solution had to handle.

Deploying the changes without downtime required some careful planning, but this was also not too difficult: we just hardcoded the status page to not update whenever we pushed new changes, keeping users guessing if our service was up or not. Managing incremental rollout was key: we aggressively A/B tested the new code. Our internal studies showed that gaslighting users by showing them a completely new interface once in a while and then switching back to the old one the next time they loaded a page increases user engagement, so we made sure to implement such a system based on a Medium article we found that had something to do with multi-armed bandits.

With our rewrite now complete and rolled out to all of our customers, we think the effort has been a massive success for us and our team. We have measured our performance and you can see a summary of the results below:

![Stonks](Stonks.png)

Every metric that matters to us has increased substantially from the rewrite, and we even identified some that were no longer relevant to us, such as number of bugs, user frustration, and maintenance cost. Today we are making some of the code that we can afford to open source available on our GitHub page. It is useless by itself and is heavily tied to our infrastructure, but you can star it to make us seem more relevant.

## Final thoughts

It's often said that completely rewriting software is fraught with peril, but we at $FAMOUS_COMPANY like to take big bets, and it's clear that this one has paid off handsomely. While we focused on our backend changes in this blog post, as we mentioned before we are using $FLASHY_LANGUAGE in our mobile apps as well, since we don't have the resources to write native applications for each platform. Unfortunately ~~to increase lock-in~~ these rewrites also mean we will be deprecating third-party API access to our services. We know some of our users relied on these interfaces for accessibility reasons, but we at $FAMOUS_COMPANY are dedicated to improving our services for those with disabilities as long as you aren't using any sort of assistive technologies, which no longer work at all with our apps.

We hope that you internalize our company's anecdote as some sort of ground truth and show it to your company's CTO so they too can consider redesigning their architecture like we have done. We know you'll ignore the fact that you're not us and we have enough engineers and resources to do whatever we like, but the decision will ruin your startup so it's not like we'll see *your* blog posts about your experience with $HYPED_TECHNOLOGY anytime soon. If you're not in a position to influence what your company uses, you can still bring it up for point-scoring the next time a language war comes up.

If you're reading this and are interested in $HYPED_TECHNOLOGY like we are, we are hiring! Be sure to check out our jobs page, where there will be zero positions related to $FLASHY_LANGUAGE.
