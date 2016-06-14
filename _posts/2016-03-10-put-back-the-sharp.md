---
title: "Putting the `#` back in operator names?"
layout: post
---

A while ago I [removed the `#` that LES/EC# adds at the beginning of all operator names](http://loyc.net/2014/in-operator-names-to-be-removed.html). Having written quite a few macros now, I am starting to think that was sort-of a mistake. The reasons for removing it seemed good, but one of the reasons turned out to be kind of bogus.

I had thought "well, operators should really just be thought of as ordinary functions... they're not really as special as mainstream languages lead us to believe... so why should their name have any clear indication of their specialness?" I've encountered a couple of problems with this:

1. With Loyc trees there's no way to ask "is this a _normal_ function call or a _pseudo_ function call, such as a dotted expression or an identifier with generic type arguments?" The problem here is that although _most_ operators (like `+` or `%`) work very similarly to functions, some of them fundamentally don't, and sometimes we need to tell the difference. For instance, a macro could easily mistake `.` and `::` for method names, which would nearly always be a mistake.
2. Although arguably there's little need to distinguish computational operators (like `==`) from normal functions, there's far less justification to be able to name a _type_ or a _variable_ as if it were an operator. If a macro wants to distinguish syntactically between operators and variable/type names, that shouldn't be difficult.

Really the main reason to get rid of `#` was for pedagogy (teaching): I don't want people to get hung up on what ``@`#=`(x, y) `` means. But if it serves a necessary purpose, so be it.

So, now what do we do? I have a couple of options in mind:

1. Designate certain specific operators as special, such as '`.`' and '`::`', and add the '`#`' prefix back on these operators only. LES could have a special rule that any operator starting with '`.`' or '`:`' gets a '`#`' on the front. But the decision of _which_ operators should be classified as special is uncomfortably arbitrary.
2. Introduce the <s>space character</s> single quote `'` as a prefix for all operator names. Visually, a quote is much less noisy than `#`, so learners might find it less distracting. There may be reasons to want two separate "specialness" prefixes (`#` and `'`), and a single machine instruction can test for both, if we can also agree that certain prefixes are "special". Specifically, we can use `if (firstCharacter <= '\'')` which will classify `'` and `#` and a few other characters as special (`'` is ASCII 39.) **Edit**: alternately, we could [use `'` instead of `#` for *all* operators and keywords](http://loyc.net/les/juxtaposition-discussion.html#keyword-statements).

Sadly, I don't expect anyone to comment on my blog anymore, so I'm not doing the extra work required to enable comments on this post.
