---
layout: post
title: "100 shades of green: The journey of a coder..."
date: 2015-09-07 18:33:18 +0530
comments: true
categories: Coding
---

I've been coding for about a year now. I've danced with Python *(a lot!)* and nowadays, I'm playing with Rust, although I've also done some basic C & Javascript. Anyways, I get a lot of questions from my fellow undergrads about how I got into coding in the first place, and yesterday, [Manish](https://manishearth.github.io/) gave me the idea to blog about it.

Also, since my commit streak has reached 100 days (with 1k commits), I think now's a great time to share my story with y'all...

![commit streak](/images/streak.png)

<small>On a side note, this post is intended for those who're about to get involved in the *art* of coding, though I assure the rest of you that it will be interesting for others as well.</small>[^1]

<!-- more -->

## Good ol' days with the computer...

When I got my first computer, all I ever wanted to do was play games (well, I still do, but I love other things too). Apart from that, I liked to paint (you know, lines, squares, circles, colors, wheee...). Though I did encounter C in my school days, I didn't know the purpose of it, because I simply didn't realize the powers of coding. The only thing that attracted me was HTML (4), because it can do some pretty stuff - that way, I realized that with some text I can make colorful things on a computer!

I discovered many things during my high school days, but the only thing that mattered the most was that characters in computers can do more than just coloring - that was the time when I played with matrices and generated prime numbers in C (basic stuff you learn in high-school). That was also the time when I had developed this weird desire to fiddle around the things which are hidden from plain sight. I just don't like it that way.

For instance, take the beautiful "Microsoft Windows XP" (the OS of my first computer) - the `C:\\` drive is forbidden by default, and it has a lot of files which are unknown to me. I had a friend who shared my trait of fiddling around with unknown files. Since his father owned an internet cafe, whenever we're lost, or screw up something (or get screwed up by a virus, mostly a worm), we get a solution almost immediately. Later, we take those problems to our school, so that we can screw up the computers in our labs (let's just say that we hated some of our teachers and we wanted to make their lives miserable!).

That was the time when I also got interested in the command prompt (which I terribly hate nowadays, since I got into bash). For example, we used this tiny snippet for recursively walking inside a path and hiding the contents inside. It was used often by us to drive the teachers and students crazy (for a month!).[^2]

``` bat
attrib [path] +h +r +s /s /d
```

## As a newbie to the internet...

It wasn't until college when I got my first laptop along with a *pathetic* excuse for an internet connection (which I still have). By the end of my fresher year, a great deal of things had happened.

