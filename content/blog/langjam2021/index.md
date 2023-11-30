+++
title = "Making a Programming Language in 48h"
date = 2021-08-26
+++

Last weekend, I built a new programming language completely from scratch in 2 days.
In fact, [quite a lot of people did](https://github.com/langjam/jam0001/pulls?q=is%3Apr+sort%3Aupdated-desc+is%3Amerged), all as part of the first ever langjam.

If you're familiar with game jams, the langjam was the exact same principle applied to programming language development:
On Friday evening (my time), the jam started with the annoucement of a theme.
From then on, participants had 48 hours to invent a programming language around that theme, build some compiler, interpreter, or other means to run programs in their language and put together some usage instructions and examples.

In this post, I want to present the language I built, reflect on the jam, and try to work out some things to learn from my experience that generalize to language development outside of two-day programming marathons.

## This Post is
 - A recap of my experience participating in langjam 2021.
 - A presentation of the language I created during the jam.
 - About some things that worked well, and some things that didn't.
 - Full of compromises.
 - Hopefully, motivating for all you folks who are working on their own (or any other) language. 

## This Post is not
 - About writing production language implementations. When I said I made a lot of compromises that was not a joke - no matter the 48 hour language, it's probably not Rust.
 - A collection of examples for good code. Same reason as above, really.
 - An overview on the languages people created during the jam. They are many and I haven't even tried half of them, but among the ones I've seen several were very cool. You can check 'em out [in the jam repository](https://github.com/langjam/jam0001).
 - Written to get votes. I was in it for the fun and not to win, and I god damn had fun. You can of course vote on the `Team:` issues [here](https://github.com/langjam/jam0001/issues?q=is%3Aissue+is%3Aopen+sort%3Aupdated-desc), but please only do so if you also look at and rate other people's entries!

---
## Note
Since this post is about something I did during an event and not for this blog, there is no code for this post in the blog repository.
You can find the full implementation of my jam entry in my fork of the jam repo [here](https://github.com/domenicquirl/jam0001/tree/main/%E0%B6%9E), but I'll also link that again below where relevant.

---

## The Jam
Since the langjam was the first of its kind, I didn't really know what to expect going into it.
Given that it was organized by Jonathan Turner, who was very involved with TypeScript, has [a YouTube channel](https://www.youtube.com/user/giard321) discussing programming languages including Rust (and more), and is behind [Nushell](https://www.nushell.sh/), my hopes were up.

_The_ question before the jam was which theme they would choose.
A jam theme needs to be general enough to allow a wide range of interpretations from participants.
Game jams often go somewhat abstract with this, having themes like 'Joined Together', 'Out of Control', or 'GENTRE, but you can't MECHANIC'[^gmtk-jams]<span id="fn-gmtk-jams"></span>.
Arguably, finding a theme is harder if you're specific to programming languages (but feel free to disagree).

When the jam came around, it's theme was **First-Class Comments**.
As soon as the theme was announced, we got together and brainstormed where we wanted to go with this.
Both the concept of [first-class citizens](https://en.m.wikipedia.org/wiki/First-class_citizen) and, well, comments, have pre-existing meaning in the programming language world, so having comments be values in our language was one of the earlier things that came up (and did get into the language, but not as its main focus).
Many participants from our and other teams also thought along the lines of [literate programming](https://en.m.wikipedia.org/wiki/Literate_programming), where you describe what you want your program to do in natural language in addition to partial source code, and a corresponding program is created that does what you told it to.
Some other ideas that I've collected from our Discord chat retroactively were (in no particular order and with no judgement on quality):
 - The compiler can read your code and react to it, making _comments_.
 - Have some rather basic instructions that are written like sentences. E.g., `I've heard about x. x is kinda 4.` would be one instruction that declares `x` and one that sets it to `4`.
 - Any comment you write must only be snobby, posh sentences. Slang or emojis in comments are compiler errors, so are grammatical errors.
 - Use comments to represent/initialize data: 
   ```
   /* 
     class: User
     name:  Max

     Max has been disappointing today.
    */
   ```
   would represent an object of class `User` where the `name` field has the value `Max`. The bottom line would be a per-object actual comment.
 - Have `// TODO` comments be a warning and `// FIXME` comments be an error.
 - Doc tests, like Rust has too.

## My Language

First off, it's time that I explain the 'I' vs. 'we' confusion that is going on in this post currently.
We started out brainstorming as a team of 4 good-hearted `#lang-dev`vers from [the Rust community Discord](https://discord.com/invite/rust-lang-community)'s language development channel.
Over the course of the weekend, some stuff came up for everyone but me, so in the end we ended up with a very me-centric language and commit history.
If you guys be reading this: I love y'all. Life happens. 

So, what kind of language _did_ I make with all of this randomly newfound authority (and in about ¼ of the original scope)?
For me, the thing that immediately came to mind when I read 'first-class comments' was meta-programming.
I don't know why, it was just the context in which I could see comments-as-values being the most useful.
Cause sure, sticking a comment in a variable like `let x = /* bar */;` is fun and all, but what does that let you do?
If you have access to a program's syntax tree however, and you have a parser that preserves comments, and then you can **inspect and manipulate** the syntax tree... well, then it's not the compiler that decides what effects comments in your program will have - it's **You** who decides[^syntax-trees]<span id="fn-syntax-trees"></span>!

Enter _Suslang_.
(Yes, the language is called Suslang and our team name was ඞ. 
If you're in the community Discord and you know Borger, you can go tell him if you like it. 
If you aren't, this is absolutely a great and important reason to [join](https://discord.com/invite/rust-lang-community) and you should definitely do it!)

One of the main concepts in Suslang is that you don't run your program, but you run a build script that then runs (or maybe doesn't run) your program.
The build script is itself a Suslang program and has access to a `parse` function which runs our parser on some given input and returns an AST, as well as an `eval` function which takes an AST and executes the program represented by it.
In the simplest case, a build script just parses some program file and then runs it using `eval`, maybe checking for errors in between ([example](https://github.com/domenicquirl/jam0001/blob/main/%E0%B6%9E/examples/just_build.sus)).

But you can do more, a lot more!
You can have a build script that [ignores your program, counts to 10 to pretend it's doing work, and then just prints `Hello World`](https://github.com/domenicquirl/jam0001/blob/main/%E0%B6%9E/examples/dont_build.sus).
Or you can have a build script that [makes up its own input and parses and runs that instead](https://github.com/domenicquirl/jam0001/blob/main/%E0%B6%9E/examples/build_parse_string.sus).
(Annoyingly, this means that the lang name _does_ kinda fit, since you can never be quite sure what version of your program will be running when executing through a build script.
Never trust a program that you haven't meddled with yourself.)

And you can do meta-programming.
You can look at the syntax tree of a program and modify it in any way you like.
[Here](https://github.com/domenicquirl/jam0001/blob/main/%E0%B6%9E/examples/build_modify_let.sus) is a build script that replaces the `value` in every `let var = value;` statement with the expression `value + 1` in the AST before running the modified program.
Alternatively, you can just [build your own program entirely from AST nodes](https://github.com/domenicquirl/jam0001/blob/main/%E0%B6%9E/examples/build_your_own_program.sus) and evaluate that.

### First-Class Comments
Let's bring this back to the jam's theme, 'first-class comments'.
I said before that Suslang does support comments-as-values, which are of course preserved in the AST.
In fact, [both `let x = /* Hello, world! */;` and `let y = // Hello, @todo!;` are supported](https://github.com/domenicquirl/jam0001/blob/main/%E0%B6%9E/examples/comments.sus), at the minor expense of banning `;`s from line comments.
But even 'regular' comments that just appear in the source file are parsed and preserved in the syntax tree, such that they can be inspected in the build script.

What you do with them is up to your imagination.
Consider a Suslang program which contains the following function
```rust
// @trace
fn has_comments(x, b) {
    let v = 0;
    if b {
        v = 1;
    }
    let y = x + v; // @todo: add more code
    return y;
}
```
that is later called somewhere else like so (this example is from [our `comments.sus` example](https://github.com/domenicquirl/jam0001/blob/main/%E0%B6%9E/examples/comments.sus))
```rust
let sum = 0;
for x in range(3) {
    // @fixme: maybe this should be `false`?
    sum = sum + has_comments(x, true);
}
print(sum);
```
I have written [a build script that acts on the `@tag`s](https://github.com/domenicquirl/jam0001/blob/main/%E0%B6%9E/examples/build_check_comments.sus) present in the comments in the above program.
After one of [our early ideas](#the-jam), it will print out warnings for `@todo`s and `@fixme`s.
It will also instrument functions with the `@trace` annotation with print statements that tell you when and with which values the annotated function is called.
If you run this build script on the above example, you will get the output
```
Parsing Build File
Executing Build File
Parsing Source File '.\examples\comments.sus'
Checking comments of '.\examples\comments.sus'
Warning: unresolved `@todo:`: ' @todo: add more code'
Warning: unresolved `@fixme:`: 
    ' @fixme: maybe this should be `false`?'
Executing Source File '.\examples\comments.sus'
Calling fn `has_comments` with (x = 0) (b = true)
Calling fn `has_comments` with (x = 1) (b = true)
Calling fn `has_comments` with (x = 2) (b = true)
6
```
The required print statements are inserted directly into the program AST by the build script.
It remembers when it has seen a `@trace` annotation, and when it reaches the annotated function it constructs a call to `print` by inspecting the function's name and parameters.
I know it was my own idea, but I think that's pretty neat, and a great way to have comments become even more 'first-class' inside the build script: they are AST nodes just like all the other program components.

If you want to try it out for yourself, [here's the link again](https://github.com/domenicquirl/jam0001/tree/main/%E0%B6%9E).
In case you find any remaining bugs, I'm not taking any responsibility, even though I probably caused them.
Have fun!

## Things That Worked

**I spent the entire first day without a parser**, and therefore also without any syntax.
<br>
I knew I wanted to do meta-programming, and the most important thing I needed to do to get that to work was encoding the syntax tree in Suslang itself and exposing it to the user.
So I started with the AST evaluator and went backwards from there.

Starting with the later parts of the implementation I feel like did a lot to get me towards a working version of what I wanted to achieve.
It kept me focused on the things that needed to work, leaving more flexible aspects like syntax aside which have a tendency to get me stuck bikeshedding.
It helped me get to my first meta-program - the [one modifying your `let` statements](https://github.com/domenicquirl/jam0001/blob/main/%E0%B6%9E/examples/build_modify_let.sus) - quite quickly, within the first day (even though I had to manually define it in the test, because there was no syntax to describe it).
This in turn was a great boost to my motivation as a sign that I was on the right track.

**Tree-Walking.**
<br>
Because I had enough to do with the whole meta-programming thing (and time was of the essence after my team size reduced by 75%), I didn't even consider more fancy evaluation schemes.
Sure, Suslang is probably not the most performant language in the world (printing the 25th Fibonacci number with a naïve, recursive definition takes roughly 3.9 seconds for me, compared to less than 10ms for the equivalent Rust code), but it is a _working_ language, which was arguably more important.
Also, as the meta-program creates and works on ASTs, working with trees seemed like a natural choice for the jam either way.

**I hand-wrote my parser.**
<br>
And I don't think that this made me significantly slower than if I had used any parser generator or combinator library.
I did use [the `logos` crate](https://crates.io/crates/logos) for my lexer, but the parser is all hand-written recursive-descent.

Now, I do have a good bit of experience with hand-written parsers.
The [only other article on this blog to date](https://domenicquirl.github.io/blog/parsing-basics/) is on parsing, I've been writing a moderately sophisticated parser in my main time and [recently practiced](https://github.com/rust-lang/polonius/pull/173) by speedrunning a small lexer + parser implementation for the experimental _Polonius_ borrow checker during one of their sprints.
Meanwhile, I have comparatively little experiences with generators like `lalrpop` or `pest`, so your mileage may very well vary.
But at least for me, I didn't feel like it slowed me down hand-writing everything.
I was able to get the entire parser done in the evening/night of the first day, including a first translation of the Rust AST to Suslang AST objects.

I also just like the amount of control I have in making the parser do exactly what I want if I hand-write it.
For example, I'm sure it would have taken me ages to figure out how to disallow `Class { field: value}` object literals in specifically `if` condition and `for` loop target positions with a generator (it's probably easy, I just don't know).
These little buggers kept trying to mess with `if b {` and `for stmt in ast.body {` statements, because of the opening `{` of the following block.
I just slapped some state onto my parser to fix that, but I'm not sure I would have been able to return to my main TODO-list as soon trying to re-structure my expression hierarchy in a non-precedence parser (I used [Pratt Parsing](https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html), which I also discuss [in my post on parsing basics](https://domenicquirl.github.io/blog/parsing-basics/#binary-operators)).


## Things That Didn't Worked

**What's a Pointer?**
<br>
When I started writing the evaluator, I knew I was on limited time.
Thinking myself smart, I didn't spend any time on engineering clever data structures or trying to be efficient, and instead focused on getting my tests working.
In Rust, one of the things this often means is ignoring references and lifetimes and all that and instead slap lots of `.clone()` anywhere you get a compiler error without it.

This had one major issue:
I wanted the meta-program to be able to not only read an AST, but also modify it - that's half the fun after all (the better half)!
Now, if the evaluator clones the AST nodes wherever it needs to pass a value, the chance that the AST object you are modifying in the build script is actually part of the original AST that is later evaluated is _pretty low_.
And you'll get a whole lot of 'hey, why's the original program running?'

In hindsight, this problem is so obvious to me that I wonder how I have ever not seen it.
Of course, in hindsight, I have also spent several hours fixing it, getting it stuck in my head.
To allow working on the original AST, some notion of access by reference or pointer was required.
Shoehorning those into the already working evaluator gave me more than a few grey hairs, as I scrambled to find (and understand) the correct combination of when to reference what, how to iterate over fields that are pointers, and what to do with awesome types like `*ref **ASTFnDef`.

**Memory Issues**
<br>
I have a confession to make.
The Suslang implementation uses `unsafe`, and I'm not sorry about it.
Even before dealing with language-level references/pointers (see above), I already didn't want to deal with all of my `Int`s and `Strings` and `Object`s having to carry around a lifetime only because _sometimes_ I want to access `a.b.c` and then assign to it, which needs to modify the original field (see how all my problems involve modifying things that would be way easier to just `.clone()`? Where have I gone wrong...).
To avoid that, I used _raw pointers._

Surprisingly, ~~carelessly~~ using raw pointers is not actually what caused me issues.
I even managed to implement all of the evaluator with them without majorly screwing up (as far as I know).
The problem came on the final day, as I thought I was already done with the implementation and was writing documentation and examples.
Suddenly, my programs started to segfault.

Because I wanted to save time (ominous foreshadowing: that maybe didn't work out), I had used [the `quickscope` crate](https://crates.io/crates/quickscope/) and its `ScopeMap` to handle nested scopes and their variables.
This will not be a dig on `quickscope`.
I am sure their implementation does a great job in many programs, such as it did in my evaluator until things started to not work.
Just in my head, because in Suslang you cannot drop variables or otherwise end their lifetime, the memory for a variable's value was a fixed location until the variable goes out of scope and you can't access it anymore anyways.
You can of course overwrite variable values, but that only changes the value stored at the corresponding location, not the location itself.

Reality had other plans with me.
I skimmed through `quickscope` and they use `SmallVec`s internally, which, in addition to already re-allocating and moving their memory when they need to grow like a regular `Vec`, also switch from stack memory to heap memory if their length exceeds some threshold.
Of course, when either of these events happened, all of my values would move to a new memory location.
And when the program processed by a build script was too big, the location of AST elements could change while you start iterating over them because you create a new variable for the loop element.

The issue was easily fixed by replacing the uses of `quickscope` with my own `ScopeMap` implementation which guarantees to not move variables within the same scope (I wrote a hilariously inefficient version using linked lists. Now that I'm writing this, I probably should've replaced the `Vec` for nesting scopes with such a list too... oh well...).
Unfortunately, I only had time to fix this one after submission (so the fix for it is not yet in the jam repo, be aware if you're trying Suslang from there instead of through my links).

## Wise Words

A jam is the perfect environment to force you to restrict yourself, to be creative despite and because of your limitations, and to just get something _done_.
Your code doesn't have to be perfect.
If I were trying to make Suslang a real language, I would refactor the evaluator _immediately_.
Like, right now, even though it is getting late and I'm already a bit tired.
But who could think of _sleeping_ while there exists this _monstrosity_ in my _production language???_

The thing is, I only know this because I have already written the evaluator once and have built enough remaining language around it to see my meta-programming goals come closer.
In fact, I have already re-written several parts of it during the jam (mostly because they weren't working, but hey).
Sometimes, you just need to write some code to do the thing you think you want to do, to see if it's actually how you want your lang to be.
Sometimes it is due time for a re-write, but maybe sometimes you can build another feature on top of even the most cursed part of your compiler and get a better feel for where you want to go and what you need to do.

Embrace the re-writes, but also embrace the cursed.

---
[^gmtk-jams]: All of these themes are taken from recent [GMTK game jams](https://itch.io/jam/gmtk-2021). <a href="#fn-gmtk-jams" class="footnote-backref" role="doc-backlink">↩︎</a>

[^syntax-trees]: I'm assuming you know what a syntax tree is. If you don't, and if the abbreviation AST confuses you, maybe read [the introduction to my post on parsing basics](http://domenicquirl.github.io/blog/parsing-basics/#a-high-level-view) or [the Wikipedia article on abstract syntax trees](https://en.m.wikipedia.org/wiki/Abstract_syntax_tree) before you go on. <a href="#fn-syntax-trees" class="footnote-backref" role="doc-backlink">↩︎</a>
