---
title: "17 Years in Java — The 10 Mistakes That Cost Me Promotions"
date: 2026-08-24
categories: [Career, Engineering]
tags: [career, software-engineering, java, tech-lead, programming]
description: "A candid retrospective on the technical and career mistakes I made as a Java developer in Singapore — and what I wish someone had told me earlier"
mermaid: true
---
I started writing Java professionally in 2009. Seventeen years, six companies, three countries, and one long stint in Singapore later, I'm finally comfortable admitting the mistakes that held me back. Not the bugs or production incidents — those are war stories we all share proudly. I mean the slow, invisible mistakes that kept me at Senior Engineer for years longer than necessary while peers moved ahead.

Some are technical. Some are political. All were avoidable.

## Mistake 1: Over-Engineering Everything

My biggest sin from years 2-8 was building cathedrals when a tent would do. I'd design systems with five layers of abstraction, plugin architectures, and event buses for applications that had three users and would be rewritten in two years.

I remember spending three weeks building a "configurable rule engine" for a loan approval workflow. My manager wanted an if-else chain. I built a Spring-based DSL with custom annotations, a rules evaluation framework, and an admin UI for non-technical users to modify rules. The non-technical users never touched it. The if-else chain would have taken two days and been easier to debug.

What it cost me: I was consistently late on deliverables. My code was "impressive" but hard for others to maintain. In reviews, I was described as "brilliant but unpredictable." That's not a promotion signal. That's a risk signal.

**What I learned:** The best engineers optimize for team velocity, not personal cleverness. The right level of abstraction is the simplest one that handles current requirements plus one reasonable extension. Not five hypothetical extensions.

## Mistake 2: Not Writing Tests (Until It Was Too Late)

For my first five years, I barely wrote tests. I was "fast" because I shipped features without test coverage, and I was proud of it. I rationalized: manual testing was fine, the QA team would catch bugs, tests slow you down.

Then I joined a team where I owned a payment gateway integration. No tests. A refactoring introduced a subtle decimal rounding bug. It went to production because nobody caught it during code review (the diff looked fine). We over-charged 2,000 customers by small amounts over three weeks. The total wasn't catastrophic, but the customer trust damage was real.

What it cost me: That incident showed up in my performance review. Not as "caused a bug" — bugs happen — but as "pattern of shipping without safety nets." My promotion packet was rejected that cycle with feedback about "engineering maturity."

**What I learned:** Tests aren't about catching bugs today. They're about proving you think about the future. A senior+ engineer who doesn't write tests is saying "I don't care what happens after I push this." That's exactly the attitude that blocks promotions.

## Mistake 3: Premature Optimization

Related to over-engineering but different. This was about spending days optimizing code paths that handled 10 requests per minute. I'd profile the application, find that a method took 50ms, and spend two days reducing it to 5ms. Meanwhile, the product team was waiting for a feature that would actually generate revenue.

The worst case: I once replaced a simple HashMap-based cache with a custom LRU implementation using a concurrent skip list because "it would scale better." The original HashMap handled our load fine. My custom implementation had a concurrency bug that caused a memory leak in production. We rolled it back to the HashMap.

What it cost me: velocity, trust, and a reputation for solving problems that didn't exist.

**What I learned:** Optimize when you have evidence of a problem. Profiler data showing a bottleneck under production load. Not theoretical analysis of what might happen at 100x scale when you're currently at 1x and growing 10% annually.

## Mistake 4: Staying at One Company Too Long

I spent five years at my second company. By year three, I'd mastered their stack, their domain, and their politics. Years four and five were coasting. I told myself I was "going deep" and "building expertise." In reality, I was comfortable and afraid of interviewing.

Meanwhile, peers who moved after two years were getting 30-40% raises at each jump. By year five, my compensation was 40% below market. When I finally moved, the salary jump was so large that it highlighted how much money I'd left on the table.

But it wasn't just money. Staying too long in one technical environment gave me a narrow perspective. I'd only seen one way of doing things. My new company used completely different architectural patterns, and I spent six months feeling like a junior again.

**What I learned:** In Singapore's tech market (and most places), two to three years is optimal. Long enough to ship meaningful projects, short enough to keep learning and keep comp market-rate. The exception: if you're getting promoted every 18-24 months internally, staying is fine. If you're not, the market is your promotion mechanism.

## Mistake 5: Avoiding Politics

I used to say "I just want to code" like it was a virtue. I actively avoided meetings that weren't technical. I didn't attend town halls. I didn't know what my skip-level manager cared about. I thought delivering good code was sufficient for career progression.

It's not. Promotions are decided in rooms you're not in, by people who may never read your code. Those people know you through two channels: what your manager says about you, and your visibility in the organization.

I watched a colleague with worse technical skills get promoted ahead of me because he presented at the company all-hands, joined cross-team working groups, and his name appeared in Slack channels that directors read.

What it cost me: three years of being "the quiet one" while less technically capable people advanced past me.

**What I learned:** Politics isn't a dirty word. It's relationship building. It's making sure decision-makers know your contributions. You don't need to be manipulative — you need to be visible. Volunteer for cross-team projects. Present your work. Write status updates that your manager can forward upward. Make it easy for your manager to advocate for you.

