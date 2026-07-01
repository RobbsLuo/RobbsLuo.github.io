---
title: How to Become an Excellent Architect
date: 2018-05-10 09:46:38
tags:
  - Architect
  - Humanity
categories:
  - Architect
description: What capabilities does it take to become an architect, and how should you work toward becoming an excellent one?
lang: en
---

## Broad Knowledge

In my view, architects have always been a talent pool for future CTOs, because the range of skills required is so broad.

> **Doing architecture, in its simplest understanding, boils down to one sentence: figuring out how to solve problems under various constraints.**

Those constraints are factors like `performance`, `stability`, `development efficiency`, and `maintainability`.

* An application like Baidu Tieba might see hundreds of millions of visits and tens of millions — or even hundreds of millions — of writes per day. Performance clearly has to come first, and you may have to trade some development efficiency for performance gains.

* In a banking scenario, user experience and access latency matter less, but data security and consistency are paramount. In that case, security and stability come first, with performance considered later.

Making tradeoffs under constraints means making a lot of choices. And to make choices, you first have to know what the options are.

For performance and security to mean anything beyond the words themselves, you have to know, in concrete technical terms, what solutions are actually available.

* For example, the Java ecosystem, the PHP ecosystem, the C ecosystem, and Python/Node.js/Golang each have their own strengths and weaknesses; you need hands-on development experience with them to make the right choice. Hearsay gives you no standing to decide.

* Although MySQL is by far the most widely used database today, Oracle and PostgreSQL have their own advantages.

* Almost no modern project avoids big data altogether, so you need at least some understanding of big-data algorithms and some hands-on experience with big-data platforms.

* Running an entire project involves a great many other concerns: how source code is managed, how releases are deployed, how testing keeps the bug rate down, system monitoring, server provisioning, canary releases, and so on.

Don't listen to people who make "architect" sound lofty and rarefied, as if it were purely a designer's job — system design, software design, and so on. That's just second-hand romanticizing.

**No internet company will hire you purely to do design, because without exception every internet company's leadership needs the implementation of features and the realization of strategic and tactical ideas.**

**Every large-scale system architecture is iterated step by step in response to the problems it faces. There is no scenario in which someone gets to design a brilliant system all in one go (even with a head start of a while), because the world changes too fast. If the project isn't implemented quickly, it may die tomorrow — who has the leisure to design for years into the future?**

**So the single most critical thing for an architect is the ability to steer the entire project and keep it running efficiently.**

## Excellent Coding Ability

If you want to become an architect, you first have to be an excellent programmer. So what makes an excellent programmer?

Just writing code without thinking or learning obviously won't do.

Concretely, you have to deeply master all kinds of data structures, design patterns, computer networking, operating systems, and common architectural patterns. These get mentioned constantly, but I suspect very few people truly understand them in depth.

I include myself: when I first studied design patterns, I felt within a month or so that I understood all of them. But every time I revisited the topic, or came across exceptionally well-written code, and went back to review this material, I always found something new and reached a deeper level of understanding.

And understanding is only the beginning. The real key is fully integrating these ideas into your own code.

Writing code is itself shot through with the feel of architectural design. In fact, I'd argue that when a programmer writes code it's called "coding," but when an architect writes code it's called "architectural design."

That's because the two consider problems from completely different angles. A programmer mainly thinks about how to implement a feature, while an excellent programmer also thinks about things like `performance`, `readability`, and `maintainability`.

For an architect, those concerns are mandatory, and the dimensions considered are usually even broader.

So don't imagine you can skip "excellent programmer" and jump straight to "architect" in one step. It's unrealistic.

## Depth in Certain Relevant Domains

I just talked about the breadth of technology. But if you know a little about everything yet excel at nothing — if you're not genuinely expert in anything — then you'll remain just a programmer.

So which domains count as key domains?

From here onward, architects essentially diverge according to their business direction.

For example, an architect in the financial domain may need financial knowledge.

In the big-data domain, deep knowledge of Hadoop, Spark, Hive, and similar technologies may be required.

In the high-concurrency domain, deeper expertise in system-wide performance optimization and distributed-system design may be needed.

## Technical Insight

"Technical insight" is a term I'm borrowing from the book *How Google Works* (originally 《重新定义公司》). A more intuitive phrasing would be "technical foresight and vision."

A few examples, with the benefit of hindsight:

1. If JD.com had not chosen the Windows platform back then, it might have developed far better than it has today.

2. If it weren't for the short-sightedness of a certain Baidu leader, who was always a few beats behind the market, Baidu wouldn't now be left far behind by AT.

3. ......

This shouldn't feel abstract. If you're currently an architect at a startup, a seemingly correct decision you make today could directly cause major losses for the company in the future — or even its collapse.



## Management Ability

Few architects lead neither projects nor people, so management ability is certainly a must.

But management is a huge topic in its own right, so I won't expand on it here.







------

Looking back, what I've written is all fairly directional.

But given the particular nature of the architect role, it's genuinely hard to lay out a concrete path such that, if you just follow it, you'll become an excellent architect.

There are, however, some principled approaches.

As an architect, so-called design ability is not really the key. There are few opportunities to design a project entirely from scratch, and in most cases you can simply combine other people's design solutions, weighed against the situation at hand.

That's why, once you've read enough architect-related posts, you realize that so-called distributed architectures and large-site architectures are basically the same handful of patterns over and over. As a result, anyone can come out and mouth a few sentences about "architecture."

So what is the key? It's the ability to steer a project, and to solve concrete problems as they arise.

And developing project-steering ability and concrete problem-solving ability sometimes requires more than just the projects at your company.

In company projects you're usually only one member of a team, doing one specific piece of work — frontend JS, an app, backend business logic, etc. You generally can't see how the project as a whole runs. Even if you're lucky and the project lead has the whole thing under control and is willing to explain it all to you, you still haven't built those parts with your own hands, and listening to someone describe them is unlikely to give you a deep understanding.

**So what should you do? Carve out time to build an entirely new project of your own — for example a BBS site, or an app with a back end. The site or app itself doesn't matter; what matters is that you develop it yourself, get the deployment system working, get the monitoring system working, get the release system working, get the testing system working, and so on.**

**Treat your own project exactly like an official product at a real company.**

After a few rounds of this, you'll find your sense of the entire project dramatically sharpened. You'll no longer be fuzzy about everything except the part you personally own.



---
> Source: https://zhuanlan.zhihu.com/p/27979747
