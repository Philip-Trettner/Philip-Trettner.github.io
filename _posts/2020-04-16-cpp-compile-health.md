---
layout: post
current: post
cover:  assets/images/compilehealth.png
navigation: True
title: "C++ Compile Health Watchdog"
date: 2020-04-16 0:00:00
tags: [c++]
class: post-template
subclass: 'post'
author: philip
excerpt: 'Who f*cked up my build times??'
---

So anyways, why is my build so slow?

**TL;DR:** I made a [website](/projects/compile-health) to check the build impact of standard or external libraries. This post describes the motivation and methodology behind it.

**NOTE:** This is _NOT_ a public shaming of C++ libraries. I wanted this website to make sensible decisions and keep myself accountable. 
The [latest isocpp survey results](https://isocpp.org/blog/2020/04/results-summary-2020-global-developer-survey-lite) indicate that I'm not the only one caring about compile times.


## Easter Project 2020

My C++ life changed dramatically ever since I discovered [zapcc](https://github.com/yrnkrn/zapcc) and got my incremental build times to below 1 second.
I've developed a kind of fetish for fast builds and sometimes rewrote or wrapped several external libraries purely to get the compile times under control.

As the old saying goes, there are really only three ways to optimize anything: _measure_, _measure_, and _measure_.

Thus, over the Great Easter Lockdown 2020 I set out and created the [C++ Compile Health Watchdog](/projects/compile-health) website to help us make informed decisions about which headers to include publicly, which to wrap, and maybe tip the scale in the choice of a library.
The code for creating the data that feeds this table can be found in [my github repo](https://github.com/Philip-Trettner/cpp-compile-overhead), contributions are more than welcome!

The rest of this post explains my methodology in detail:


## The Impact of a Header

Each row in the table is a either a single header or source file, compiled with a certain configuration (compiler, build type, C++ version).
A [python script](https://github.com/Philip-Trettner/cpp-compile-overhead/blob/master/scripts/analyze-file.py) is responsible for running various tests and extracting metrics we care about.

I'm using the compiler (clang and gcc for now) in two modes:

* Compile: `-c main.cc -o main.o` (+ extra args)
* Preprocess: `-E main.cc -o main.o` (+ extra args)

The extra args depend on the configuration, for example Clang 8 (`/usr/bin/clang++-8`) for C++17 in RelWithDebInfo results in `-std=c++17 -O2 -g -DNDEBUG -march=skylake`. I've added `-march` mainly for consistency.
The `main.cc` we're compiling looks like this:

```cpp
#include <file-to-test>
int main() { return 0; }
```

I'm also testing a `baseline.cc` without the include.

First, we're counting the _Lines of Code_ in the preprocess-only file to get two metrics: `line_count_raw` (all lines) and `line_count` (only those lines containing `[a-zA-Z0-9_]`).

The file size of the compiled `main.o` gives us `object_size` (with a `_base` version for the baseline).
We continue by invoking `nm -a -S main.o` to collect information about the contained symbols (number and size of undefined, data, code, weak, and debug symbols as well as the size of the _names_ of the symbols).
Then we invoke `strings main.o` to get the number and size of all contained strings, follow by `size -B main.o` to get the size of the `.text`, `.data`, and `.bss` sections.
Big binaries slow down the build process by increasing the load on the linker.
"Symbol explosions" can happen relatively easy by accident.

Finally, we measure how long the compile and the preprocessing commands take for `main.cc` and `baseline.cc`.
I've experimented with a few different approaches and my current local optimum for accuracy, reproducibility, and sane data generation time is:
Execute each command 10 times and take the lowest elapsed wallclock time.
Compiling is more or less deterministic so higher times are likely due to OS interruptions and other processes.
The whole benchmarking is done single-threadedly as multi-core compilation will likely skew the results.
I'm currently testing 36 configurations per file and some files can take pretty long.
Relative accuracy is probably already good enough for slower files so I only execute the commands 3 times if the file takes more than 500ms to compile.
For reference: 36 configurations, compiled 10 times each for a file that takes 1s is at least 6 minutes benchmarking for a single file!


### Impact Score

On the [website](/projects/compile-health), all the measured metrics are available by hovering over the table cells.
However, I also wanted a single visible number that summarizes the "impact" of including a given header.
I call this the _Impact Score_ `S` and it is, of course, highly opinionated.
It is based on the increase in compile time `t` and the increase in binary size `s`.

\\[ S_{\text{time}} = 100 \cdot \left( 1 - \frac{200\ \text{ms}}{200\ \text{ms} + t} \right) \\]
\\[ S_{\text{binary}} = 100 \cdot \left( 1 - \frac{5\ \text{MB}}{5\ \text{MB} + s} \right) \\]
\\[ S = \min(100, S_{\text{time}} + S_{\text{binary}}) \\]

Each score is designed to be between 0 (zero compile time or binary increase) and 100 (compile time or binary increase goes to infinity).
50% impact is reached at 200 ms or 5 MB.


## Roadmap

This project is not finished and I might broaden its scope considerably.

There are a few features that I want to implement for the website, like filtering configurations and uploading your own data.
It would be nice if the data generator can directly use a JSON Compilation Database (e.g. via `CMAKE_EXPORT_COMPILE_COMMANDS`) to analyze your own projects without hassle.
And of course I need to add Visual Studio to the mix.

There are also things [you can help me with](https://github.com/Philip-Trettner/cpp-compile-overhead#contributing).
Let me know which libraries I should add to the table!
Is there additional data you would like to see for each compilation?
