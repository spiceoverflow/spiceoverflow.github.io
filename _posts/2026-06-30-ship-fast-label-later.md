---
layout: post
title: "Ship Fast, Label Later"
date: 2026-06-30
excerpt: "Cloudflare is using 'experimental' banners on OSS repos as grounds to close bug bounty reports as out of scope. I don't know if it's about report volume, vibecoding liability, or something else entirely. This is me thinking through what it means for how I hunt."
---

> **tl;dr** Cloudflare ships AI-generated OSS that real developers use in production, and when bugs get reported, the "experimental" label in the README becomes the reason to close them as out of scope. Whether that label was always there or not doesn't matter — it's being used as an exit hatch. If that becomes the norm, it's worth talking about.

## The Setup

A few months ago I started hunting on Cloudflare's bug bounty program, targeting their open source repositories on GitHub. Packages actively published to npm, real developers building real things on top of them. I found real issues — account takeovers, RCEs, the kind of stuff that has concrete impact in a production deployment. I reported methodically. Most got triaged.

Then one of my higher-severity reports got closed with reasoning that boiled down to: *the README says experimental, so this is out of scope.*

## The Vibecoded OSS Problem

The repositories I was looking at weren't toy projects. They were AI agent frameworks, shell packages, developer tooling — things with real npm download counts and active promotion on the Cloudflare blog. They were also vibecoded: shipped fast, largely AI-generated, with security as a distant afterthought.

This isn't speculation about Cloudflare's process. They've been open about it — in January they [published a blog post](https://blog.cloudflare.com/serverless-matrix-homeserver-workers/) claiming to have implemented a Matrix homeserver on Cloudflare Workers, which turned out to be AI-generated code [littered with `// TODO: Check authorization`](https://tech.lgbt/@JadedBlueEyes/115967791152135761) where the security-critical logic should have been. The README was updated to say "not endorsed as ready for production" after the backlash. Ship it, blog about it, disclaim it when it becomes inconvenient.

This is where the accountability gap lives. Cloudflare gets the credibility of open source — community trust, developer adoption, a bug bounty program that signals they care about security — while the "experimental" label becomes a pressure relief valve when the bill comes due.

The problem isn't that experimental software exists. The problem is that experimental software with no security guarantees was already being consumed by developers who had no reason to know that. Slapping a warning on it after the fact doesn't protect those users. It just protects the program.

If the experimental label were a principled policy, you'd expect it to apply consistently. It doesn't. The [vinext](https://github.com/cloudflare/vinext) repository carries nearly identical language — *"🚧 Experimental — under heavy development... the vast majority of the code, tests, and documentation are written by AI... Use at your own risk"* — and is explicitly listed as in scope and bounty-eligible on the same program page. The label isn't the rule. It's selectively the rule.

## On AI-Assisted Hunting

I used AI to help find these issues. Heavily supervised — I micromanaged the tooling, validated everything independently, wrote all the PoCs myself. The reports were not slop. But I understand the underlying anxiety from program owners: AI has genuinely lowered the floor for report volume, and not everyone hunting with it is being careful.

Here's the thing though: that's a volume problem, not a validity problem. Closing a technically sound, fully reproduced, high-severity finding as informational because of an experimental label isn't how you handle noise. It's how you use noise as cover for something else. And that's worth naming, even if uncomfortable.

## What This Means for How I Hunt

Honestly? It makes me not want to hunt on corporate OSS targets anymore.

That's not a decision I'm happy about. I genuinely enjoy finding bugs in community-driven open source — sometimes I report things for free just because they matter. The security of the OSS ecosystem is worth caring about independent of bounties.

But there's a meaningful difference between a scrappy open source project with no budget and a well-funded company using an open source repo as a marketing surface. The former deserves goodwill. The latter is running a program with commercial incentives, and when those incentives shift, the experimental label is right there waiting.

I'm not done hunting OSS. I'm just a lot more selective about who's on the other side of the report.
