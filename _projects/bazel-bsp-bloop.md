---
layout: project
title: 'Bazel-BSP + Bloop'
caption: > 
  Bazel-BSP is a BSP plugin for IntelliJ (and other IDEs supporting the build server protocol).
links:
  - title: GitHub
    url: https://github.com/JetBrains/bazel-bsp
sitemap: false
---

Bazel-BSP is a BSP implementation for the bazel build tool.  Bazel-BSP + Bloop builds upon this by allowing
users to export the project model to [Bloop](https://github.com/scalacenter/bloop), resulting in significantly faster compilations (at the expense of a possibly less accurate model import).

In real-world use cases, projects imported via Bloop have rebuild times on the order of milliseconds, vs 1-5 seconds using 
bazel natively.