+++
date = "2016-06-19T00:00:01+01:00"
draft = false
title = "Emscripten and SDL2 Tutorial Part 7: Get Naked, Owl"
series = "Emscripten and SDL 2 Tutorial"
image = "emscripten_tutorial.png"
+++

We're so close to being able to make the perfect game you can almost taste it,
there's just a few things in our way -- all that 'powered by Emscripten' gumpf
that associates itself with or masterwork every time we get Emscripten to
generate some HTML for us. That console might come in very useful when we were
needing to debug things, but now we're convinced that our code is completely,
absolutely, bug free and we don't need those crutches any more!

The time has come to write our own HTML and say good-bye to that 'powered by
Emscripten' header and debug consoles forever! Or at least until we discover
that, actually, we have got a bug...

To do this we need to understand a little better what Emscripten is doing when
it generates its JavaScript. A lot of it involves a
[`Module`](https://kripken.github.io/emscripten-site/docs/api_reference/module.html)
object, and the generated JavaScript will try and call various methods of this
object at various points. For example whenever `printf` (or anything writing to
the `stdout` stream) is called in C/C++ code an attempt to call the `Module.print` 
method is made and passed the text that was passed to printf. I say attempted
because when you don't tell `emcc` to generate HTML the `Module` object is not
defined and it is up to you to define it and any methods you would like to use.

This opens up to you such exciting possibilities of defining `Module.print` to pop
up irritating alerts containing the text, or log them to the JavaScript console,
or append the text to a `<div>`... or just do nothing with it, as we're going to do.
One method we are going to implement is `Module.canvas`. This is a method that
takes no arguments and returns a function that also takes no arguments and
returns the DOM `<canvas>` element that will be used as the 'window' by our app.

So to give the application a screen to show itself in, we need to put a `<canvas>`
element into the HTML and define a `Module` object with a `canvas` method that
returns a function returning this DOM element:

{{< highlight html >}}
<canvas id="canvas"></canvas>
<script type='text/javascript'>
var Module = {
    canvas: (function() {
        var canvas = document.getElementById('canvas');
        return canvas;
        })()
};
</script>
{{< / highlight >}}

Another thing that is essential is loading the file with the static memory via
an XML HTTP request. This bit of JavaScript looks moderately scary, but really
it is just defining where the file containing the static memory is (in this case
`naked_owl.js.mem`) and making a XML HTTP request to fetch this file:

{{< highlight js >}}
(function() {
  var memoryInitializer = 'naked_owl.js.mem';
  if (typeof Module['locateFile'] === 'function') {
    memoryInitializer = Module['locateFile'](memoryInitializer);
  } else if (Module['memoryInitializerPrefixURL']) {
    memoryInitializer = Module['memoryInitializerPrefixURL'] + memoryInitializer;
  }
  var xhr = Module['memoryInitializerRequest'] = new XMLHttpRequest();
  xhr.open('GET', memoryInitializer, true);
  xhr.responseType = 'arraybuffer';
  xhr.send(null);
})();
{{< / highlight >}}

The last thing is to load the main JavaScript (`naked_owl.js`) and append it to the HTML
document's body:

{{< highlight js >}}
var script = document.createElement('script');
script.src = "naked_owl.js";
document.body.appendChild(script);
{{< / highlight >}}

That is all that is essential for getting a graphical application to run, you
are now free to apply CSS to that page to your heart's content, but here are
some bonus nuggets of knowledge...

The `Module` object has a `requestFullScreen` method, that when invoked makes your
application go fullscreen. This takes two boolean arguments, the first says
whether to hide the mouse cursor or not, the second whether to keep the canvas
at its current resolution or to expand it to fullscreen and lower the
resolution.  Because of browser security there are some caveats to calling this,
it will only work when called from within a user triggered event, for example
clicking on a button. To make such a button an element something like this needs
to be added to the document:

{{< highlight html >}}
<p id="fullScreenButton" onclick="Module.requestFullScreen(true, false)">Click for full-screen</p>
{{< / highlight >}}

Another thing that can come in handy for impatient users is the display of a
loading screen. This can be achieved by putting some loading message or image on your HTML
page and then making it hidden when the Emscripten application is all loaded and
ready to go. When the application is all ready it calls the
`Module.onRuntimeInitialized()` call-back, so by providing this call-back as a
function that hides the loading message or image, the effect can be achieved of a
loading message that disappears when the game has loaded. This would make our
`Module` object look something like this:

{{< highlight html >}}
<canvas id="canvas"></canvas>
<div id="loadingDiv">The owl is loading....</div>
<script type='text/javascript'>
var Module = {
    onRuntimeInitialized: function() {
        var e = document.getElementById('loadingDiv');
        e.style.visibility = 'hidden';
         },
    canvas: (function() {
        var canvas = document.getElementById('canvas');
        return canvas;
        })()
};
</script>
{{< / highlight >}}

<span id="ccall_section">
Finally, we'll not start the application running straight away, but add a button
that starts the application. The `Module` object has a
[`ccall`](https://kripken.github.io/emscripten-site/docs/api_reference/preamble.js.html#ccall)
method that can be used to call a C function from JavaScript. It takes three
argumments, the first is a string containing the name of the function, the next
gives the expected return type, the next is an array describing the types of the
arguments the C function takes. By renaming our `main` function in our C
program, say by giving it the signature:
</span>

{{< highlight c >}}
void mainf();
{{< / highlight >}}

we can call it from a JavaScript function by doing something like this within
the HTML's JavaScript:

{{< highlight js >}}
var start_function = function() {
    Module.ccall('mainf', null, null);
};
{{< / highlight >}}

The very observant amongst you will notice that in the last couple of paragraphs
I've stopped referring to C/C++ and started referring to plain old C. The reason
for this is that `ccall` only supports C functions, if you have a C++ project and you
want to call a function from within JavaScript, you'll have to place the key
words `extern "C"` before the function definition. This is because of C++ [name
mangling](https://en.wikipedia.org/wiki/Name_mangling), where in C++ you can have
two functions with the same name and Emscripten wouldn't know which one you are
referring to.

Other problems with function names can also be encountered because Emscripten
compresses its compiled JavaScript and sometimes tampers with function names. To
stop it doing this with functions you wish to call from JavaScript you have to
pass the `-s EXPORTED_FUNCTIONS:'["_mainf"]'` command line argument to `emcc`.
Note the underscore before the function name, it's important!

To get `emcc` to only output JavaScript, rather than HTML, change the `-o`
argument to refer to a file with a `.js` subscript, like so:

{{< highlight c >}}
emcc naked_owl.c -O2 -s USE_SDL=2 -s USE_SDL_IMAGE=2 -s SDL2_IMAGE_FORMATS='["png"]' -s USE_SDL_TTF=2 \
    -s EXPORTED_FUNCTIONS='["_mainf"]' --preload-file assets -o naked_owl.js
{{< / highlight >}}

Source code implementing these ideas and a page showing their effect is included
below:

[[naked_owl.html](/pages/naked_owl.html), [naked_owl.tar.gz](/code/naked_owl.tar.gz), 
[write_owl.zip](/code/naked_owl.zip)]
