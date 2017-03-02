+++
date = "2016-06-19T00:00:02+01:00"
draft = false
title = "Emscripten and SDL2 Tutorial Part 6: Write Owl"
series = "Emscripten and SDL 2 Tutorial"
image = "emscripten_tutorial.png"
+++

One final component of a game that I will cover in this tutorial is text.
Having a look at the available Emscripten ports (`emcc --show-ports`) will
reveal a library called [SDL2_ttf](https://www.libsdl.org/projects/SDL_ttf/) is
at your disposal. This is a library that enables you render true type fonts
into an `SDL_Surface`, that can then be rendered in a similar fashion to what
we've done previously.

`--show-ports` tells us that the command line argument we need to pass to emcc is
`-s USE_SDL_TTF=2` so all we need to do is add that to our command line, giving
us...

{{< highlight c >}}
emcc write_owl.c -O2 -s USE_SDL=2 -s USE_SDL_IMAGE=2 -s SDL2_IMAGE_FORMATS='["png"]' \
    -s USE_SDL2_TTF=2 --preload-file assets -o write_owl.html
{{< / highlight >}}

in our code we need to remember to include the SDL_TTF header file:

{{< highlight c >}}

#include <SDL/SDL_ttf.h>

{{< / highlight >}}

and then we can use the SDL_TTF functions to load a true type font (that we have
uploaded to our virtual filesystem, like our image, by placing in a preloaded
directory) using:

{{< highlight c >}}

TTF_Font *TTF_OpenFont(const char * file_path, int ptsize)

{{< / highlight >}}

where `file_path` is the path to the true type font in the virtual filesystem, and
`ptsize` is the size of the font. 

You can render some text to an SDL_Surface by passing the resultant TTF_Font
pointer into the following function:

{{< highlight c >}}

SDL_Surface *TTF_RenderText_Blended(TTF_Font *font, const char *text, SDL_Color fg);

{{< / highlight >}}

where font is the TTF_Font pointer, text is a pointer to the string you want to
render, and fg is the colour you want it rendered in. This surface is then just
made into a texture and rendered in same way as we did for the previous owl
image. See the soure code below for details!

[[write_owl.html](/pages/write_owl.html), [write_owl.tar.gz](/code/write_owl.tar.gz), 
[write_owl.zip](/code/write_owl.zip)]
