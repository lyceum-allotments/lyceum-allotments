+++
date = "2016-06-19T00:00:07+01:00"
draft = false
title = "Emscripten and SDL 2 Tutorial Part 1"
series = "Emscripten and SDL 2 Tutorial"
image = "emscripten_tutorial.png"
+++

[Emscripten](https://kripken.github.io/emscripten-site/)
 is a compiler that allows you to take standard C/C++ and compile it
to JavaScript, making it possible to port your C/C++ programs and run them in
any modern browser. This tutorial series aims to be your guide and lead you to using the
[SDL2](https://www.libsdl.org/) library to implement all the basic components of a 2D game; showing an
image, moving an image and listening for user input, and enabling you to make a
game like this example of 'snake' with relative ease in C/C++, all without
writing a single line of JavaScript.


<a href="/pages/snake.html">
<img style="display:block;margin-left:auto;margin-right:auto" src="/images/snake_screen_shot.png" alt="snake screenshot">
</a>

The tutorial is broken into the following stages, that take you from an
introduction into what Emscripten does to making a customised web-site to host
your application:

* [**Introduction to LLVM, Emscripten and asm.js**](/2016/06/emscripten-and-sdl2-tutorial-part-2-introduction-to-llvm-emscripten-and-asm.js/)
-- a discussion of the various tools that will be used and how they work together

* [**Hello World**](/2016/06/emscripten-and-sdl2-tutorial-part-3-hello-world/)
-- the Emscripten compiler is downloaded and the classic hello world program written and compiled

* [**Look, Owl**](/2016/06/emscripten-and-sdl2-tutorial-part-4-look-owl/) -- the SDL2 media library is used with Emscripten to display a
  delightful image of an owl

* [**Move, Owl**](/2016/06/emscripten-and-sdl2-tutorial-part-5-move-owl/) -- upping the complexity level and moving the owl depending on
  user input

* [**Write Owl**](/2016/06/emscripten-and-sdl2-tutorial-part-6-write-owl/) -- using the SDL2_TTF font rendering library to put some text
  into the scene

* [**Get Naked, Owl**](/2016/06/emscripten-and-sdl2-tutorial-part-7-get-naked-owl/) -- making some final tweaks to the presentation of our
  application

* [**Making Something Practical**](/2016/06/emscripten-and-sdl2-tutorial-part-8-making-something-practical/)-- a bigger project, a game of snake, is
  discussed and issues surrounding bigger projects addressed

I hope this tutorial's helpful to you and wish you all the best as you dive into
the brave new world of transpiling C/C++ into JavaScript!
