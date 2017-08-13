---
layout: post
title: "Exploring the human genome (Part 1)"
date: 2017-02-12 16:54:14 +0530
comments: true
categories: [Bioinformatics, Job]
---

You may already know that [my work in bioinformatics]({{site.url}}/blog/2016/07/12/new-job-new-field/) is mostly, well, *research*. All these months, I've been writing little tools in Rust (things that help speed up some *boring* analysis). These days, I'm involved in something new, something *very interesting!* Before we get into all that, I'll try to give a general overview of the flow of data (without going way too much into biology), what kinds of data we deal with, how we analyze it, where I come in, etc., starting from this post.

<!-- more -->

## Preamble - Reading the DNA...

I'm sure you already have an idea about the DNA - the double helix thing, a bunch of ATCGs, the genetic code, the basis of life, etc., but I'll tell you a better story. To begin, we should go *deeper*, all the way down to the nucleus of a cell.

In there (for humans), we'll find 23 pairs of wiggly thingies (chromosomes, if you want). That's where we'll also find the tightly packed and coiled DNA strand. Each chromosome has a specific set of nucleotides (ATCGs) and are labeled based on their size - one has ~250 million of them, and so it's "Chr. 1", another one has ~50 million of them, and so it's "Chr. 22".[^1]

A genome is a collection of all the genes. A gene is just a sequence of bases used to manufacture a protein (more on this next time). When we say "human genome", we mean the whole thing, starting from the first base of the first chromosome to the last base of the last chromosome, which contains all the genes necessary for a human being.

Even though the DNA is very small, it's rather *long* in its scale. If the size of each nucleotide is around 3Å, then there are ~3 billion of them in total, which means if you stretch the DNA end-to-end, then it will span about 1 meter! So, every cell in your body has ~1 meter of DNA.[^2] Your body [has about 10 trillion cells](https://biology.stackexchange.com/q/3327/3446) and if they produce a DNA in every cycle, then your body alone [produces a light year of DNA](https://calculatedimages.blogspot.in/2015/04/light-years-of-dna.html) in your lifetime! More importantly, that's *almost* the same copy of DNA that began in the embryo.

*... which is a lot!*

We know that the DNA is the basis of all the wonderful mechanisms going on inside, so the first step is to *read* the DNA. That's what we call *sequencing*. The basic process goes something like this. A sample (say, from blood, or liver) goes through a complex preparation and gets loaded into the sequencing machine. Since there are only 4 bases and they bind only with their *mate* (A to T, or C to G, and vice versa), reading a single helix is enough (as we can always *deduce* that if one side was "ATTG...", then the other side would be "TAAC...").

Firstly, the double helix is [unwound](https://en.wikipedia.org/wiki/Nucleic_acid_thermodynamics#Denaturation) and individual strands are separately sequenced. As we're still in the molecular level, identifying the bases is *tough* (limited by the sensitivity of our instruments), and so, the sequence is amplified [by making huge copies](https://en.wikipedia.org/wiki/Polymerase_chain_reaction). All you need is a suitable environment and a bunch of [additional ATCGs](https://en.wikipedia.org/wiki/Primer_%28molecular_biology%29) to stick to the chain.

The interesting catch here is that the DNA sample won't be a single straight strand during the process. **It should be split into numerous sequences** (ranging from 100 to a few thousand bases) and laid into multiple container-like thingies (*lanes*), where they're *individually* amplified and *collectively* sequenced. Now, [special ATCGs](https://en.wikipedia.org/wiki/Dideoxynucleotide) are used to stop the amplification, which light up (by fluorescing a color for each base) as they attach themselves to a particular nucleotide. Sensitive photoelectric devices are then used to take snapshots of these lights in the lanes.

Each color indicates the presence of the corresponding nucleotide, and *voila!* DNA sequenced! Here's a [wonderful TED lesson](https://www.youtube.com/watch?v=MvuYATh7Y74) to visualize this.

## The Reference Genome

Remember that we had to slice the DNA into fragments? Once the sample has been *read*, the sequencing machine generates a [FASTQ](https://en.wikipedia.org/wiki/FASTQ_format) file[^3], which looks something like this,

    $ head -8 demo_blood.fastq
    @ERR009127.307 IL22_2005:8:1:5:1941 length=36
    GCAGACCCAGCGGGGCATGGGCGGACAGAGCCGCAC
    +
    <B?<@BB>>3BB>))<>43)94@=11=A?@=@B:/-
    @ERR009127.308 IL22_2005:8:1:5:944 length=36
    GGCGAACGCTTCGCTGGCCATTTAGGAGCTCTGCTC
    +
    B@AAAA=ABBB@B@BBB><;<A;<9<<;<;=<A;;3