I got into Stack Exchange, [which had some great impacts on my personality](http://wp.me/p3OCmi-D) over the years (that's also where I met Manish). At the start, I had the motivation to explore the system, interact with a lot of users, and learn to contribute, but once that became handy, I got bored. Yeah, reading stuff on the internet, participating in a great community is awesome and all, but I needed something more - something that could keep me equipped when I'm bored of reading.

I got excited about math and wanted to solve problems - it was right about that time when I discovered [Project Euler](https://projecteuler.net/). It demanded the users to code to solve the problems. I'm sure every newbie coder gets interested in those problems, because it demands only the solution for a particular problem, which means you ought to guess an algorithm first (sometimes, it'd be the awful bruteforce), try to implement it (when you'll discover more about the language of choice - with the help of [Stackoverflow](https://stackoverflow.com/), of course), get the result and then you can refine the algorithm (unlike SPOJ or Code Chef, which are designed to concentrate more on algorithms and data structures). In those days, I used C.

My sophomore year began with [blogging](https://wafflescrazypeanut.wordpress.com) - when I was mostly writing about physics-ish stuff (whatever I got excited about) or my favorite experiences (or ranting about something). At this point, I knew about *Mathematica* and its glories. Luckily, the Wolfram community had offered plenty of documentation for it, that I was able to learn its basics pretty quickly.

Now, I had no reason to use C for solving the problems, since *Mathematica* had all the built-in functions available in hand. I used it to generate plots and some simple animations, which I later embedded in my blog posts. One thing led to another and soon, I was making animations for teaching my classmates to visualize things.[^3]

Along the way, I also got fond of a markup language - which affected me so much, that I got addicted to just seeing its *beauty*. It was the awesome LaTeX! I used to prepare notes for some of my courses with it, just because I felt happier to read all those silly formulas in LaTeX. Once I got to know about its clockworks, I began using [LyX](http://lyx.org) and [Geogebra](http://geogebra.org) to speed things up.

## Hands on an easy, aesthetic & powerful language...

Then, I discovered Github. I didn't understand the point of it at first, and so I assumed it as a "dropbox for coders" - I didn't know about the concept of version-control or open-source. Anyway, the repositories served nicely as a *backup place* for all my code.

I'm a *perfectionist*, alright? I admire the beauty of things when they're neat, but I often struggle to maintain their clean state as time progresses. Since I realized that LaTeX seemed to consume most of my time, note-taking also came to an end. And, *Mathematica* (the only language I was currently working with) seemed too abstract, since it hides all the details from my view. Now, I wanted to know how things work behind the cloak, but I also don't wanna fall back to C. *(meh)*

When I was a sophomore, I used to hang out a lot in our physics chatroom, [the H-bar](https://chat.stackexchange.com/rooms/71/the-h-bar) and most often, I could hear about Python. When I asked about it, they told me that the scientific community is using Python most of the time (from handling the data to plotting the results), especially because it's easy. *"Hmm, looks like something I should try out..."*, and I got curious just like that.

By then, I somehow got interested in cryptography. After reading about the old [Caesar](https://en.wikipedia.org/wiki/Caesar_cipher) & [Vigènere](https://en.wikipedia.org/wiki/Vigenère_cipher) ciphers, I got a desire to create my own cipher (that often happens if you get too *excited* about cryptography without thinking about the past few decades of research), and that little project of mine consumed my entire vacation at the end of my sophomore year.

On the brighter side, this was the point where an endless discovery was going on. As days passed, I dug deeper and deeper into Python. Some new idea kept popping up every now and then, and I immediately implemented it. Man, I was totally productive! As a free perk, I also [translated the code to Javascript](https://github.com/wafflespeanut/scripts/blob/0f1b6ee61b3929eacce4fa32638ff328e6f8858e/site-old/javascripts/zombify.js) (by which I learned some HTML5 and JS along with a bit of CSS). This was also where my life as a coder began...

The day came soon, when I learned some stuff in public-key cryptography, when I realized about the depths of cryptography, when I'd also *finally* decided to **ditch all my work** because it just seemed too *stupid* to keep on developing a nonsensical *beast* which does nothing but consumes more time and memory in the name of a "[non-conformant pet cipher](https://github.com/wafflespeanut/scripts/tree/0f1b6ee61b3929eacce4fa32638ff328e6f8858e/python/AutomatonX)"! I had to *move on*. By then, I knew some serious stuff in Python (thanks to Stackoverflow which helped me to learn all the way down to its internals), and so I used it to reduce the work in my academic stuff - like grabbing data from the lab machines, minor computations, iterations and plotting (which simply took too much time in Excel - some of my classmates realized that later).

Every time I encountered a problem requiring repetitive steps, I used Python. Since Python can do many things (given the vast amount of packages it has) - I used it to crawl through webpages to download stuff, switched to Python for solving those old Project Euler problems, clean my files (and sometimes, dirty code). Like I said, I'm a *perfectionist*.

After a few months, I got an idea (for another little project!). I often forget what happened every day (which is one of my problems which I had to solve) and so, I wanted something to keep track of my memories. That "something" was my next project - [a diary](https://github.com/wafflespeanut/biographer). Apart from Mozilla, that project was something that consumed a great deal of time, and every single line in it was *molded* by me over the months. Ideas popped up in the same way - *"Let's have an option for searching!"*, *"Authentication per session would be a great idea!"*, *"Hmm, wouldn't it be wonderful to have [CBC](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Cipher_Block_Chaining_.28CBC.29) before shifting?"*, and I implemented those as and whenever they cropped up in mind.[^4]

## Into an awesome community - open-source and beyond...

Anyways, that thing had taken some of my time and since I was indulged in it, I never bothered to look into Project Euler, and so my problem-solving days were done (I simply didn't get the mood!). Now, I needed something else - something I've never tried in my internet lifetime.

Then, I recalled Manish's talks about Mozilla, Rust, etc. He had often told me that Mozilla's one of the most welcoming communities to get involved. One of the most important things which Stack Exchange taught me is to **"search more before asking questions!"** Because, we often get silly questions at the site, where some users don't even bother to put some effort on their questions. The point is that it was pretty hard for me to ask questions (especially when it's a new place). Also, I had never heard of "IRC".

I was afraid. The gigantic codebase and the numerous code it contained freaked me out! I've never seen a codebase that big in my life! *"Millions of lines of code? C'mon! Seriously? It's just a browser!"*. I never understood the bugs, nor did I know how to patch those. Soon, my vacation began, and my sole aim was to get involved in a Mozilla codebase! [^5]

I got into IRC and discovered that they offered mentors for new users. I got one for myself, who even suggested me a "good first bug", which took me some days to finally submit a patch. That also required me to get into Ubuntu and a new version control system called *Mercurial*. My first bug explained me how things worked over there. It also came with a perk - I never used Windows from then on! I realized the awesomeness of Ubuntu! [That was it](http://wp.me/p3OCmi-uA), and later bugs became a piece of cake, thanks to those awesome Mozillians.

It was by that time Rust 1.0.0 was released. I never got to play with a low-level language. The release of Rust and its success was my motivation. I got excited about it, and immediately got indulged in it (with the help of the [wonderful docs & book](https://doc.rust-lang.org/)). Now, an idea popped up while reading about [concurrency and FFI](https://doc.rust-lang.org/book/rust-inside-other-languages.html) in Rust (that was my first time reading about "FFI", though I had some idea about concurrency). Anyways, that diary I'd written was completely in Python. So, If I could somehow link it through FFI and hand over the searching to Rust (and utilize its concurrency), then I could save a lot of time. And, it did.

This task consumed an entire weekend of mine (but, it was worth it!). Though FFI was hard, I got help from a lot of Rustaceans (mostly at IRC for minor things, or Stackoverflow, whenever [things went out of hand](https://stackoverflow.com/q/31083223/2313792)). I loved Rust so much (just like I loved Python), which is also why I never felt hopeless, and by the end of the day, I [got the thing to work!]({{site.url}}/2015/07/08/a-pythonist-getting-rusty-these-days-part-2/) *(finally)*

## Coding everyday...

There's something that I learned from my journey. Once I got into the art of coding with the help a language, jumping into other languages wasn't a big deal. Moreover, if you get involved in a project (open-source, or something of your own), then you'll more likely learn a great deal of stuff than you normally would (though that cipher and my diary wasn't much useful, I did learn a hell lot of things while working in them).

Also, since I'd gone for a high-level language (Python), I didn't have to worry about the types very much, but I did get into trouble when I tried to get into a low-level *beast* like Rust. Because, Rust doesn't *abstract away* many things like Python does, I had to do quite a few tasks by myself! Moreover, Rust's design made my coding pretty challenging, because of its static type system and its merciless borrow checker (among other things). **I had to pay the iron price!** (for my choice of a high-level language at the start). But, [I had quite a lot of fun]({{site.url}}/2015/07/05/a-pythonist-getting-rusty-these-days-part-1/) while playing with it.

Nowadays, I'm involved in the [Servo browser engine](https://github.com/servo/servo), to learn more about Rust, which gets pretty exciting every day. And, while I'm playing with Rust, I've also got back into C, because firstly, I've realized the power of these low-level languages (plus, C doesn't have any abstractions at all!), and secondly, I'm about to finish my college and industries only know about established languages like C/C++/Java. So, I gotta learn one of those to get a job *(sigh)*.[^6]

<small>Thanks to Manish for *patiently* reviewing this post...</small>

[^1]: If you'd grown in my locality (in your childhood), then all you know is that a computer (which you often see in school) is *"something"* where you can only type, draw, and play games - nothing more! I'm not embarrassed to say that we were never curious!

[^2]: Well, we learned that from a virus (and that too because we never believed in the concept of an anti-virus, since everything is just the cause & effect of code). This command is often utilized by a harmless worm which infects pen drives.

[^3]: It was for a course, where we draw vector diagrams for the velocities and accelerations of various parts of some machine. I'd even [made some boring videos](https://www.youtube.com/playlist?list=PLpqCCxmTKpa1NBYfoEJlKb8X4fq0Pf6N9) for the junior undergrads using those animations.

[^4]: It's not a great *project* and all (I created that only to dig deeper into Python, and I have!). And yeah, I do write my story every day (I've got about a year of stories now). I don't usually spend much time other than just recalling the events, and I just write a paragraph or two (like hints) - to just remind myself that something so nice/worse has happened on that day.

[^5]: Meanwhile, I [got into a hackathon](http://wp.me/p3OCmi-ws), when I learned a lot of stuff - more JS and some frameworks - especially the MVCs like Django and Angular JS, and I'd also figured out how Git worked.

[^6]: Well, to be honest with you, based on the things I've seen so far, I personally feel like C being quite easy, since I've learned the *"Rust"*.
