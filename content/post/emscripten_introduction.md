+++
date = "2016-06-19T00:00:06+01:00"
draft = false
title = "Emscripten and SDL2 Tutorial Part 2: Introduction to LLVM, Emscripten and asm.js"
series = "Emscripten and SDL 2 Tutorial"
image = "emscripten_tutorial.png"
+++

JavaScript is a remarkable language, cobbled together in 10 days in 1995 and
intended to be a forgiving language to enable simple pieces of interactivity to
be incorporated into web pages, it now finds itself implemented in every
graphical web-browser and the natural choice for writing client-side web-apps
that run anywhere that will run a browser.

Rather like a drunken tattoo of Baphomet acquired at a time when you'd no idea
you would later wish to join the clergy, the issue now is how to deal with the
consequences of this unforseeable unfortunate choice. There are multiple
options. The first is simply to join the occult, JavaScript has its redeeming
features (as, I'm sure, so does Satanism) and some espouse adopting it on
**both** the client and server side of things.
[node.js](https://nodejs.org/en/) facilitates this, this while
[MongoDB](https://www.mongodb.com/) and [CouchDB](http://couchdb.apache.org/)
allow even database queries to be scripted in JavaScript.

Another option when confronted with an unwanted tattoo of a Satanic goat etched
onto your chest (we've all been there) is to embilish it and render it less
offensive to whatever type of people it is normally make up vicar school
interview panels. Altering it so it resembles a much-missed former family pet,
say, or a favourite Disney character. This is the path favoured by JavaScript
libraries such as [jQuery](https://jquery.com/) or
[MooTools](http://mootools.net/), libraries which take JavaScript and build,
using JavaScript, a better JavaScript - one with nice functional programming
constructs and sane universal ways of accessing the DOM.

There is, however, a third way to hide your ill-advised ink, this is the route
chosen by Emscripten and the subject of this tutorial - it is simply to to cover
it up. The Satanic symbol will remain there underneath but the outside world can
deal with something much more old-fashioned and acceptable, like a Fair Isle
patterned jumper, or, as in Escripten's case, C++.

Emscripten's functionality is simple, it takes normal C or C++, and with the
help of parts of the [LLVM](http://llvm.org/) compiler tool-chain produces not
assembler code as normal compilers do, but JavaScript that can run in any
(modern) browser.

There are a number of advantages of this. Firstly, if you don't know JavaScript
but do know C++, you can write C++, not JavaScript. Secondly, you get to take
advantage of using a strongly-typed language if you like that sort of thing. A
strict compiler with static analysis, whilst being occassionally frustrating,
can catch a large number of bugs before the compiler even lets you run your
code, bugs that the more forgiving JavaScript will often leave for your users to
discover. Finally, it is possible to use the fact that C/C++ is a language
designed to be compiled into optimised assembler, running quicker than assembler
any human could write, to compile C/C++ into optimised JavaScript that runs
quicker than JavaScript any human could write.

This optimised subset of JavaScript is known as [asm.js](http://asmjs.org/), a
strict sub-set of JavaScript where the features chosen to be included are
designed to be suitable for being aggressively optimised by JavaScript
interpreters. The specific subset of asm.js was originally chosen and designed
by Mozilla, and a highly optimised interpretter inplemented as part of the
Firefox browser, but implementations of optimised asm.js interpretters are now
implemented into the Chrome and Edge browsers.

Emscripten can be seen as more or less a drop-in replacement for C compilers
gcc or clang, C/C++ files are compiled and linked into a JavaScript executable
which can then be incorporated into a website and run in any web-browser. That
it can compile C/C++ with few alterations means that it is suited to compiling
existing libraries and making their functionality available in the browser with
minimal effort. Thus, rather than writing a 2D physics engine in JavaScript,
and pulling your hair out trying to get it to run fast enough to power your
simulation of irritable birds before your game is beaten to market, you can
simply compile an existing C/C++ library like Bullet and job done. As an
illustration of what web-browsers can achieve if you only ask them nicely, some
show-offs compiled the [Unreal 3D](https://www.youtube.com/watch?v=BV32Cs_CMqo)
game engine into asm.js and the results are very impressive indeed.
 
This tutorial series will put you on the path to becoming that impressive,
using an Emscripten port of SDL2 to implement all the basic components of a
game, loading and displaying an image, moving it, and listening for user input.
SDL2 was designed to be a thin layer of abstraction over a computer's graphics,
input, and audio components, and so it proves in JavaScript, the port of SDL2
enables you to effortless, and while barely noticing it, leverage technologies
such as webGL and so you can rest at ease that your application will be using
the client's native graphics drivers if the brower supports it.

Before we start all that though we should concentrate on walking before we can
run and start off the same way that any adventure in silicon tends to start, by
corralling Emscripten to say 'hello world'.

[**Part 3: Hello World**](/2016/06/emscripten-and-sdl2-tutorial-part-3-hello-world/)