Every 4 lines in this file is a "read". People are usually interested in the second line (the sequence fragment) and the fourth line (the ASCII-encoded quality of individual bases) in each read. The [quality score](https://en.wikipedia.org/wiki/Phred_quality_score) is much like the machine's confidence on a particular base.

For example, the last base in the first sequence "C" has a value 45 ('minus' in ASCII) whereas in the second sequence, it's 51 (number '3' in ASCII), which means we're relatively more confident about "C" in the second sequence. The scale is logarithmic and so, you can't expect an ASCII value more than 100 to show up all the time in reality.

Even though we've done so much to get this FASTQ file, it won't be of any use by itself, since it doesn't have the necessary information like where the particular sequence *was* in the sample, which means we've no idea what gene we're looking at, which isn't really useful.

So, we should reconstruct the DNA!

[Assembly of DNA](https://en.wikipedia.org/wiki/Sequence_assembly) is a *big deal*, because you have to figure out where a sequence belongs to. It's like shredding a novel into bits of paper (which contain nothing more than a few words) and recreating it back from the bits. This takes a *long* time! And, it's error-prone. Years have passed since [the first attempt](https://en.wikipedia.org/wiki/Human_Genome_Project), and yet, we don't know some parts of our own genome.

Building a DNA from the FASTQ file every time we read a sample would be rather *silly*. So, people have spent years to land on a basic template DNA. This is called the *reference genome*. It's incomplete, but it's accurate enough. Every species has its own reference genome. The human's is the largest - [a horrible 3GB file!](http://hgdownload.cse.ucsc.edu/downloads.html#human) It serves as a template because all humans share more than 99% of the genome. In other words, we differ only by a few million bases.

This is what the reference genome looks like...

{% highlight bash %}
$ head -5 ~/data/BWAIndex/hs37d5.fa
>chr1 dna:chromosome chromosome:GRCh37:1:1:249250621:1
NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN
NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN
NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN
NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN
{% endhighlight %}

It begins with "chr1" indicating the first chromosome (and its range - 1 to 249,250,621). Note that it begins with a lot of Ns. "N" indicates that we've no idea what base occupies that position (like I said, parts are still incomplete). Let's seek to a span with some data...

{% highlight bash %}
$ sed -n '200,204p' ~/data/BWAIndex/hs37d5.fa
TCAGCCTTTTCTTTGACCTCTTCTTTCTGTTCATGTGTATTTGCTGTCTCTTAGCCCAGA
CTTCCCGTGTCCTTTCCACCGGGCCTTTGAGAGGTCACAGGGTCTTGATGCTGTGGTCTT
CATCTGCAGGTGTCTGACTTCCAGCAACTGCTGGCCTGTGCCAGGGTGCAAGCTGAGCAC
TGGAGTGGAGTTTTCCTGTGGAGAGGAGCCATGCCTAGAGTGGGATGGGCCATTGTTCAT
CTTCTGGCCCCTGTTGTCTGCATGTAACTTAATACCACAACCAGGCATAGGGGAAAGATT
{% endhighlight %}

Throughout the file, this is what you find most of the time. So, it's accurate enough.

## Stitching the DNA...

Our *quest* is always to answer questions like, "Where the changes have happened?", "Why they happened there?", "Does/Doesn't it cause any disease/disorder?", etc. Now that we have a digital version of the reference genome, the next checkpoint is to [align](https://en.wikipedia.org/wiki/Sequence_alignment) the reads from our FASTQ file to the reference.

Before that, we also do a *quality check* on the FASTQ file (using a Rust-powered tool) to ensure that it's good. For instance, you can't have a lot of Ns in a particular sequence (we obviously don't want a lot of unknown bases). Also, if we assume that ATCGs are randomly distributed (which is often the case), then you can expect each base to occur 25% of the time on the average (if A is less than 5% and T is more than 40%, then we clearly don't want that). Then, we look for patterns of DNA which are *junk* (apparently, we won't get any useful info out of those). So, after these (and a lot more similar) quality checks, we trim the sequences, we filter some sequences, or even throw away the FASTQ file as needed.

If the FASTQ file is good, then we proceed for alignment. This is where things get a bit more interesting, because alignment is very much a string searching problem. If we're able to afford ~20 GB of RAM (which we *can*, given the low cost of cloud services like AWS or Google Cloud), then [suffix arrays](https://en.wikipedia.org/wiki/Suffix_array#Applications) could be used for the exact string matching problem. We build the suffix array for the reference genome (takes around 20-30 mins) and binary search for the occurrence of the FASTQ sequence. Once we find the position, we can find the chromosome it belongs to (if it's less than ~250 million, then it's definitely the first chromosome and so on).

But, that wouldn't be enough. All the sequencing machines make mistakes. What if we encounter a "N" along the way? What if the machine wasn't very confident about the base it just read? What if a mutation has happened at a particular position? What if a virus infection deleted a bunch of nucleotides consecutively from the sample genome?

Statistically, this happens 20-30% of the time in a FASTQ file. Apparently, it's an [approximate string matching](https://en.wikipedia.org/wiki/Approximate_string_matching) problem. So, how do we solve it?

Almost all the sequence aligners out there use the *magical* [Burrows-Wheeler transform](https://en.wikipedia.org/wiki/Burrows%E2%80%93Wheeler_transform). It's much like a wrapper over the suffix array (as a matter of fact, for longer strings, you need the suffix array to build the BWT - the usual sorting could take *days!*). Once the BWT is in place, we use the [FM-index](https://en.wikipedia.org/wiki/FM-index) to find things in O(1) time. For the human genome (without any optimization), you may need ~25 GB of RAM to do this, but it works pretty well, and it's worth every penny!

Moreover, since the reference genome doesn't change (until we get the next version), its BWT will be the same. It won't take more than 5-6 mins to build the FM-index from the BWT.

{% highlight bash %}
$ wc -c ref_genome gram_bwt
3137454505 ref_genome
3137454506 gram_bwt     # BWT needs one additional byte
6274909011 total
{% endhighlight %}

I'd [written an Rust library](https://github.com/wafflespeanut/nucleic-acid) for this. Let's take a look at its powers! I'll try the first sequence from the FASTQ file I just showed you above.

{% highlight bash %}
$ ./gram -c GCAGACCCAGCGGGGCATGGGCGGACAGAGCCGCAC
[["chr16",69776079,"NOB1"]]
{% endhighlight %}

And, we have a wonderful exact match! The sequence matches to position 69776079 in 16th chromosome, which is the range of the `NOB1` gene (don't worry about genes for now, we'll get into it in future posts). You can [check this with Wolfram Alpha](https://www.wolframalpha.com/input/?i=GCAGACCCAGCGGGGCATGGGCGGACAGAGCCGCAC) (you know, just in case, you don't believe me). The time taken for the Rust backend[^4] is ~50μs.

This means, we can align ~5000 sequences at a time (if they match perfectly) using a single CPU, and that's *lightning* fast!

## Magic behind the FM-index

Now, we come to the question of how it's so *magical*, and how we go about approximately matching a sequence. In order to generate the BWT, we append a null byte to the string, get all its rotations and sort it. For a string, say, "AACCG", we append the null-byte, say `$` to its end, and get the sorted rotations like this,

`$ A A C C G` <br/>
`A A C C G $` <br/>
`A C C G $ A` <br/>
`C C G $ A A` <br/>
`C G $ A A C` <br/>
`G $ A A C C` <br/>

Here, the last column `G$AACC` is the BWT of `AACCG$`. The suffix array of the same string is `[5,0,1,3,2,4]`. If we map the suffixes to their indices in the suffix array, we get this,

`$` <br/>
`A A C C G $` <br/>
`A C C G $` <br/>
`C C G $` <br/>
`C G $` <br/>
`G $` <br/>

And, we find a strong resemblance. They share the first column. The FM-index has the BWT and some additional information regarding the first column and the BWT itself. Using this, we can narrow down our search space. If your query starts with "C", then you'll know that "C" suffixes lie in the range `[3, 5]` in the suffix array. For the next base, you feed the previous range along with the base, and the index will return the next range.[^5] If it's a valid range, then the sequence exists, or if it's invalid, then we're *unlucky*.

*I won't go into the details here (I'm excited to talk about it, but this post is already big!).*

Let's try a sequence...

{% highlight bash %}
$ head -4 SRR413130.fastq
@SRR413130.1 A2097DABXX:4:1:2222:2104/1
GGACNGAGTTATCGAGGCACATACTCCACCACTGTCACAGGAAGAACCT
+
AA?@#BCCCCEGGGGGGEGGGGFGGGGGGGGGGGGGGGFGGGGGFGGGG

$ ./gram -c GGACNGAGTTATCGAGGCACATACTCCACCACTGTCACAGGAAGAACCT
[]
{% endhighlight %}

We don't have a match, because we have a "N" there (most often, that's the case). Let's try query'ing the sequence after the "N"...

{% highlight bash %}
$ ./gram -c GAGTTATCGAGGCACATACTCCACCACTGTCACAGGAAGAACCT
[["chr6",-161038213,"LPA"],
 ["chr6",-161043759,"LPA"],
 ["chr6",-161049303,"LPA"],
 ["chr6",-161060396,"LPA"],
 ["chr6",-161065943,"LPA"]]
{% endhighlight %}

Whoa! We have a lot of matches now. They all belong to the 6th chromosome and "LPA" gene, but the positions are negative. Why? That means we have matched its reverse complement `AGGTTCTTCCTGTGACAGTGGTGGAGTATGTGCCTCGATAACTC`. We've reversed query, complemented the bases (A for T, C for G, etc.), and then we get a match. Let's try it.

{% highlight bash %}
$ ./gram -c AGGTTCTTCCTGTGACAGTGGTGGAGTATGTGCCTCGATAACTC
[["chr6",161038213,"LPA"],
 ["chr6",161043759,"LPA"],
 ["chr6",161049303,"LPA"],
 ["chr6",161060396,"LPA"],
 ["chr6",161065943,"LPA"]]
{% endhighlight %}

There we go! The same matches (in forward direction). What this really means is that we've matched the other side of the double helix (which is fine too, since both are from the same DNA). Now, let's try the whole sequence, changing the "N" to "A"...

{% highlight bash %}
$ ./gram -c GGACAGAGTTATCGAGGCACATACTCCACCACTGTCACAGGAAGAACCT
[["chr6",-161038213,"LPA"],
 ["chr6",-161043759,"LPA"],
 ["chr6",-161049303,"LPA"],
 ["chr6",-161060396,"LPA"],
 ["chr6",-161065943,"LPA"]]
{% endhighlight %}

And, [we have the matches!](https://www.wolframalpha.com/input/?i=GGACAGAGTTATCGAGGCACATACTCCACCACTGTCACAGGAAGAACCT)

Let's have a look at the ranges while querying FM-index...

{% highlight bash %}
$ ./random unmatched_SRR413130.fastq
@SRR413130.12176383 A2097DABXX:4:62:1132:102596/1
AGAGCGGAGGCAGGAGTTGGGCCCCAATTTGCTTCACGTNAAATTTATG
+
DDDDDDDCDDD:CD=;2<<?CBDDCBBBBBBBBBB;>:7#08665BBBB

$ ./gram -c AGAGCGGAGGCAGGAGTTGGGCCCCAATTTGCTTCACGTNAAATTTATG
[]
{% endhighlight %}

So, we have another sequence which doesn't match. I've modified the wrapper to show us the range output for each base. Let's feed the bases one by one...

{% highlight bash %}
$ ./gram -c e:::G
(G, 1449387873, 2042847480)

$ ./gram -c e:1449387873:2042847480:T
(T, 2642364268, 2853319690)
{% endhighlight %}

We begin from the last base[^6] "G", and we get a range. Then, we feed the second last base "T" with G's range, and I get a new range. It's valid. Now, let's try feeding the whole thing.

{% highlight bash %}
$ ./gram -c AGAGCGGAGGCAGGAGTTGGGCCCCAATTTGCTTCACGTNAAATTTATG --debug
(G, 1449387873, 2042847480)
(GT, 2642364268, 2853319690)
(GTA, 730337789, 783909233)
(GTAT, 2436460939, 2448269684)
(GTATT, 2902060112, 2905545457)
(GTATTT, 3044849837, 3046261437)
(GTATTTA, 832453834, 832844405)
(GTATTTAA, 276956227, 277060107)
(GTATTTAAA, 109010853, 109046976)
(GTATTTAAAN, 2042847950, 2042847950)
[]
{% endhighlight %}

Apparently, we're losing wonderful matches just because of a few accidental/incidental substitutions, insertions or deletions (and like I said, this happens 20-30% of the time). In our case, it has stopped at "N", because `GTATTTAAAN` has returned an invalid range. This is how we approach the fuzzy string matching. All we have to do is once (and whenever) we encounter an invalid range, we try querying a new base with the previous range, and *backtrack* from there. Since there are only 4 possible bases (and since the FM-index is *fast*), depending on our algorithm, we won't be risking a lot of computing time unnecessarily.

As for this query, changing the "N" to "C" will [return a match](https://www.wolframalpha.com/input/?i=AGAGCGGAGGCAGGAGTTGGGCCCCAATTTGCTTCACGTCAAATTTATG). This is a *mismatch*. Insertions and deletions can happen too, and it's up to the aligner to decide which alignment to take/leave.

However, limiting the depth of backtracking is up to our resources i.e., how much edit distance we're willing to allow (the more we allow, the more we're prone to bad sequences and end up spending more computing time, and disallowing them entirely results in leaving out the most important sequences).

Now that we've aligned the FASTQ file to the reference genome, the next checkpoint is to infer the alignments - like how many gaps/insertions have occurred, whether a particular substitution is a mutation or whether it's a machine error, or whether the alignment itself is wrong, etc., but that's for the next post.

*Auf Wiedersehen...*

[^1]: Then, there's the X and Y chromosomes which determine sex.

[^2]: Actually, it's every cell with a nucleus. That's because not all cells have a nucleus - even though the red blood cells, the cells in hair, skin, nail, etc. start with a nucleus (with a DNA), they destroy their nucleus as they mature.

[^3]: I'm choosing the simpler case here, because nowadays, a machine generates two FASTQ files - one belonging to each strand of the DNA (one will be in the forward direction, and the other will be in the reverse direction, as if their tails are tied up). When we analyze the files, we get reads from the *paired* files at the same time.

[^4]: I have a simple TCP server that listens to a particular port for sequence requests (or job requests for running an entire file!). That's just a client making a request and printing the response. Rust's [concurrency primitives](https://doc.rust-lang.org/std/sync/) are rather *charming* to work with, and it's always pushing me to write parallel code.

[^5]: The range is very useful by itself for counting the occurrences of substrings. The difference between them indicates the number of occurrences.

[^6]: That's because I have a forward BWT, and it's got to do with how the FM-index works. If I had a BWT of the reverse genome, then I'll be querying from the start.
