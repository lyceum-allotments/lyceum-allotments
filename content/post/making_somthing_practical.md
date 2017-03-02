+++
date = "2016-06-19T00:00:00+01:00"
draft = false
title = "Emscripten and SDL2 Tutorial Part 8: Making Something Practical"
series = "Emscripten and SDL 2 Tutorial"
image = "emscripten_tutorial.png"
+++

At long last we've reached a position where we can make a 2D game that is
something like fully functional. As an illustration I've made a simple version
of Snake, which can act as a guide for any projects that you'd like to do.

It doesn't really cover anything conceptually new that we haven't covered
before, there's just more of it, with multiple source files to be compiled. One
problem that will be encountered if you just try and compile it using the
previous command is that when you try and run the program you'll be told that
you've run out of memory.

For dynamic memory, Emscripten declares a JavaScript typed array and passes
'pointers' to that whenever the program requests some dynamic memory. To deal
with a memory problem we simply need to make this typed array larger. By passing
the command line argument `-s TOTAL_MEMORY=67108864` to `emcc` a typed array of
size 67108864 bytes (that is, 2^26 bytes) will be used, which should be enough.

If you're not sure how much memory your program is going to use it is possible
to pass the `emcc` command line argument `-s ALLOW_MEMORY_GROWTH=1` which allows
the memory array to grow if needs be, but this involves a performance cost.

[[snake.html](/pages/snake.html), [snake.tar.gz](/code/snake.tar.gz), 
[snake.zip](/code/snake.zip)]
