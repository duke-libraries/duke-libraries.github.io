---
layout: post
title: "Hydra Rules: Thoughts on Community Software Development Practices"
author: David Chandek-Stark
---

**Disclaimer:** Don't bother looking at my code to see if it's consistent with the suggestions made here.  It isn't.  I'm still learning.

### Why do we need rules for community software development?

If you haven't seen [Sandi Metz's talk on Rules](https://youtu.be/npOGOmkxuio), stop reading and watch it.

Now, I don't believe Sandi when she says her rules are completely arbitrary.  I suspect that's a bit of rhetorical hyperbole to make the point that *having rules* is more important than the specific rules you follow.  It's in that spirit that I write this article.

Informally I have adopted at least two of Sandi's rules: 

- A method may have no more than 5 lines
- A class may have no more than 100 lines

In my experience so far the second rule is much harder than the first, since it's usually easier to refactor a method than a class.  Note that I treat these rules less as laws and more as "gauges" -- to use a visual metaphor, a red light goes on when a rule is "broken".  It doesn't necessarily mean something is wrong, but that you should probably check it out.  Or to use another metaphor in programmer parlance, it signals a "code smell".

I think the real motivation behind the method and class length rules is that smaller methods and smaller classes are easier to comprehend, test, and use.  With a short method, you can often see what it does at a glance.  Classes, of course, are more complex ... and that's where I'll make a pitch for a corollary to the class length rule: You can't cheat by factoring out pieces into one-off "concerns" -- or, Just Because You Can, Doesn't Mean You Should.

### What's the Concern?

One of the chief offenders of readable, testable, and thus, useable code IMHO is the oft-named "concern" (a.k.a. "mixin", "behavior").  I found [Corey Haines's article](http://blog.coreyhaines.com/2012/12/why-i-dont-use-activesupportconcern.html) helpful in clarifying my sense of the problem here.  There is, obviously, a legitimate use of "mixins" for programming aspects that cut across many objects.  But I think it's at best a code smell when chunks of code are factored out of class only to be re-added via `Include`.  There is a superficial gain in readability by chunking class code that way, but it comes at a cost by making large classes falsely appear smaller.  At worst, these chunks have cross-dependencies, which only makes them more confusing and hard to test.

Unfortunately, Ruby on Rails (or at least its creator) [encourages this practice](https://signalvnoise.com/posts/3372-put-chubby-models-on-a-diet-with-concerns) by adding a conventional place to put your "concerns".  While I can see how this may not be a Bad Thing if your app is smallish and self-contained, the practice doesn't scale within and across apps, where the ability to quickly comprehend different parts of the codebase becomes very important.

Rather than chunking out pieces of classes, we should instead think about breaking down large monolithic objects into compositions of smaller objects -- each of which will be easier to understand, test, and use.

### Pull Requests Welcome?

I finally decided to write this article after reading Jeremy Friesen's fine presentation, ["Accepting Application Ownership"](https://docs.google.com/presentation/d/1TvjNVuQyEOwrITIgcd7J2HMV5BvgG-XRqqGGVtrW4tY/edit?usp=sharing). His text mentions "inconsistent styles and idioms" as factors in making apps expensive because of the "higher code-orientation cost".  To that observation I would add that community-developed software such as Hydra can evolve in a way -- hard to understand and test -- that makes the cost of contribution prohibitively high for many developers who have only slivers of time to devote to such efforts.  Therefore, the community as a whole benefits from adopting principles that foster more readable code that is easier to modify and extend.  I suspect most folks would agree with that statement in theory, but in practice we tend to be more focused (and not without good reason) at the micro level of immediate functional goals -- or, concretely, pull requests.

My point is that the programming culture of the community -- our de facto practices -- matter.  We all want the functionality of the software.  But we also have to use, maintain, modify and extend it to meet new needs and our coding patterns determine the costs of those future efforts.  

And, sorry, I don't have a pull request that makes it all better. :)
