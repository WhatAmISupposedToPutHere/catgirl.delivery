---
layout: post
title: Sometimes It Is Correct to Blame the Compiler.
---

I tend to make bad decisions. I haven't yet figured out whether running Gentoo
on my primary machine is one of them, but it will be the one at the start of
this story. While I was running a system update one day, the Firefox build
failed with an error. From taking a cursory look at the logs it was a problem
in some javascript file that runs during build using Node.js, so of course I did
the right thing... by which I mean I downgraded node, successfully compiled
Firefox, and hoped that someone else will hit this bug and figure out the issue
and fix it.

# Am I a "someone else"?

A few Firefox versions later the issue was still not fixed, and doing the
downgrade dance each time was getting annoying, so I decided to take a look at
it again. The specific issue was that a checked-in version of Babel was trying
to write a read-only property on a `WeakMap` and getting thrown a `TypeError`
for its troubles. Naively thinking "It is probably a missing check in a
polyfill," I've fed the file through a pretty printer, ran it again to get a
precise backtrace, and looked at that line

```
try {
   e[$t] = void 0; // Traceback points to this line
   var n = !0;
} catch (e) {
}
```

...

Had node forgotten how to catch an exception?

# Do not blame the compiler, blame your compiler's compiler.

While I had experienced several ICEs in ancient clang versions, this was the
first time a "compiler" was producing results that were outright wrong. But
even in such obvious code, I had to check whether javascript suddenly added
uncatchable exceptions, or whether I was wrong about how `try/catch` works. But
no, this really was a node bug. It was
[reported](https://github.com/nodejs/node/issues/47522) on node's issue tracker,
promptly redirected to [v8's](https://issues.chromium.org/issues/42203868),
where someone noticed that Fedora knew that this issue happens when node gets
compiled with `-O2` and disappears when using `-O1` or `-O3`. They then
[decided](https://src.fedoraproject.org/rpms/nodejs20/blob/908db65fbdfcaa64b4a73d222840bb9768a05cbd/f/nodejs20.spec#_535)
to go with `-O3`.

At this point I thought I was almost done, after all there is a fix, and if
it is good enough for Fedora, it is good enough for Gentoo, so I made a ebuild
patch that replaces `-O2` with `-O3` and sent it as a PR upstream.
All was left was to wait for review.

# The compiling begins.

While a typical Gentoo user will happily run their system with
`-05 -ffunsafe-math -fplacebo` and probably a few other insane flags, the
maintainers rejected my PR and suggested to find the specific optimization
that causes the error. So why not, let's do exactly that.

Trying to figure out which of the flags included in `-O3` fixed the problem
was a bad decision, in that it was a complete waste of time. None of the flags
fixed the issue when included one at a time, and I was not going to compile
anything `2^n` times, no matter the value of `n`. 

What was a success was expanding `-O2` into it's constituent flags and then
binary searching for the offending flag by removing half of them at a time.
After 6 or so rebuilds, I got my culprit `-ftree-slp-vectorize`, which I then
confirmed both by adding it alone to `-O1` and disabling from `-O2`. The results
were as expected.

# Needles, haystacks... just use a magnet.

Nodejs is an enormous project, and a bug report that says that there is a
miscompile somewhere inside it is not exactly helpful, especially if I do not
know if it is a miscompile in the first place and not some UB-triggering code.
Thankfully a binary search helps us again, with a simple `g++` wrapper, one can
enable or disable the specific optimization flag for half of the files and then
see if that results in the bug appearing. Just 10-ish rebuilds later, I've got
the file, `deps/v8/src/objects/objects.cc`.

This was better, but it is still a massive file containing a lot of logic.
A possible approach would have been to binary search within the file,
compiling half the functions with flags enabled and another half without.
But this time i did nothing of that sort, and instead looked at the functions
that could return a `TypeError`, picked one that looked the most suspicious,
added `__attribute__((optimize("no-tree-vectorize")))` to it, and got it right
on the first try.

# Reduction in size.

Even after removing all other code but the offending function, the file was
15 MB with all the includes, and even then, I am no compiler expert and had no
idea what specific part of the function was causing the incorrect return.
But thankfully there is a tool for that: as long as you can write a test that
checks whether your c program is "interesting"
[cvise](https://github.com/marxin/cvise/) can reduce it's size and complexity
while maintaining the "interestingness" property. In my case, I noticed that
after compiling the function to assembly, the incorrect version sets the return
register to 1 in both branches, while the correct version sets it to 1 in one
and to 0 in the other. Some grep crimes later, the test was ready.

Of course it was not that easy, cvise was unable to process said 15MB file in
the paltry 32GB of ram installed in my laptop, but after manually removing large
chunks of it, it managed to create a tiny reproducer. Now I had a small test
case and a specific flag that triggers it, it was finally time to find the bug.

# What did it cost?

It would probably surprise nobody at this point that I was doing a yet another
binary search. This time it was a git bisect of gcc, and 12-ish more rebuilds,
i've arrived at the commit that caused it all... Said commit was changing the
arm64 instruction cost model.

Obviously this was not the commit that introduced the issue, it was already
there, and it only exposed by it. After some consultation in `#gentoo-toolchain`
and debugging by `Arsen` we have filed a
[gcc bug](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=116057).

# Not a hero of this story.

At this point you'd probably expect me to write that I found a way to keep
bisecting for the actual commit that introduced it, and then found and fixed
it, but nothing of that sort happened. After all, I've never even seen gcc's
code and while I did keep bisecting, `pinskia` and `rguenth` found the issue
with the optimization pass and fixed it. It will ship in gcc 15.0.

# Conclusion, eh?

1. Binary search is the most important algorithm ever invented.
1. Compilers are not magic, they are big, but even someone like me, who never
looked at the code before can figure out how some of it works.
1. Deciding when you have enough information to file a bug is a bit of an art,
while someone else knows the code to any specific project better than you, and
will be faster at root causing and fixing it, you should try to give them as
much info as you can collect on your own.
1. I've given you the tools and the playbook on how to find the root causes
of a possible compiler bug. So next time you want to blame the compiler, you
can use it to conclusively prove that you are right, and which specific part of
the compiler is wrong. After all, it would only take 3-ish days of just
compiling various versions with a rather powerful laptop.

