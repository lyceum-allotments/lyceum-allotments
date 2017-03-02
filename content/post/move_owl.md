+++
date = "2016-06-19T00:00:03+01:00"
draft = false
title = "Emscripten and SDL2 Tutorial Part 5: Move, Owl"
series = "Emscripten and SDL 2 Tutorial"
image = "emscripten_tutorial.png"

+++

We can now use SDL2 to load and display an image. The next step is to learn how
to move this image, and then to use SDL's functions to listen for user input in
order to control the movement.

Moving the image turns out to be one of those things that needs to be done a
little differently to how it is normally done in native C/C++ programs. On those
occassions you typically have a main event loop and inside that main loop you
increment the position of the image a small amount on each iteration, something
like this:

{{< highlight c >}}

/**
 * Initialise SDL and open window and renderer here
 */

bool carry_on = true;

SDL_Surface *image = IMG_Load("assets/owl.png");
SDL_Texture *tex = SDL_CreateTextureFromSurface(renderer, image);
int x = 0;


/**
 * Have an infinite loop...
 */
while (carry_on)
{
  /**
   * Clear the screen
   */
  SDL_RenderClear(renderer);

  /**
   * Create a rectangle to copy the image into, with the x
   * location of the image set to the value of the 
   * variable x that will be incremented in the loop
   */
  SDL_Rect dest = {.x = x, .y = 100, .w = 200, .h = 200};

  /**
   * Copy the image into the renderer in the new location
   * defined by dest
   */
  SDL_RenderCopy (renderer, tex, NULL, &dest);

  /**
   * Render the screen
   */
  SDL_RenderPresent(renderer);
  
  /**
   * Increment x by 5 pixels, ensuring that next time we
   * render the image it is drawn to the right 5 pixels
   */
  x += 5;
}

{{< / highlight >}}

The problem of doing this in JavaScript is related to how JavaScript is run in
the browser. The browser event model uses cooperative multitasking as a
multitasking model, this means that each event must voluntarily give control
back to the process scheduler which can then allow another process to run for a
while (until that, in turn, gives control back to the process scheduler). When
an infinite loop is implemented in the manner above the JavaScript running
process will never give control back to the browser and so no other processes in
your browser will be able to run (in actuality, your browser will detect this and
offer you the chance to shut down the offending process).

Luckily, the Emscripten C API provides us with some functions that act like an
infinite loop, whilst playing nice and sharing, giving control back to the
browser periodically so that other things can be done. The one we will use here
is [`emscripten_set_main_loop_arg`](https://kripken.github.io/emscripten-site/docs/api_reference/emscripten.h.html#c.emscripten_set_main_loop_arg),
defined in the header file `emscripten.h`:

{{< highlight c >}}

void emscripten_set_main_loop_arg(em_arg_callback_func func, void *arg, int fps, int simulate_infinite_loop)

{{< / highlight >}}

Here, `func` is a function pointer to a function taking a void pointer as an
argument, `arg` is the void pointer that will be passed as an argument, `fps` is the
number of times per second that you wish `func` to be called and
the `simulate_infinite_loop` arguement we will get to in a moment.

It turns out that modern browsers support a method on the `window` DOM object
specifically for the purposes of animation. This is
`window.requestAnimationFrame()` and calls a function repeatedly at the same rate
as the browser refresh rate. If you're using the main loop for updating graphics
(as we will be) it is a waste of resources to update the image at a greater rate
than the browser is refreshing. By passing an `fps` argument to
`emscripten_set_main_loop_arg` equal to -1 Emscripten will use
`requestAnimationFrame` under the bonnet and so you will not refresh the graphics
faster than the browser, in general, it is a good idea to do this.

To get out of the main loop, Emscripten provides the function
[`emscripten_cancel_main_loop`](https://kripken.github.io/emscripten-site/docs/api_reference/emscripten.h.html#c.emscripten_cancel_main_loop):

{{< highlight c >}}

void emscripten_cancel_main_loop(void)

{{< / highlight >}}

which when called will simply stop the main loop looping. It's important to note
though, that the loop handler will not return.

Here's a simple example using `emscripten_set_main_loop_arg` and
`emscripten_canel_main_loop` to pass an integer to a loop function and increment
it until it equals 100 at which point the looping is stopped:

{{< highlight c >}}

#include <stdio.h>

/**
 * Provides emscripten_set_main_loop_arg and emscripten_cancel_main_loop
 */
#include <emscripten.h>

/**
 * A context structure that we can use for passing variables to our loop
 * function, in this case it just contains a single integer
 */
struct context
{
    int x;
};

/**
 * The loop handler, will be called repeatedly
 */
void loop_fn(void *arg)
{
    struct context *ctx = arg;

    printf("x: %d\n", ctx->x);

    if (ctx->x >= 100)
    {
        /**
         * After 101 iterations, stop looping
         */
        emscripten_cancel_main_loop();
        printf("got here...\n");
        return;
    }

    ctx->x += 1;
}

int main()
{
    struct context ctx;
    int simulate_infinite_loop = 1;

    ctx.x = 0;

    emscripten_set_main_loop_arg(loop_fn, &ctx, -1, simulate_infinite_loop);

    /**
     * If simulate_infinite_loop = 0, emscripten_set_main_loop_arg won't block
     * and this code will run straight away.
     * If simulate_infinite_loop = 1 then this code will not be reached
     */
    printf("quitting...\n");
}

{{< / highlight >}}

[[loop_test.tar.gz](/code/loop_test.tar.gz), 
[loop_test.zip](/code/loop_test.zip)]

This example can be simply altered to illustrate the effect of the
`simulate_infinite_loop` argument given to `emscripten_set_main_loop_arg`. When it
is set to 0, the call to `emscripten_set_main_loop_arg` won't block, code after it
will execute simulataneously to the looping. This means that if you have
clean-up code after your main loop has finished, you risk calling it and cleaning
up stuff that your main loop might still need. Setting `simulate_infinite_loop` to
1 will prevent this from happening, and the compiled JavaScript will stop the
execution of the caller at this point. This means that the code after the loop
will never be reached, note that this means it won't be reached even after
`emscripten_cancel_main_loop` is called!

With a straight-forward application of these principles we can alter our
`hello_owl` program so that we can listen to user input and move the owl picture
as appropriate. All we have to do is poll the SDL event system on each loop
iteration to check if the user has pressed any buttons and if so to react
appropriately. Depending on what button was pressed the destination rectangle
that the owl image is rendered into is moved.

An example implementing this can be seen and source-code downloaded below:

[[move_owl.html](/pages/move_owl.html), [move_owl.tar.gz](/code/move_owl.tar.gz), 
[move_owl.zip](/code/move_owl.zip)]
