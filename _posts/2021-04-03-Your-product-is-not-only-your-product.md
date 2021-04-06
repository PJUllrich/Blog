---
layout: post
title:  Your product is not only your "product"
description: What comes to mind if you think about "Your product"? You probably think about features built for your customers. However, your "product" also includes something else. Your software.
date:   2021-04-06 07:01:35 +0200
tags:   [development, scrum]
---

What comes to mind when you describe "your product"? Do you think about features built for your customers? How it solves a problem your customers have? Maybe even a logo, a slogan, or its theme color?

These aspects describe your product's business side and [value proposition](https://www.investopedia.com/terms/v/valueproposition.asp). However, they fail to mention the foundation of your product: **its software**.

The reason you might have forgotten about the software part lies in the fact that product development tends to revolve around the central question:

> Which feature should we build next?

Answering this question is undoubtedly a critical step in product development. It justifies the need for a product and its development and is the bedrock of job security for everyone involved. Therefore, the most attention and perceived value lie on the business side of the product. Unfortunately, this causes the software and its maintenance to become nothing more than a by-product.

Think about the last time you worked on a purely technical ticket, the last time you spend most of a sprint cleaning up the codebase or restructuring your architecture. In short: Paying off your [technical debt](https://martinfowler.com/bliki/TechnicalDebt.html). If you did this just recently: Good for you. I mean it. If not: This is sadly more common than you might think, and it is a problem.

Ignoring the technical side of a product means neglecting its foundation, and the risk of an eventual collapse increases. The demise might start with a loss in agility or development speed. The pace with which you build new features will slow until it comes to a screeching halt. Bugs will occur more often and will become more severe until you spend most of your time fire fighting. Your stakeholders will become first concerned, then annoyed, and eventually: angry. Your once-happy customers will become less so and eventually will leave if they can.

If you want to avoid this scenario, you have to start including the technical side into the definition of "your product". In other industries, this is already common practice. Think about what makes a car a *good* car. It probably will be its engine and other mechanical parts, which allow its low maintenance, low consumption, and that it doesn't break down, ever. Carmakers know this and focus actively on maintaining and improving the technical side of their product as well. We need to adopt this perspective in software development too. The first step in this direction is to realize that:

> Your product is not only your "product"

"Your product" is not only a set of features and the problem it solves. It also includes its codebase, its architecture, its dependencies, and its technical debt. Once you acknowledge this, you need to adapt your workflow to incorporate this shift of perspective. Here are a few tips:

1. Minimize piggybacking tasks to reduce your technical debt on feature tickets. Don't make technical work like *clean up the codebase* part of developing new features. 
2. Move technical work to purely technical tickets like *tasks* or *chores*.
   1. Estimate and prioritize these tickets in your backlog.
   2. Don't make them inferior to feature tickets. Technical and feature tickets are equals from now on.
3. Plan your sprint with both technical and feature tickets. You decide about their ratio, but don't hesitate to have purely technical sprints if you feel like the technical debt overwhelms you.
4. Consider adding a timebox to every sprint for technical tasks that are too small for their own ticket.

The depreciation of technical work has led to many battles between "business" and "technical" people. Typically, developers try to get a time or financial budget for reducing the technical debt, but the "business people" reject them since they don't see the value in such work. I hope that this opinion piece convinced you to change your perspective on this issue.