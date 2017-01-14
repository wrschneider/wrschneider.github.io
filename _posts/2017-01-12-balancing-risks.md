---
layout: post
title: Balancing early and later project risks
---

One of the things I liked about this post on ["Senior Engineers Reduce Risk"](https://hackernoon.com/senior-engineers-reduce-risk-5ab2adc13c97#.9brfu91rj) is how it called out two different kinds of project risks:

-   Early in a project lifecycle, the biggest risk is building the wrong thing
-   Later in the project lifecycle, once you know you’re building the right thing, the “-ilities” (scalability, maintainability etc.) become bigger risks

The author’s point is that senior engineers need help identify and mitigate these risks. 

One additional responsibility of a senior engineer, in my opinion, is to understand the tradeoffs between these kinds of risks, and how to balance those tradeoffs.  This is tricky because, to paraphrase Yogi Berra, predictions are hard--especially about your future user load or revenue.  You can think of this like type 1/type 2 error (false positive/negative) in hypothesis testing:

-   Type 1 error: premature optimization/generalization/etc.  You spend time scaling something that doesn’t sell, or designing a generic platform that only gets used once.  
-   Type 2 error: technical debt.  By the time you realize you have a scaling problem it’s too late, and your users end up unhappy.  Or, your lack of CI processes and tests slows down future releases.

The type 1 vs. type 2 metaphor assumes you have constrained resources - an engineering hour spent on scaling is an hour not spent on prototyping to get feedback from users.  So reducing one kind of risk will increase the other kind of risk and vice versa.

Given that both kinds of error are bad, what do you do?  You have to balance the possible outcomes from these risks, and prioritize based on what’s more important to you, and this is context-dependent.  A senior engineer should know how to reach out and communicate with business stakeholders to figure out the right balance, telling a good story about the risks that may not be immediately evident to non-technical team members.  A senior engineer will have lived through both kinds of errors and can draw from their past experience in their storytelling.

My own personal opinion: after living through projects with both kinds of type 1/type 2 errors, I would rather take type 2 over type 1 most of the time.  37signals sums this up with the mantra ["It's a problem when it's a problem"](https://gettingreal.37signals.com/ch04_Its_a_Problem_When_Its_a_Problem.php).  The catch is you have to be disciplined enough to identify and communicate future risks, *and* have a plan to address them if and when they become issues.  It can be OK to defer scaling if and only if it is a deliberate, conscious tradeoff to prioritize something else, so there are no surprises later.   

This is also why "debt" is a good metaphor.  In personal finance, some kinds of debt are good because they help reach a strategic goal: buying a house, getting an education, starting a business.  Other kinds of debt are bad: racking up credit card balances without a plan to pay them off.  Similarly, deferring some "-ilities" in pursuit of a higher priority business goal can be a good thing, while ignoring them outright is bad.