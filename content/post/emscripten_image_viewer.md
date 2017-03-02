+++
date = "2016-08-03T10:14:48+01:00"
draft = false
title = "Emscripten Image Viewer"

+++

Recently I've ben playing around with developing a zoomable and panning
image viewer using Emscripten and its proved to be a good exercise in accessing
parts of the page's HTML document outside of the Emscripten run-time.

<a href="/pages/newsham_park/newsham_park.html">
<img style="display:block;margin-left:auto;margin-right:auto" src="/images/complete_image_viewer_screen_shot.png" alt="complete image viewer screenshot">
</a>

The end goal is to provide a JavaScript API to an image viewer, consisting of an
`ImgViewer` object and a single method:

 * `ImgViewer(image, canvas)` -- sets up the viewer to use the
   `HTMLCanvasElement`, `canvas` and show a pannable and zoomable copy of the
   `HTMLImageElement`, `image`
 * `ImgViewer.changeImage(image)` -- changes the image shown by the image viewer to the
   `HTMLImageElement`, `image`

The main bit of ground here that I didn't cover in my previous
[tutorial on Emscripten and SDL2](/2016/06/emscripten-and-sdl-2-tutorial-part-1/),
is how to somehow get the image data from the `HTMLImageElement` to be
accessible from the Emscripten compiled C program.

Previously, images were loaded using Emscripten's concept of a virtual
filesystem, where assets are 'preloaded' with an XML HTTP request. This is very
good for keeping changes down to a minimum when porting desktop applications to
run in the browser, but it can be equally useful to access parts of the HTML
document from within an Emscripten compiled C program. In this post, the image
data from an `HTMLImageElement` will be accessed and used within SDL2 to make a
zoomable and panning image viewer, but the concepts explored here could be used
for any number of applications, using the speed of JavaScript produced Emscripten
to do client-side image processing is one example.

What C Wants
------------

