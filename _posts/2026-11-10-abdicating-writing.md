---
title:  "Abdicating writing to LLMs"
author: cldellow
tags:
  - llm
  - norms
---

I've been reviving my [Hiker Atlas](https://hikeratlas.com) hobby project. My interest in it has been renewed by LLMs making a lot of the tedious scutwork easier. Hey, if I have to face the downsides of my profession being destroyed, I may as well enjoy the upsides of software development being ~free.

One consequence of LLMs is that development cycles happen faster. Much faster. `tilemaker` gives an example -- [performance work that took me 3 months in 2023](https://www.hikeratlas.com/blog/claude-code) is now being done in days or weeks in 2026.

When you iterate faster, you also learn faster. I'm[^1] running more experiments. I'm rolling out new features. I'm exploring more ideas. When they work, I want to capture them, both as a reference for myself, and as a resource to which other map devs can be referred.

Writing then becomes a bottleneck. Do you accept that, and produce hand-crafted text at the cost of making progress on the app? Do you let an LLM write up the discoveries? Are you obliged to disclose the use of an LLM?

Everyone can reach their own decisions here. For me personally, I'm landing on two policies, split by whether the publication is one-to-many (a blog post) or one-to-one (an email).

For one-to-many communication:
- It's OK to have an LLM draft an article.
- If an LLM drafts an article, say so.
  - This lets readers assess how much of your effort went into the post, and thus calibrate their own expectations.
  - An example Hiker Atlas blog post that follows this policy: [ARM runners are 1.5x faster](https://www.hikeratlas.com/blog/arm-runners)
- If an LLM drafts an article, review it for accuracy and terrible prose.
  - Should someone complain about the quality of the text, don't take it personally. Fix it. It's not your text, why should you be offended when someone dislikes it?
- Using an LLM to suggest edits is fine, and does not require disclosure.
  - If you are the one incorporating the edits, your voice and personal judgment are still in play.
- Using an LLM to wrangle data, generate charts, etc, is fine and does not require disclosure.

For one-to-one communication:

- LLMs should never draft. It violates the social norms of personal communication, and [can lead to this ridiculous outcome](https://marketoonist.com/2023/03/ai-written-ai-read.html):
![A cartoon about AI-written content](https://marketoonist.com/wp-content/uploads/2023/03/230327.n.aiwritten.jpg)
*Cartoon by [Tom Fishburne, Marketoonist](https://marketoonist.com/)*
- LLMs as reviewers that provide critiques are OK, but shouldn't be directly editing your content.

Oxide Computer Company's [Using LLMs at Oxide](https://rfd.shared.oxide.computer/rfd/0576) policy seems similar, although it further differentiates one-to-many communication into public writing that represents the brand (don't use LLMs) and operational writing that is reference material (LLM use is OK). I consider the blog posts on Hiker Atlas to fall in their "operational writing" category more than their "public writing" category.

[^1]: Well. "I". Claude. Whoever. Whatever?
