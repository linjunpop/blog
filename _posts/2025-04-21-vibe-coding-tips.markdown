---
layout: post
title: "Vibe Coding Tips"
date: 2025-04-21 18:00:00 +0800
---

Some tips for [Vibe coding](https://en.wikipedia.org/wiki/Vibe_coding)

## README.md

Make the README.md file the guide and memory for AI

[Rust RFC's template](https://github.com/rust-lang/rfcs/blob/master/0000-template.md) is a very good template for README.md

Make use of the Cursor rules:

```
---
description: README.md
globs:
alwaysApply: true
---

Before doing anything, always read the @README.md file.
It contains important information about the project.
```

And please remember to instruct AI to update the README.md file to reflect the current state of the project.

## Try and throw away

When the project goes to wrong direction, you can throw away the project and copy the README.md file to new project then start over again.

## Make AI make a plan before coding

Generating a plan is kind of bridge between human and AI. It can help human to understand and tweak the plan before AI start coding.
