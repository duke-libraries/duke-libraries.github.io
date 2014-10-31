---
layout: post
title: Extracting Repository Models
author: David Chandek-Stark
tags: hydra refactoring ruby rails models
---

When we started build a repository application with the [Hydra](http://projecthydra.org) framework nigh about two years ago, nearly everything about the code environment was new to us -- including Ruby, Rails, RSpec and Git.  While we had a respectable level of repository domain knowledge (by no means experts), we tended to follow coding and testing patterns we observed within the Hydra development community.

I think we got the conceptual modeling mostly right.  We stuck to models that seemed to satisfy our initial use cases and hoped that they would give us a good framework to refine as we rolled out the repository to a broader audience.