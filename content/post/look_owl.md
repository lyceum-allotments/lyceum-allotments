+++
date = "2016-06-19T00:00:04+01:00"
draft = false
title = "Emscripten and SDL2 Tutorial Part 4: Look, Owl"
series = "Emscripten and SDL 2 Tutorial"
image = "emscripten_tutorial.png"

+++

Although you can do plenty of interesting things with just logic and text, to
impress the man on the Clapham omnibus these days you generally need to do
something graphical. To this end we'll now write a program to load and show this
fetching picture of an owl coughing up a pellet:

<img style="display:block;margin-left:auto;margin-right:auto" src="/images/coughing_owl.png" alt="couging owl">

Solving this problem can be broken down into two broad stages: firstly how do we
refer to the image of an owl stored on a server's file system when it
will be viewed in a web-page visited from the client's computer where
JavaScript runs in a sandboxed environment? Secondly, even if we can access the
image file, how do we leverage the [SDL2](https://www.libsdl.org/) library to
load and display it on the screen?

Emscripten provides a number of ways to solve the first problem of making files
on the server accessible to C/C++ programs when they are run on a client, we
will look at two here. They both work in similar ways, a directory is specifed
and the files in that directory serialised and uploaded to the client where
Emscripten maps them to a virtual filesystem with the same layout as that of
where you compiled the program. The difference between the two methods lays in
where the files are serialised and how they are uploaded.

The first option involves embedding the files in the JavaScript itself, and the
files are loaded with the JavaScript. By default, the files to be loaded should
be stored in a directory nested inside the directory where you compile your
program from. You can then tell Emscripten to embed the files inside this
directory by passing the `--embed-file <directory>` command-line argument to
emcc at compilation.

{{< highlight c >}}

    hello_owl
    │   
    ├── hello_owl.c
    │   
    └── assets
        └── images
            └── owl.png

{{< / highlight >}}

Take as an example this file hierarchy, if we are in the `hello_owl` directory
and we compile `hello_owl.c` with the command:

{{< highlight c >}}
emcc --embed-file assets -o hello_owl.html
{{< / highlight >}}

then the filesystem from the `assets` directory downwards will be serialised and
compiled into the `hello_owl.js` JavaScript script. `hello_world.c` could now open
the file in the same way a native C/C++ program would by calling something like:

{{< highlight c >}}

FILE * fp = fopen("assets/images/owl.png", "r");

{{< / highlight >}}

This is all well and good, but embedding files straight in the JavaScript like
this isn't very efficient. The problem is that everything has to be loaded in
one go over one connection. It is better if the files are stored in a separate
file and loaded separately via a XML HTTP request, Emscripten can then make sure
that the compiled JavaScript only runs once this XML HTTP request has completed
and the virtual filesystem has been set up. 

This is achieved in a very similar way to how a filesystem is embedded, only
this method is known as preloading the files and you would compile `hello_owl.c`
with the following command:

{{< highlight c >}}
emcc --preload-file assets -o hello_owl.html
{{< / highlight >}}

This would result in a file `hello_owl.data` being produced upon compilation,
containing the filesystem information which would then be loaded via a XML HTTP
request on loading `hello_owl.html`. One small caveat with this method and doing
local testing on the Google Chrome or Microsoft Edge browsers is that these
browsers cannot load local files via XML HTTP requests, so local testing must be
done using a web-server as opposed to just opening `hello_owl.html` in the browser
(alternatively, you could just do local testing in Firefox).

A similar choice between embedding and preloading data occurs regarding
statically allocated memory in the C/C++ program (memory used for static
variables and such). In a program with a lot of local variables it becomes
inefficient to embed this memory in the JavaScript and upload over one
connection, in such a case it is a possible to do a thing similar to what we
just did with preloading file assets; by passing emscripten the `--memory-init-file
1` command line argument emscripten will put the memory for static variables in a
separate file (`hello_owl.html.mem`) which is loaded via a XML HTTP request on page
load. By telling Emscripten to use second level optimisation (the `-O2` command
line argument) the `memory-init-file 1` functionality will be turned on by default,
and that is what we will use from now on.

Regarding the second issue of how to access SDL2 to load an image, here we can
make use of a number of libraries that have been ported to, and can be used from
within, Emscripten. Using these ports is really easy, they are all hosted on
[github](https://github.com/emscripten-ports), and by passing the command line
argument `-s` followed by the appropriate port argument they can be accessed
from within your code. A list of ports and their corresponding names can be acquired
with the command:

{{< highlight c >}}
emcc --show-ports
{{< / highlight >}}

Doing this, we can see that Emscripten ports provides the SDL2 library and the
SDL2_image library and these are accessible by using the arguments `-s
USE_SDL=2` and `-s USE_SDL_IMAGE=2`. An additional subtlety that must be
observed with SDL2_image is that you must pass the image formats that you wish
SDL2_image to support, for example, to make SDL2_image support png images pass
the command `-s SDL2_IMAGE_FORMATS='["png"]'` at compilation.

Putting all this together with the previous discussion we reach the following
compilation command for a program that can load png images from a virtual
filesystem using the SDL2 library:

{{< highlight c >}}
emcc hello_owl.c -O2 -s USE_SDL=2 -s USE_SDL_IMAGE=2 -s SDL2_IMAGE_FORMATS='["png"]' \
    --preload-file assets -o hello_owl.html
{{< / highlight >}}

Now that we have the compilation command, all that remains for us to do is
actually write the program!

If you're familiar with SDL2 this step is straightforward because it's
identical to how you'd do it for compiling for a native machine! You can
download the `hello_owl.c` program source code below, it uses the SDL2_image
function `IMG_Load` to load an image file (loaded from the preloaded virtual file
system) and copies that to a renderer that is then used to display the image to
the screen using SDL2 functions.

[[hello_owl.tar.gz](/code/hello_owl.tar.gz), 
[hello_owl.zip](/code/hello_owl.zip)]

