+++
date = "2017-03-02T23:41:37Z"
draft = false
title = "Python and Pipes Part 2: Introduction to Unix Pipes"
series = "Python and Pipes"
+++

Pipes are one method of interprocess communication in the Unix world, the other
being sockets. The differences are that pipes are more simple to set up and use
but more narrow in scope. Pipes are unidirectional, having one writer and one
reader, and operate on a 'first in first out' (or FIFO) principle -- i.e. the
first bit of data you put in is the first you get out; if I put 'hello world'
into a pipe I will receive a 'h' first and the 'd' last.

Using the Unix shell it's simple to see how pipes work. Just open two virtual
consoles and change directory so that you are in the same directory in both of
them. Make a named pipe by using the command 

{{< highlight c >}}
mkfifo my_pipe
{{< / highlight >}}

which will make a pipe with the name `my_pipe`.

Listing the contents of your directory with `ls -l` you will see `my_pipe`
listed in the directory just like a normal file, only it will have the letter
`p` preceding its permissions:

{{< highlight c >}}
prw-r--r-- 1 username username 0 Nov 17 21:10 my_pipe
{{< / highlight >}}

You can redirect the output of any program to `my_pipe` in just the same way as
you would redirect it to a file, a litle something like this:

{{< highlight c >}}
echo "hello through a pipe" > my_pipe
{{< / highlight >}}

In the other virtual terminal you can access the contents of the pipe in a
similar way to reading the contents of a file:

{{< highlight c >}}
cat my_pipe
{{< / highlight >}}

and you should see "hello through a pipe" printed onto the screen.

Note also that the `echo` command blocked until you read from the pipe. As an
experiment, try things the other way round; `cat my_pipe` will now block until
you write something into the pipe with, say, `echo "hello through a pipe"`.

Hopefully this short introduction will have illustrated to you how simple yet
effective Unix pipes can be, named pipes look and behave in many ways just like
a file and so make interprocess communication as simple as reading/writing from
files. The [next section](/2017/03/python-and-pipes-part-3-pipes-in-python) will go on to discuss how these pipes can be used from
Python.
