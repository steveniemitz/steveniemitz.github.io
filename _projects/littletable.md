---
layout: project
title: 'Littletable'
caption: A Bigtable emulator written in Scala
links:
  - title: GitHub
    url: https://github.com/steveniemitz/littletable
sitemap: false
---

Littletable is a JVM based alternative to the Google-provided Bigtable emulator.  It aims to provide feature parity with the Bigtable API, more so than the Go emulator does.  For example, Littletable has full support for RE2 binary regexes, SINK filters, etc.

Additionally, for JVM applications, Littletable is much simpler to set up and use than the Google version since it is a native JVM library and can run in-process.