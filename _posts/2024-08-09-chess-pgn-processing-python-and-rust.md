---
layout: post
title: "Chess PGN Processing: Python and Rust"
date: 2024-08-09 17:00:00 +03:00
categories: DATA-ENGINEERING RUST PYTHON
---

# Parsing millions (or billions) of PGN games

I love chess. I've been playing since I was a child, albeit with long pauses in between . Recently, I came across this [huge database of games on lichess.com](https://database.lichess.org/) which contains literally **billions** of [.pgn](https://en.wikipedia.org/wiki/Portable_Game_Notation) files, which are compressed with [zstd](https://en.wikipedia.org/wiki/Zstd). I wanted to do the following:

1. Download the files for each year
2. Preprocess them, meaning: extract the headers, the moves, correct faulty data, and store them in a more suitable format (CSV? JSONL?)
3. Load them up in a database to perform some statistical analysis and extract patterns

All these (barring the statistical analysis part which involves Data Science) are basically the bread and butter of Data Engineering. I won't get into too many technical details about the type of processing (I will probably upload the code on GitHub soon).

This becomes more interesting when you notice that there is data for over 5.8 **billion** games. They probably amass over *2TBs* worth of data, compressed!

## Python stuff

I have been using Python for over 7 years, and I have always liked its simplicity and the sheer number of libraries available for any task. Writing a script for [async](https://en.wikipedia.org/wiki/Async/await) downloading of the files was a piece of cake.  

For the processing of the files, I went with the following approach:

1. Read each compressed file line by line, up until a whole game has been read
2. Extract the headers and the moves, and correct faulty data
3. Write each game to its own line in a JSONL file

This is easily parallelizable across multiple files for a given year (or more).  This approach saves both memory and disk space, because at most one game will be stored in memory at a time, per core. Extracting the files beforehand would result to a disk usage increase by over 8 times the compressed data. 

I used the following core libraries: [python-chess](https://github.com/niklasf/python-chess),[orjson](https://github.com/ijl/orjson), [zstandard](https://pypi.org/project/zstandard/).

## My problem with Python

Basically... it's **slow**. Processing around **3.3M** games takes over **10** minutes, with multiprocessing distributing some of this load!  

Now, I'm pretty confident that I can write performant Python code. The libraries I was using are also pretty staple for the given task, and **orjson** in particular is extremely performant. While I did not dive into in-depth profiling of which parts of the code are the bottleneck, it's probably a combination of a lot of factors: the PGN parser itself, [I/O](https://rabexc.org/posts/io-performance-in-python), [notoriously slow for loops](https://stackoverflow.com/questions/8097408/why-python-is-so-slow-for-a-simple-for-loop) and possibly multiple dict operations per line. The PGN parser which does most of the job is in pure Python.  

![slowest things on earth](/images/slowest_things.jpg)

## Switching to Rust

TL;DR: Rust finished in ~**8.8 seconds** instead (to be continued)

![rust vs pythno pgn parsing](/images/rust_python_pgn.png)