## Mistake 6: Not Speaking Up in Architecture Reviews

For years, I'd sit in architecture review meetings and spot problems but stay quiet. I'd tell myself: "They probably thought of that already" or "I don't want to seem confrontational" or "Who am I to question the principal engineer?"

This was especially acute as a non-local in Singapore's corporate culture, where direct confrontation is less common. I misread "don't be aggressive" as "don't disagree." They're different things.

Every time I stayed quiet and was later proven right, I felt frustrated. But I'd robbed myself of the opportunity to demonstrate senior judgment.

What it cost me: The perception that I was a strong executor but not a technical leader. "Follows designs well" is mid-level. "Identifies problems before they become expensive" is senior+.

**What I learned:** Frame disagreements as questions: "Have we considered what happens when X?" or "I'm curious about the trade-off between A and B here." This is culturally appropriate everywhere and still demonstrates architectural thinking. The worst outcome of speaking up is being wrong — and being wrong in a meeting is fine. Nobody remembers. They do remember who found the problem that saved a quarter of re-work.

## Mistake 7: Ignoring Soft Skills and Writing

I could write 500 lines of clean Java but couldn't write a one-page design document that a product manager would understand. I couldn't explain technical decisions to non-technical stakeholders. My PR descriptions were "fixed the thing" level.

When I finally started writing — design docs, architectural decision records, tech blog posts — two things happened: I became a better thinker (writing forces clarity), and people outside my immediate team discovered what I was working on.

What it cost me: years of my work being invisible to people who mattered for my career trajectory.

**What I learned:** Writing is the most underrated skill for engineers past mid-level. The ability to explain complex technical ideas clearly is what separates individual contributors from technical leaders. You don't need to be a great writer — you need to be a clear one. Structure, clarity, and honesty beat eloquence.

## Mistake 8: Gold-Plating Instead of Shipping

Similar to over-engineering, but this is about the last mile. I'd get a feature to 90% complete, then spend as long on the last 10% — handling edge cases nobody asked about, polishing error messages, adding configuration options for hypothetical flexibility.

My manager once told me: "You've been 'almost done' for two weeks." That stung because it was accurate. The business needed the feature live. My gold-plating was a form of perfectionism that looked like poor estimation to everyone else.

What it cost me: reputation for reliability. When I said "two weeks," people mentally added 50%. You can't be a tech lead if people don't trust your estimates.

**What I learned:** Ship the 80% version. Get feedback. Iterate. Real user feedback is worth more than your imagination of what might be needed. The "polished" version you're building in isolation often misses what users actually care about.

## Mistake 9: Not Building Relationships Outside My Team

For the first decade of my career, my professional network consisted of my immediate team. I didn't attend meetups, didn't contribute to open source, didn't maintain relationships with former colleagues.

When I wanted to move to a new role, I had to cold-apply. No referrals, no introductions, no "hey, we have an opening you'd be perfect for" messages. Every job search started from zero.

What it cost me: longer job searches, less negotiating leverage, and zero exposure to how other companies solved problems.

**What I learned:** Your network isn't about getting jobs (though that's a benefit). It's about perspective. Talking to engineers at other companies shows you different approaches, different trade-offs, and different career paths. In Singapore's relatively small tech scene, a strong network is incredibly high-leverage.

## Mistake 10: Thinking Technical Skill Alone Determines Career Trajectory

This is the meta-mistake that underlies all the others. I operated under a mental model where "best engineer = fastest promoted." This model is wrong.

Promotion decisions weigh:

- **Impact** — did your work move business metrics?
- **Scope** — did you operate beyond your immediate team?
- **Leadership** — did you make others more effective?
- **Communication** — can you explain and persuade?
- **Technical skill** — are you competent? (Note: competent, not exceptional)

Technical skill is table stakes. Necessary but not sufficient. The engineers who get promoted fastest are the ones who combine decent technical skill with high impact, broad scope, and clear communication.

I've seen average coders become engineering directors because they excelled at items 1-4. I've seen exceptional coders stay at Senior for a decade because they only had item 5.

**What I learned:** After 17 years, I define seniority differently. A senior engineer isn't the one who writes the most elegant code. It's the one who consistently makes the right trade-offs — in code AND in career.

## What I'd Tell My 2009 Self

If I could go back, I'd say:

- Ship fast, iterate faster. Done is better than perfect.
- Write tests. Not for the code. For your reputation.
- Change companies every 2-3 years until you're in a genuine growth trajectory internally.
- Speak up in every meeting. Wrong is better than silent.
- Write publicly. Teach what you learn.
- Relationships compound. Invest early.
- Your manager is not your enemy. Help them help you.
- The best code is the code you don't have to write. Simplicity is the ultimate sophistication.

Seventeen years is a long time to learn lessons that could have taken five. But I'm writing this so maybe someone reads it at year three and saves themselves a decade of quiet frustration.

Your career doesn't happen to you. You build it. And the building has far more to do with people, communication, and positioning than it does with code.
