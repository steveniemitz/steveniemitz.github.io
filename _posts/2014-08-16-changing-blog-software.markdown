---
layout: post
title: Changing Blog Software
date: '2014-08-16 16:36:04'
---

I decided I wanted to try out a new blog platform, driven mostly by the fact that my current (old) one looked kind of crappy and outdated.  I like the more modern, simpler, cleaner look that a lot of newer platforms have, and I wanted to move away from a HTML based editor, since I do mostly coding.

I ended up chosing [Ghost](https://ghost.org), a very simple blogging platform running in node.js.  I also chose to host it in [Google Compute Engine](https://cloud.google.com/products/compute-engine/), a competitor to Amazon's EC2 and azure.  I'm running on a f1-micro instance, which should be more than enough to run this blog.  It's 1 "CPU", and .6 GB of RAM for less than $7 / month.

Migrating from BlogEngine.NET -> Ghost wasn't really a supported path, so I decided to write a quick little utility to take the BlogML XML file BlogEngine.NET can output and convert it to the [Ghost import format](https://github.com/TryGhost/Ghost/wiki/Import-format), which has a pretty close 1-1 mapping to the BlogML schema.  I've put the utility on [GitHub here](https://github.com/steveniemitz/BlogML2Ghost) in case it's helpful to anyone else.

I've set up rewrite rules on my old blog (hosted on ASP.NET) to redirect to this new site, along with the same for the RSS feed, so the transition should be seamless.

So far I like ghost a lot, it's super simple to use and set up, and looks really nice.  I haven't even played around with other themes yet either.