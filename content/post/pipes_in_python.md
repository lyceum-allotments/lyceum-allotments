+++
date = "2017-03-02T23:42:37Z"
draft = false
title = "Python and Pipes Part 3: Pipes in Python"
series = "Python and Pipes"
+++

In the [previous
section](/2017/03/python-and-pipes-part-2-introduction-to-unix-pipes/) it was
shown how pipes were represented in the operating system in Unix, and how they
were written and read from in a way that was very much analagous to how ordinary
files are. This is one of the major insights of the Unix family of operating
systems, they make explicit the analogies between interprocess communication and
ordinary input and output -- if a program is made to operate based on the input
of plain text and produce plain text as an output it does not matter about the
origins of that input, be it a human with a keyboard, another program or even
input travelling from across the world over the internet.

It should come as no surprise that Python continues in this happy tradition,
treating named pipes in much the same way as files.

Writing to Pipes in Python
--------------------------

Here is a Python program for writing a short string of text to a file:

{{< highlight python >}}
with open("my_file", "w") as f:
    print "have opened file, commencing writing...."
    f.write("hello to a file\n")
{{< / highlight >}}

and here is how we would read the contents of that file from a virtual console:

{{< highlight c >}}
cat my_file
{{< / highlight >}}

The equivalent program for writing to our pipe `my_pipe` is fundamentally
exactly the same,

{{< highlight python >}}
with open("my_pipe", "w") as f:
    print "have opened pipe, commencing writing...."
    f.write("hello through a pipe\n")
{{< / highlight >}}

a quick

{{< highlight c >}}
cat my_pipe
{{< / highlight >}}

will read the contents of the pipe in a way exactly analagous to a file.

One thing worth noting is that the message, "have opened pipe, commencing
writing" does not get displayed until we open the other end of the pipe for
reading -- the call to `open` in Python is blocking until the pipe has got an
end to be written to.

Reading from Pipes in Python
----------------------------

Reading from pipes is also exactly analagous to reading from a pipe:

<span id="read_from_pipe">
{{< highlight python >}}
# read_from_pipe.py
with open("my_pipe", "r", 0) as f:
    print "have opened pipe, commencing reading...."
    for l in f:
        print l,
{{< / highlight >}}
</span>

by running `write_to_pipe.py` in one virtual console and `read_from_pipe.py` in
another we recover the same behaviour as we observed in using `echo` and `cat`
before. Indeed we can run the read and write and the opposite order, and see
that this time it's the call to `open` in the read program that blocks, until
the write program is ran and opens the other end of the pipe.

In the next section we look a bit more at the subtleties around these
operations, specifically the subtleties around [buffering](/2017/03/python-and-pipes-part-4-on-the-buffers).
