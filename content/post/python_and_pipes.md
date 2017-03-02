+++
date = "2017-03-02T23:40:37Z"
draft = false
title = "Python and Pipes"
series = "Python and Pipes"
+++

I've been working a bit with pipes in Python, using the
[`subprocess`](https://docs.python.org/2/library/subprocess.html) module to
kick off a couple of processes, a client and a server, and using pipes to
redirect their input and output in such a way as to test them. I thought it
might be an idea to write about using pipes primarily from a Python viewpoint,
with a view especially to clarifying things like blocking and buffering.

In the course of this tutorial we will develop a testing tool for a pair of
programs. This tool will be able to start both programs, issue some commands to
both of them and then source its input from named pipes, allowing the user to
enter their own commands as and when they see fit. The output will be provided
via `stdout` and will be the multiplexed output of both prgrams, prefixed with
an appropriate identifying prefix. A typical use case of this testing tool (and
what I am using it for) is testing a client and server, by which I need to fire
up both client and server, automate the entry of some configuration commands
and then have manual control to enter more commands to both client and server.

The tutorial is broken into the following pieces:

* [**Introduction to Unix Pipes**](/2017/03/python-and-pipes-part-2-introduction-to-unix-pipes/)
-- a brief introduction to Unix named pipes in general, and how to use them
* [**Pipes in Python**](/2017/03/python-and-pipes-part-3-pipes-in-python/)
-- Python is used to write and read from pipes
* [**On the Buffers**](/2017/03/python-and-pipes-part-4-on-the-buffers/)
-- looking at the effect of buffering in reading and writing from pipes, writing
a program that sends a message letter by letter and line by line
* [**Subrocesses and pipes**](/2017/03/python-and-pipes-part-5-subprocesses-and-pipes/)
-- where a subprocess is spawned in Python and the input and output redirected
and manipulated to our heart's content
* [**Multiple subprocesses and pipes**](/2017/03/python-and-pipes-part-6-multiple-subprocesses-and-pipes/)
-- the fully-fledged testing tool is developed, a program that spawns two
subprocesses, and redirects the inputs and outputs to four pipes 
