---
layout: post
title: "New job! New field of science!"
date: 2016-07-12 20:36:59 +0530
comments: true
categories: [Job, Life, Rust]
---

It's been about a year since I've blogged. A lot of stuff has happened in the mean time. I became a [reviewer for the Servo browser engine](https://twitter.com/ServoDev/status/684472921818017794) - especially the python code (which felt good), attended a flight training program at IIT, Kanpur (which was pretty fun), had a *war* with some of the professors (which has postponed my bachelors degree, *meh*), and now I'm working for a bioinformatics company, writing production code in **Rust** (which is cool!).

<!-- more -->

While I was doing my final year project, I applied for an internship at a [bioinformatics company](http://www.genomels.com/). For the first week or so, it was just python and shell scripts (boring stuff, really), until one day, I ported some of the python code to Rust and
gave a demo on that. That was it! From then on, until the end of the internship, and now, my job, is totally on Rust!

All these days, they've been using third-party tools for their analysis, connecting them with shell pipes, tinkering the output with a few scripts, and finally bringing it to the front-end. Now, there's an opportunity (for them) to break their painful dependencies, research on things, write stuff from scratch, while I can get deeper into systems programming, and simultaneously get an actual experience in writing production code (in Rust!). So, it's a "win-win" situation!

As an example, there's this Java-powered tool called [FastQC](http://www.bioinformatics.babraham.ac.uk/projects/fastqc/) which analyzes [FASTQ](https://en.wikipedia.org/wiki/FASTQ_format) data. With the help of some bioinformatics fellas, it's pretty easy to reconstruct what the tool does, from its output. By the end of the internship, I was asked to rewrite the tool (ASAP!). They helped me with the spec, and it took me exactly 12 days (~70 hours) to write the Rust version of that tool. [^1]

The initial version doesn't have any kind of *unsafe* code, and I didn't optimize it very well. It only utilizes the APIs in the [standard library](https://doc.rust-lang.org/std/) for efficient reading, data storage, and parallelization, but it was already ~20% faster than its (carefully crafted) *rival*, and there's no limitation for this. Now that we've got our own tool, we can have as many features we want! [^2]

Anyway, I'm pretty sure we're the only ones using Rust for production [in our state](https://en.wikipedia.org/wiki/Tamil_Nadu) (right now), and I'm quite happy about this, because my job allows me to play with the language I love, and I've got more than enough learning space here. So, maybe I'll stay around for a while, and see how far this is gonna take me...

Even though there are languages specifically designed for scientific computing (like Julia, for example), I personally believe Rust has a great future in big data analysis!

[^1]: I'm planning to blog about it soon.

[^2]: They were actually impressed by this, and it got me the "game changer" award :P
