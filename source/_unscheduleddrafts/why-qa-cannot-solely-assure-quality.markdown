---
title: Why QA Cannot Solely Assure Quality
author: chris
published: false
comments: true
---

It is in the name, right? "QA" does "Quality Assurance" (or 
sometimes "Quality Control"). But a traditional QA team can 
only watch over a narrow part of the product's quality,
namely "does the product do what the customer wants now,
without any obvious defects". Traditional QA *cannot* make
sure that those defects will be easy to fix, that future
functionality will be feasible, or that anyone hired will
ever be able to maintain the code.

<!-- more -->

My point is that *Developers have to care about quality*. This
might be obvious on your team, but it isn't uncommon for quality
to be either not thought about or the first victim when there is
a time crunch.

What do developers get when they concentrate on code quality?

1. Code that easily changes with the requirements
 * Obvious where and how to make changes
 * Number of lines that must be touched is constrained 
2. Confidence that changes haven't broken a random other feature
 * Dependencies on modified code are easy to find
 * Test coverage allows for quick verification of regressions
3. Shorter ramp up time for new developers joining the team
 * Well organized code means the developer doesn't need to 
   understand *all* of it to understand *any* of it
 * Processes help guide new developers in consistent practices
   with the rest of the team

So there are definite upsides to having high quality code. In a
future post, we'll talk about some of the reasons code quality
gets thrown under the bus!


