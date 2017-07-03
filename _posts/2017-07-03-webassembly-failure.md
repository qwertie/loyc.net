---
title: "My failure with WebAssembly"
layout: post
toc: false
---

With WebAssembly, there's no doubt that I failed completely. I knew how important it was - as I told my wife, "if I can make this happen, it will be the most important thing I have ever done in my life." But I allowed myself to lose confidence and let fear of failure - fear of being ignored - to take over.

The main thing I wanted to contribute to WebAssembly was to have an LES-compatible text format, so that, among other things, any language with an LES parser could parse WebAssembly text. Thus, each programming language would not need to have its own separate implementation of a WebAssembly text parser. If I could succeed with LES, there were other things I wanted to do too. Specifically, I created an organization on GitHub called [wasio](https://github.com/wasio), meaning WebAssembly Standard for InterOperability, but I never even made a single commit. Sad!

It became fairly clear to me as time went on that the key decisions about WebAssembly were being made behind closed doors in Silicon Valley. The first really clear signal about that was the announcement (mid-2016) that they would switch from a expression language to a stack machine. Now, while it's a large change conceptually, it turned out that it was actually a fairly small change in terms of implementation. But the point is, the decision was made with no public discussion, they just suddenly said one day "hey everyone, here's how it's gonna be now".

So with LES, they pretty much consistently ignored my proposal. The only person (with a connection to the browser devs) that liked it was Alon Zakai, but in my conversation with him I got the impression that he was relatively powerless himself, that he saw his role as being to just accept whatever the main devs decided.

I responded to the change to a stack machine by noting that it actually wasn't a big deal, there's no reason you couldn't use an LES-like syntax for WebAssembly with a stack machine. But I think my comment about that was ignored. Due to time constraints they also wanted to drop the expression-like syntax; but again this was fine, and really didn't impede LES at all. Loyc trees and LES are meant to be "universal", after all!

I was a little disturbed when they announced via PR early this year that the version number was to be changed to 0x1 (It was agreed long ago that the switch to 0x1 meant that the WebAssembly standard was finalized - that once the version number became 0x1, it would never change again). Again, it didn't seem like a discussion to me, it was more like "oh, by the way, WebAssembly is done and we're about to ship it."

A couple of months earlier I made a really modest suggestion to [change the first eight bytes](https://github.com/WebAssembly/design/pull/928) of all WebAssembly files. As before, no one objected to my proposal - they simply ignored it. So by the beginning of 2017 I already felt convinced - or I convinced myself - that they weren't going to pay any attention to my suggestions, so I got really down and completely stopped trying.

At that point, when the version switched to 0x1, I just kind of assumed their custom text format was also planned out and implemented. I'm not sure though; the actual the PR for the text format came in two or three months later.

There was another guy called wlllang (formerly JSStats) in the WebAssembly CG, and frankly I wish I was more like him, because he was pushing his ideas at every opportunity. He had some personality flaws himself - he was a bit judgmental and belligerent, he didn't always seem to understand what the main developers were proposing, and he did not tend to express his ideas very clearly. But I think his heart was in the right place - that is, he wanted WebAssembly to be something really great. And more to the point, he pushed his opinion fairly relentlessly, despite the fact that the main developers usually ignored him.

Maybe if I had had his forthrightness and consistently pushed my opinion (without the belligerence), maybe they would have let me design the text format. And honestly, why did they want to use up their own engineers' precious time designing a custom format when I was already happily designing LESv3 for free? I can't know what they were thinking, but I know it's at least partly my fault for not finding ways to reach out and push more.

On the other hand, the decision to ignore LES and make a custom text format was made behind closed doors and since (AFAIK) that decision was never announced, I did not have an opportunity to object to it.