Let's tackle this problem from the C side of things back, outwards, to what we
need to do in JavaScript. What our C program will require is a pointer to the
chunk of memory containing the image data we wish to show in our image viewer
(along with how many bytes make up the image data). Once we have this, SDL has
the concept of a [`SDL_RWops`](https://wiki.libsdl.org/SDL_RWops) structure, to
provide a common interface for reading from all sorts of stream devices. Using
the function

{{< highlight c >}}
SDL_RWops* SDL_RWFromConstMem(const void *mem, int size);
{{< / highlight >}}

can create a `SDL_RWops` from a pointer to some constant memory.

This can then be passed to the SDL_image function,
[`IMG_Load_RW`](http://jcatki.no-ip.org:8080/SDL_image/SDL_image.html#SEC12):

{{< highlight c >}}
SDL_Surface *IMG_Load_RW(SDL_RWops *src, int freesrc);
{{< / highlight >}}

which reads the memory referred to by `SDL_RWops` and gives us a SDL_Surface
pointer that we can turn into a texture and render in the
[usual way](/2016/06/emscripten-and-sdl2-tutorial-part-4-look-owl/).

<span id="load_image_section">
In code, the C function for changing the image will look something like this:
</span>

{{< highlight c >}}
struct context
{
    SDL_Renderer *renderer;

    SDL_Texture *img_tex;
};
struct context ctx;

// Set up renderer in context before calling this function

void load_image(void *image_data, int size)
{
    SDL_RWops *a = SDL_RWFromConstMem(image_data, size);
    SDL_Surface *image = IMG_Load_RW(a, 1);

    ctx.img_tex = SDL_CreateTextureFromSurface(ctx.renderer, image);

    SDL_FreeSurface(image);
}
{{< / highlight >}}

Providing What C Wants
----------------------

We've a long way to go yet though. The next step is to be able to call the C
function from JavaScript, passing it a pointer to some 'memory' we've previously
written to in JavaScript. I've put 'memory' in inverted commas because
JavaScript isn't the sort of language that goes around just handing out access
to raw memory. I've alluded in the past to how Emscripten provides C programs
with the illusion of memory through
[typed arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Typed_arrays)
-- similar in interface to a JavaScript
[`Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array)
but containing elements all of same type. In the same way that C programs can cast
a `char *` to an `int *` and by doing so change what data-type the array is
treated as, JavaScript has the concept of a 'view' of an array buffer;
`var ta = Uint8Array(buffer)` will view the array buffer `buffer` as an array of
unsigned 8 bit integers, `var tb = Uint32Array(buffer)` will access the same
data as an array of unsigned 32 bit integers.

As an illustration, 
{{< highlight javascript >}}
var ta = new Uint8Array([1,2,0,1]);
console.log(ta[0]); // prints 1
var tb = new Uint32Array(ta.buffer);
console.log(tb[0]); // prints 16777729 (1 + 2 * 2^8 + 0 * 2^16 + 1 * 2^24)
{{< / highlight >}}

Which is broadly similar to doing the following in C:
{{< highlight c >}}
uint8_t ta[] = {1,2,0,1};
printf("%u\n", ta[0]); // prints 1
uint32_t *tb = (uint32_t *)ta;
printf("%u\n", tb[0]); // prints 16777729
{{< / highlight >}}

Typed arrays can be passed C side by the Emscripten provided JavaScript
functions `ccall` and `cwrap`. I've discussed `ccall`
[before](/2016/06/emscripten-and-sdl2-tutorial-part-7-get-naked-owl/#ccall_section),
`cwrap` is similar, but returns a JavaScript function that can be used for calling
the 'wrapped' C function rather than calling it directly. If a C function
`void print_array(void *a)` has been exported when Emscripten was compiled, a
function, `print_array_js` taking a typed array as an argument can be created
like so:

{{< highlight javascript >}}
print_array_js = Module.cwrap('print_array', NULL, ['array']);
{{< / highlight >}}

Putting all these concepts together let's show how you'd create an array of
unsigned 8-bit integers in JavaScript and print them in the C program.

The guts of the HTML document will be:

{{< highlight html >}}
    <textarea id="output"></textarea>
    <script type='text/javascript'>
        var print_array = 0; // when Emscripten run-time loads, will be overwritten
                             // by function wrapper over a C print_array function

        // Creates a typed array and passes this to C function for printing
        var start_function = function() {
            var array = new Uint8Array([2,4,6,8]);
            print_array(array);
        };

        var Module = {
            onRuntimeInitialized: function() {
                // On load of Emscripten run-time, wrap the print_array C function and
                // call 'start_function' to get us underway
                print_array = Module.cwrap('print_array', null, ['array']);
                start_function();
            }, 
            print: function(t) {
                var element = document.getElementById('output');
                element.value += t + "\n";
            },
        };
    </script>
{{< / highlight >}}

while the C program will look like:

{{< highlight C >}}
#include <stdio.h>
#include <stdint.h>

void print_array(uint8_t *a)
{
    printf("array is [%u, %u, %u, %u]\n", a[0], a[1], a[2], a[3]);
}
{{< / highlight >}}

[[print_array.html](/pages/print_array.html),
[print_array.tar.gz](/code/print_array.tar.gz),
[print_array.zip](/code/print_array.zip)]

As a next step, let's put these ideas to work through the task of printing
'hello world' via passing an array of C characters (or, equivalently, an array
of unsigned 8 bit integers) from JavaScript to a C program. This is mostly
unnecessary as Emscripten allows you to pass JavaScript `String` types but
nontheless it will come in useful in the next step.

The important function at this juncture is the JavaScript `String` method
[`charCodeAt(index)`](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/String/charCodeAt)
which returns the character code (for our purposes, the ASCII value) of the
character at position `index` in the string.

This means that if we replace `start_function` from the previous example with
the following:

{{< highlight javascript >}}
var start_function = function() {
    var test_string = "hello world!";
    var array = new Uint8Array(test_string.length + 1);

    for (var i = 0; i < test_string.length; i++)
    {
        array[i] = test_string.charCodeAt(i);
    }
    array[test_string.length] = 0;

    print_array(array);
};

{{< / highlight >}}

and the `print_array` C function with:

{{< highlight c >}}
void print_array(char *a)
{
    printf("string is '%s'\n", a);
}

{{< / highlight >}}

then we can see that we've successfully converted a JavaScript `String` into a C
string and passed it from JavaScript to the Emscripten run-time. This trick will
turn out to be handy in the next section where we will end up converting the
bytes of the image we want to view in our viewer into a `String` and then converting
that, in turn, into a `Uint8Array` for passing into the C program.

It should probably be mentioned here that if you're going to try and write to
the array in the C code and expect your changes to be still there when you
access them back in JavaScript land you're going to be sorely disappointed,
array function arguments are passed on the stack in Emscripten and so don't
persist. There are ways of achieving persistence, but they'll have to wait for
another day.

A Picture Paints 1000 Words (if you know how to make it)
--------------------------------------------------------

We now have the capability of passing arbritrary data from JavaScript to an
Emscripten C function and, if that arbritrary data happens to be the bytes of a
png file, can turn that into an SDL_Surface. The final step then is to take an
`HTMLImageElement` and somehow get the bytes of the underlying png image from
it.

You have to take a somewhat roundabout route to achieve this, but it's do-able.
As an overview, you have to load the image element onto a canvas, and then use
the [`toDataURL()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/toDataURL)
method of the canvas to get a data URI for the image. For the uninitiated, a
data URI is a base64 encoded (that is an encoding where a character is
represented by 6 bits rather than 8) string which is the binary representation
of an image. The idea is that data URIs can be used to include an image file
directly in an HTML document via the `src` attribute of an image element rather
than the more usual method of setting the `src` attribute to a URL for which a
separate request must be made. By converting this base64 JavaScript string to be
ASCII and loading it into a `Uint8Array` for passing to the C program we can
achieve our goal of making the bytes of a HTMLImageElement accessible by SDL2.
There's even a JavaScript helper function `atob()` which converts a base64
encoded string to ASCII for us.

Doing all this, the JavaScript for getting the bytes from an image will look
something like this:

{{< highlight html >}}
<img src="static/owl.png" onLoad="get_bytes(this)"/>
<script type='text/javascript'>
    // get bytes of the image
    var get_bytes = function(img){
        // create a temporary canvas element for this task
        var canvas = document.createElement('canvas');

        // set it to have same size of underlying image
        canvas.width = img.naturalWidth;
        canvas.height = img.naturalHeight;

        // load the image in the canvas
        canvas.getContext('2d').drawImage(img, 0, 0);

        // convert the image to a data URI.
        // A data URI has a prefix describing the type of data the string 
        // following contains, we don't need this so just replace it with nothing
        var string_data = canvas.toDataURL('image/png')
            .replace(/^data:image\/(png|jpg);base64,/, '');

        // change the data URI from a base64 to an ASCII encoded string
        var decoded_data = atob(string_data);

        // make an unsigned 8 bit int typed array large enough to hold image data
        var array_data = new Uint8Array(decoded_data.length);

        // convert ASCII string to unsigned 8 bit int array
        for (var i = 0; i < decoded_data.length; i++)
        {
            array_data[i] = decoded_data.charCodeAt(i);
        }

        // just show this data on the console, for now...
        console.log(array_data);
    };
</script>
{{< / highlight >}}

For now, we just print the byte array to console rather than sending it to the C
program, but we can see from looking at the array that the bytes two to four are
the values `80`, `78`, `71`, that is the ASCII codes for the letters 'P', 'N'
and 'G', as required by the [file header of a png file](https://en.wikipedia.org/wiki/Portable_Network_Graphics#File_header).
It's starting to ominously look like things are going to work...

Rather than just logging the data, we just need to pass it to that C function
[`load_image`](#load_image_section) we discussed writing so long ago. There are
one or two subtleties that should be taken into account; you need to be aware
that the image byte getting function may well end up being called before the
Emscripten run-time is loaded and the exported image loading function set up,
or, conversely, that Emscripten may be loaded and ready to go before any image
bytes have been got and so there is no image to show. I circumvent this
potential problem by initialising a `load_image` variable to 0 and later
overwriting it with the wrapped `load_image` C function once Emscripten has
loaded, and having a `image_bytes` variable, similary set to 0 and overwritten
with the image byte array once the image bytes have been retrieved. In this way
you can figure out in the code when the `load_image` function or `image_bytes`
are available.

The example below is very simple, featuring two `HTMLImageElement`s and having
the `onClick` handler of both of them call an exported C function that changes
the texture that SDL shows (after remembering to free any previously created
texture, of course!). The initialisation of SDL and creation of the
`SDL_Renderer` are done only once, in a separate exported C function, and since
the context is kept in a virtual scope it is persistent and available after this
initialising function is called and for every subsequent call of `load_image`.
See the source code below for details...

<a href="/pages/show_img.html">
<img style="display:block;margin-left:auto;margin-right:auto" src="/images/basic_image_viewer_screen_shot.png" alt="basic image viewer screenshot">
</a>

[[show_img.html](/pages/show_img.html),
[show_img.tar.gz](/code/show_img.tar.gz), [show_img.zip](/code/show_img.zip)]

Wrapping It All Up
------------------

Our image viewer as-is provides the bare bones for a zoomable image viewer;
the bytes of `HTMLImageElements` can be passed to an Emscripten C program where we
can load them into an `SDL_Texture` and from that point any manipulations such
as panning or zooming will be much easier to achieve and possibly handled by
webGL if the client supports it.

It remains though to create a nice interface to all this, this means providing
the neat object interface to an image viewer mentioned at the start, bundling
the JavaScript code that's been produced so it can be easily included in a page
and protecting the scope of the produced code so that the variables don't polute
the global JavaScript scope and a user can easily include multiple image viewers
in a page.

All the JavaScript that Emscripten's produced so far has started with something
similar to the following bit of JavaScript:

{{< highlight javascript >}}
var Module;if(typeof Module==="undefined")Module={}; // and then a long load of
                                                     // hard to understand JavaScript
{{< / highlight >}}

In other words, a Module object is introduced into the current scope (in
examples we've done so this has been the global scope). This is bad from the
point of view of providing a neat encapsulated interface; a user naively including
this code twice in a page would simply overwrite all Emscripten's carefully
declared variables and they wouldn't get the two image viewers they were
wanting.

What is needed is a function to be defined encapsulating all this code which can
be called with a set of options and each time sets up a separate Emscripten
runtime. As luck would have it a couple of `emcc` compile time options can
achieve that very effect. The setting `-s MODULARIZE=1` tells Emscripten to
produce the following JavaScript instead of that described above:

{{< highlight javascript >}}
var EXPORT_NAME = function(Module) {
    Module = Module || {};

    var Module; // and then a long load of hard
                // to understand JavaScript
};
{{< / highlight >}}

`EXPORT_NAME` is a string defined by the `emcc` setting `-s EXPORT_NAME`. For
example, including the setting `-s EXPORT_NAME="'EmImgViewer'"` when `emcc` is
called will lead to a function named `EmImgViewer` being added to the current
scope. Calling this function with a `Module` configuration object, the same as
the Module object we were defining in global scope previously, will lead to an
Emscripten runtime being created. Calling `EmImgViewer` will create an
Emscripten image viewer, calling `EmImgViewer` again, with a different
configuration option, perhaps specifying a different canvas and different
functions to use to wrap exported functions, will create a completely separate
`EmImgViewer`.

This is still not the friendliest interface to provide to a user; firstly, they
have to know all about Emscripten and the configuration option object they need
to pass to `EmImgViewer`, secondly, they need to know about C functions that are
exported by the `EmImgViewer` so they can wrap them in JavaScript functions and
so manipulate the viewer. For this reason, we need to hand-craft a bit more
JavaScript to neaten up the interface and make the image viewer more easily
usable.

We will put this in a file, `post_img_viewer.js`, that will be appended to the
compiled JavaScript and define an `ImgViewer` object. When constructed, through
a call to `ImgViewer(image, canvas)`, the user specifies an initial image and a
canvas to use for a display. At this point the Emscripten runtime is created, a
call back defined to set-up the viewer when the runtime is ready, and the bytes
of the image element are prepared for viewing.
The construtor of the object will look something like this:

{{< highlight javascript >}}
ImgViewer = function(image, canvas_element) {
    // Uint8Array of bytes of image will be stored here
    var image_bytes = 0;

    // Emscripten exported function for loading an image into
    // the viewer will be stored here
    var loadImage = 0;

    var loadImageBytes = function(image) {
        // function gets bytes of `image` and stores an array of 
        // these bytes in image_bytes
    };

    var options = {
      canvas: (function() {
          return canvas_element; // set the canvas to be what the
                                 // user specified
      })(),
      onRuntimeInitialized: function() {
          this.ccall('setup_context', null); // once runtime is initialized
                                             // call the C function to
                                             // initialize SDL and set up the
                                             // context

          // wrap the `load_image` C function in a private method of 
          // this object
          loadImage = this.cwrap('load_image', null, ['array', 'number']);

          if (image_bytes) // if an image has already been loaded by the time
                           // this call back is called, load the image bytes
                           // into the viewer
          {
            loadImage(image_bytes, image_bytes.length);
          }
      },
    };

    // load the initial image
    loadImageBytes(image);

    // create an image viewer with the options
    EmImgViewer(options);

    // TODO return the object interface...
{{< / highlight >}}

After the object is constructed, we should return the object's interface -- the set
of public methods that the user can call. In this case, the interface is one
method, `changeImage`. This is quite simple, all it has to do is load the image
bytes from the `HTMLImageElement` passed to it as an argument and then call the
wrapped `load_image` C function to put these bytes onto the canvas:

{{< highlight javascript >}}
    // returning the object interface...
    return {
        'changeImage' : function(image) { // method to change viewer's image
                                          // to the HTTPImageElement `image`
            loadImageBytes(image);
            if(loadImage)
                loadImage(image_bytes, image_bytes.length);
        }
    };
};
{{< / highlight >}}

Upon altering the `Makefile` of the project to concatenate this JavaScript file
at the end of the Emscripten produced JavaScript, we are left with one script
file that our image viewer users should include in a page in order to use our
image viewer. All that is required in order to use the image viewer is that this
script be included and the `ImgViewer` constructor called specifying the `image`
to be shown and the `canvas` to show it on. An example of how to do this is
given below:

{{< highlight html >}}

<script type='text/javascript' src='img_viewer.js'></script>
<script type='text/javascript'>
    var setup_viewer = function(image) {
        var canvas = document.getElementById('canvas');
        viewer = ImgViewer(image, canvas);
    };
</script>
</head>
<body>

<div>
    <img src="static/owl.png" onLoad="setup_viewer(this)" onClick="viewer.changeImage(this)"/>
    <img src="static/cat.png" onClick="viewer.changeImage(this)"/>
    <canvas id="canvas"></canvas>
</div>
{{< / highlight >}}

It is worth reemphasising that because the Emscripten code is now nicely
encapsulated, there is nothing to stop an `ImgViewer` being instantiated being
given a different `canvas` element and voila, with no effort you have two image
viewers on one page! See the example and source code below for details:

<a href="/pages/img_viewer/img_viewer.html">
<img style="display:block;margin-left:auto;margin-right:auto" src="/images/double_image_viewer_screen_shot.png" alt="double image viewer screenshot">
</a>
[[img_viewer.html](/pages/img_viewer/img_viewer.html),
[img_viewer.tar.gz](/code/img_viewer.tar.gz),
[img_viewer.zip](/code/img_viewer.zip)]

The Final Product
-----------------

The steps that remain now are all SDL and C. Events such as mouse button clicks
must be listened to and converted into manipultions of the `SDL_Texture`
containing the image. There is nothing specific to Emscripten here except for
the calling of `emscripten_set_main_loop` rather than having a main animation
loop, as described [here](/2016/06/emscripten-and-sdl2-tutorial-part-5-move-owl/).

Links to an example page using the `ImgViewer` and to the github repository
containing the `ImgViewer` library are included below.

<a href="/pages/newsham_park/newsham_park.html">
<img style="display:block;margin-left:auto;margin-right:auto" src="/images/complete_image_viewer_screen_shot.png" alt="complete image viewer screenshot">
</a>

[[newsham_park.html](/pages/newsham_park/newsham_park.html), [img_viewer_repository](https://github.com/lyceum-allotments/img_viewer)]
