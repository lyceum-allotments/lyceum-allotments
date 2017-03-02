+++
date = "2016-11-24T07:29:05Z"
draft = true
title = "Python and Pipes Part 4: On the Buffers"
series = "Python and Pipes"
+++

In the last section we looked at sending a message through a pipe and everything
worked great.  However there was a porblem, and this becomes apparent if we
alter the sending program and make it run a little slower, perhaps by adding a
short sleep inbetween sending each letter of the message, something like this:

{{< highlight python >}}
import time
message = "hello to a pipe\n"

with open("my_pipe", "w") as f:
    print "have opened pipe, commencing writing...."
    for c in message:
        f.write(c)
        print "have sent a letter"
        time.sleep(1)
{{< / highlight >}}

if we run this program and attempt to read from `my_pipe` in a different virtual
terminal with `cat my_pipe` we'll see that we need to show a bit of patience!
The terminal running the Python process will periodically print reassuring
messages saying it has sent letters, and yet we'll see no letters shown on the
virtual terminal running `cat my_pipe`, at least not until the end of the
message is reached, at which point the whole message will suddenly appear.

Unbuffered Writing
------------------

The root of this problem is buffering -- rather than being sent straight through
the pipe, Python's IO, by default, buffers the data, and this data is not
flushed until the `EOF` (end of file) signal is encountered.

Looking at the manual page for Python's
[`open`](https://docs.python.org/2/library/functions.html#open) function we see
that there is a way to control the size of this buffer, through the `buffering`
argument. `buffering` has a special meaning when it is set equal to 0, where
no buffering is done at all. Altering our slow write to have `buffering = 0`:

<span id="write_to_pipe_buf_0">
{{< highlight python >}}
import time
message = "hello to a pipe\n"

with open("my_pipe", "w", 0) as f:
    print "have opened pipe, commencing writing...."
    for c in message:
        f.write(c)
        print "have sent a letter"
        time.sleep(1)
{{< / highlight >}}
</span>

and running again, while attempting to read again with `cat my_pipe` in another
virtual terminal, we can see the effect of altering the buffering; each letter
gets shown one at a time.

The manual page for
[`open`](https://docs.python.org/2/library/functions.html#open) also mentions a
special value of `buffering = 1`, which is line buffered. This does what its
name implies, as can be seen by running this program, which has our message
broken down into lines, while reading from a pipe:

{{< highlight python >}}
import time
message = "hello\nto a\npipe\n"

with open("my_pipe", "w", 1) as f:
    print "have opened pipe, commencing writing...."
    for c in message:
        f.write(c)
        print "have sent a letter"
        time.sleep(1)
{{< / highlight >}}

Other values of `buffering` change the size of the chunks in which the data is
sent across the pipe, see for example what effect a `buffering` of `4` has on
this program:

{{< highlight python >}}
import time
message = "hello to a pipe\n"

with open("my_pipe", "w", 4) as f:
    print "have opened pipe, commencing writing...."
    for c in message:
        f.write(c)
        print "have sent a letter"
        time.sleep(1)
{{< / highlight >}}

This program sends the message across the pipe in chunks of approximately 4
bytes.

Unbuffered Reading
------------------

The natural counterpart to a Python program that does unbuffered writing is one
that does unbuffered reading, so let's write one now.

Firstly, trying our previous program for reading from a pipe,
[`read_from_pipe.py`](/2016/11/python-and-pipes-part-3-pipes-in-python#read_from_pipe),
(by running [`write_pipe_buf_0.py`](#write_to_pipe_buf_0) in one virtual
terminal, and
[`read_from_pipe.py`](/2016/11/python-and-pipes-part-3-pipes-in-python#read_from_pipe)
in another)
we see that it doesn't display the message being received letter by letter, but
that it instead blocks, the problem being that the iterator through the pipe's
`FILE` object, `for l in f`, doesn't commence until `f` has reached the `EOF`.

To read just a set number of bytes from a `file` object we need to try a
slightly different approach, using the
[`f.read(x)`](https://docs.python.org/2/library/stdtypes.html#file.read)
method which blocks until it reads `x` bytes from `f` or reaches `EOF`,
returning either a string of the data used or `None`, if the `file` object is
finished.

Feeling our way towards solving the problem, a program like this:

{{< highlight python >}}
import sys

bufsize = 1

with open("my_pipe", "r") as f:
    print "have opened pipe, commencing reading...."
    c = f.read(bufsize)
    print c,
{{< / highlight >}}

will read from 1 character from the pipe, print it and then the Python program
finishes. When we run it with [`write_pipe_buf_0.py`](#write_to_pipe_buf_0)
we get the first character of the message printed as soon as it is available in
the pipe. Increasing `bufsize` we see that we can alter the number of characters
that are printed,
[`read`](https://docs.python.org/2/library/stdtypes.html#file.read) blocking
until that many characters have been sent to the pipe.

Another bit of behaviour that should be notice is that when the process reading
from the pipe closes before the message being written has finshed being written
the writing process crashes, throwing an
[`IOError`]((https://docs.python.org/2/library/stdtypes.html#file.read)
exception complaining of a broken pipe. We won't concern ourselves with that
here.

If we want to read a message through a pipe letter by letter, then, we need to
loop, reading a character and, whenever one is available, printing it and waiting
for the next character. We break from the loop whenever `read` returns `None`,
i.e. when the message has ended. In short, something like this:

{{< highlight python >}}
with open("my_pipe", "r") as f:
    print "have opened pipe, commencing reading...."
    while 1:
        c = f.read(1)
        if c:
            print c,
        else:
            break
{{< / highlight >}}

The only problem with this is it doesn't work! Once again the message won't
display until it has completely sent and the pipe closed. The problem this time
is write-buffered IO at the operating system level. When writing to `stdout` (as
`print` does) the output is buffered, i.e. will only display when enough
characters have been printed. Once again the message won't display until it has
completely sent and the pipe closed. The problem this time is write-buffered IO
at the operating system level. When writing to `stdout` (as `print` does) the
output is buffered, i.e. will only display when enough characters have been
printed.

To circumvent this we need to go a little lower level and use Python's `sys`
module.
[`sys.stdout`](https://docs.python.org/2/library/sys.html?highlight=sys%20module#sys.stdout)
is a `file` object that represents Unix's `stdout` pipe, i.e. the pipe that when
written to gets displayed on the screen.

Doing something like:

{{< highlight python >}}
import sys
sys.stdout.write("hello world")
sys.stdout.flush()
{{< / highlight >}}

will write a string to `stdout` and then flush it, i.e. force what's been
written to be displayed on the screen. With this in mind by changing our
unbuffered reading program to this:

{{< highlight python >}}
import sys

with open("my_pipe", "r") as f:
    print "have opened pipe, commencing reading...."
    while 1:
        c = f.read(1)
        if c:
            sys.stdout.write(c)
            sys.stdout.flush()
        else:
            break
{{< / highlight >}}

and running this in concert with [`write_pipe_buf_0.py`](#write_to_pipe_buf_0)
in another buffer we see that it works how we want, the message is displayed
letter by letter, as it is received!
