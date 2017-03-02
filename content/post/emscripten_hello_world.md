+++
date = "2016-06-19T00:00:05+01:00"
draft = false
title = "Emscripten and SDL2 Tutorial Part 3: Hello World"
series = "Emscripten and SDL 2 Tutorial"
image = "emscripten_tutorial.png"
+++

The Emscripten SDK containing the Emscripten compiler can be downloaded
[here](https://kripken.github.io/emscripten-site/docs/getting_started/downloads.html).
On Linux and MacOS some dependencies need to be installed prior to installing
the compiler, details of these and instructions for their installation can be
found [here](https://kripken.github.io/emscripten-site/docs/getting_started/downloads.html#platform-notes-installation-instructions-portable-sdk)
but after those are dealt with the installlation of Emscripten itself is quite
straightforward; if you're using the portable SDK it's a case of unzipping it in
a convenient place, changing into the 'emsdk_portable' directory and running the
following commands which fetch the latest online tools from the web, installs
them, and makes them active:

{{< highlight c >}}
./emsdk update 
./emsdk install latest
./emsdk activate latest
{{< / highlight >}}

Linux and MacOS X require a further step to set the system path to the active
version of Emscripten:

{{< highlight c >}}
source ./emsdk_env.sh
{{< / highlight >}}

If all this has worked as it should, you should have the Emscripten compiler,
`emcc`, at the file path `./emscripten/master/emcc` in the `emsdk_portable`
directory you downloaded. You can add this to your system path, or just
reference the whole path whenever you want to invoke the compiler.

To check that all this has proceeded as expected we need to write a short test
program and compile it to JavaScript. The standard C implementation of the
classic 'Hello World' program will do the trick nicely:

{{< highlight c >}}
#include <stdio.h>

int main()
{
    printf("hello world!");
    return 0;
}
{{< / highlight >}}
[[hello_world.tar.gz](/code/hello_world.tar.gz), 
[hello_world.zip](/code/hello_world.zip)]

Compile with the Emscripten C compiler with the following command:

{{< highlight c >}}
emcc hello_world.c -o hello_world.html
{{< / highlight >}}

and you should get the files `hello_world.html` and `hello_world.js` appearing
in your working directory. `hello_world.js` is the guts of your program, the
JavaScript that your C program has been compiled into. The `-o hello_world.html`
argument which was passed to `emcc` told `emcc` to also generate an HTML file,
`hello_world.html`, which interfaces with the compiled JavaScript, including the JavaScript
in an HTML document and also defining what should be done with the C program's
output. It's possible to write this yourself, but we'll concentrate on
other things first, for now Emscripten's way of dealing with output is
fine, via a `<textarea>` element for text output and a `<canvas>` element for
anything graphical we'll be doing. If you open `hello_world.html` in your
browser you should see the Emscripten HTML document with the words `hello
world!` in the text area that's been defined to handle stdout. I warn you, it's
pretty spectacular. 
