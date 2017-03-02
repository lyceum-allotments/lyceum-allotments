+++
date = "2017-03-02T23:46:37Z"
draft = false
title = "Python and Pipes Part 6: Multiple Subprocesses and Pipes"
series = "Python and Pipes"
+++

In the [previous
section](/2017/03/python-and-pipes-part-5-subprocesses-and-pipes/)
we explored start a subprocess and controlling its input and output via
pipes. In this section we'll do the same, but this time for two sub-processes.
A use for this, and the original reason I first developed this, was for testing
a client and server. Basically, I wanted a program to start up the client and
the server, to provide a set of pre-scripted commands to each to get them in a
certain state, and then a way of providing my own custom commands, to do some
interactive testing.

What we aim to end up with is a program that starts up two sub-processes, let's
call them `A` and `B`, and connects to two named pipes in the file system. From
these two pipes `A` and `B` will read their respective inputs. Output from both
of the processes will be printed on `stdout`, but to enable us to differentiate
which output comes from which process we'll prepend output from process `A` with
an `A:` and from process `B` with a `B:`.

The Two Sub-Processes
---------------------

As an illustration of what can be achieved, the two sub-processes that we are
going to spawn will be kept very simple, doing nothing more than printing a
prompt, reading a line of input, echoing that back and prompting for more input: 

{{< highlight python >}}
# proc_a.py

import sys
print "what should proc A say?"
for name in iter(sys.stdin.readline, ''):
    name = name[:-1]
    if name == "exit":
        break
    print "Proc A says, \"{0}\"".format(name)
    print "what should proc A say?"
{{< / highlight >}}


{{< highlight python >}}
# proc_b.py

import sys
print "what should proc B say?"
for name in iter(sys.stdin.readline, ''):
    name = name[:-1]
    if name == "exit":
        break
    print "Proc B says, \"{0}\"".format(name)
    print "what should proc B say?"
{{< / highlight >}}

A program to start both of these sub-processes and run them until
either of them, or the mother process, finishes, looks pretty similar
to the stuff we were doing with sub-processes in the previous section:
all we have to do is two start two subprocesses rather than one and make
sure we poll and check two return codes rather than one:

{{< highlight python >}}
# two_subprocesses.py

import subprocess

# start both `proc_a.py` and `proc_b.py`
proc_a = subprocess.Popen(["stdbuf", "-o0", "python2", "proc_a.py"], stdin=subprocess.PIPE,
    stdout=subprocess.PIPE)
proc_b = subprocess.Popen(["stdbuf", "-o0", "python2", "proc_b.py"], stdin=subprocess.PIPE,
    stdout=subprocess.PIPE)

while True:
    # check if either sub-process has finished
    proc_a.poll()
    proc_b.poll()

    if proc_a.returncode is not None or proc_b.returncode is not None:
        break
{{< / highlight >}}

Note that both our subprocesses are opened with their `stdin`s and `stdout`s
from internal pipes. This is necessary because we want to have control over
their input and outputs since we have two inputs and outputs and only one mother
process `stdin` and `stdout`. In order to avoid the two outputs becoming
interleaved and two know what input to send to which child process we're going
to have to do something clever...

Non-blocking Outputs: The Power of Threads
------------------------------------------

We'll start with output. Our basic strategy is going to be read a line from a
subprocess and pass through to the mother process's `stdout`, prepended with an
`A:` if it came from procedure `A` and a `B:` if it came from procedure `B`.
However, there's a snag, and that snag is that the `read()` method of Python's
file object is blocking: if our mother process is waiting to read from the
`stdout` of `B` and no output is produced from `B`, then our mother process will
just wait as long as it takes for some output to turn up. If no end of
interesting output is coming from `A` it makes no odds, it just has to wait.

What we need is a way of doing two things at once: reading from `A` and `B`
simultaneously and putting a line from `A` or `B` to the mother process's
`stdout` whenever one appears. Luckily, Python has a way of doing two things at
once: the [`threading`](https://docs.python.org/2/library/threading.html)
module.

A thread, if you didn't know, is a lightweight separate thread of execution that
shares the same memory space as the spawning thread. You make one in Python by
calling the `Thread` constructor with a call something like this:
`threading.Thread(target=function, args=(arg1, arg2))`. The argument `target` is
the function that the thread will start at when the thread is started, and
`args` is a tuple containing the arguments that will be passed to this function.

A thread is started by calling its `start()` method, at which point the function
`target` will be called in a separate thread of execution, running in parallel
to the spawning thread, and any other threads.

To get an idea of how threads work, take a look at this example program:

{{< highlight python >}}
# simple_threading_eg.py

import threading
import random
import time

def test_fn(a, b):
    # pause a random number of seconds before doing anything else
    time.sleep(random.random())

    print "{0} {1}".format(a, b)


# make 4 threads, which will all end up calling the same function
# (but will pass different arguments to it)
thread_1 = threading.Thread(target=test_fn, args=(1, "john"))
thread_2 = threading.Thread(target=test_fn, args=(2, "paul"))
thread_3 = threading.Thread(target=test_fn, args=(3, "george"))
thread_4 = threading.Thread(target=test_fn, args=(4, "ringo"))

# making a thread a `daemon` means that when the main process
# ends the thread will end too
thread_1.daemon = True
thread_2.daemon = True
thread_3.daemon = True
thread_4.daemon = True

# start the threads running
thread_1.start()
thread_2.start()
thread_3.start()
thread_4.start()

# wait for all the child threads to terminate before ending
thread_1.join()
thread_2.join()
thread_3.join()
thread_4.join()
{{< / highlight >}}

Communicating Between Threads
-----------------------------

This example creates four threads, all calling the same simple function with
different arguments. The `target` function is set up to sleep for a random
number of seconds before printing the arguments it was passed.

When one of our threads reading a process's output gets some output, we need to
pass that output back to our main thread in order to do some post processing and
print it. How do we do that with threads? 

There is a Python module called
[`Queue`](https://docs.python.org/2/library/queue.html#Queue.Queue) that
implements a thread-safe queue, one thread can put objects on a queue and
another thread can pop objects off, safe in the knowledge that these things are
safe despite the fact that they may be happening simultaneously.

A queue is created with:

{{< highlight python >}}
import Queue
q = Queue.Queue()
{{< / highlight >}}

objects (in this case the variable `a`) are placed on it with:

{{< highlight python >}}
a = 5
q.put(a)
{{< / highlight >}}

and read off in a non-blocking way with:

{{< highlight python >}}
try:
    b = q.get(False)
    # b will now have the value '5' 
except Queue.Empty:
    pass
{{< / highlight >}}

A simple example is given below, building on the simple introduction to threads
that was introduced before. In this case, the string that the `test_fn` produces
after a random period of time is put on a queue. The main thread has an infinite
loop that keeps checking if anything is on the queue, and it is the main thread
which does the printing if something is found:

{{< highlight python >}}
# simple_threading_queue_eg.py

import threading
import random
import time
import Queue

def test_fn(a, b, q):
    # pause a random number of seconds before doing anything else
    time.sleep(random.random())

    # put a message on the queue
    q.put("{0} {1}".format(a, b))


q = Queue.Queue()

# make 4 threads, which will all end up calling the same function
# (but will pass different arguments to it)
thread_1 = threading.Thread(target=test_fn, args=(1, "john", q))
thread_2 = threading.Thread(target=test_fn, args=(2, "paul", q))
thread_3 = threading.Thread(target=test_fn, args=(3, "george", q))
thread_4 = threading.Thread(target=test_fn, args=(4, "ringo", q))

# making a thread a `daemon` means that when the main process
# ends the thread will end too
thread_1.daemon = True
thread_2.daemon = True
thread_3.daemon = True
thread_4.daemon = True

# start the threads running
thread_1.start()
thread_2.start()
thread_3.start()
thread_4.start()

while True:
    try:
        # if there is any message on the queue, print it.
        # if the queue is empty, the exception will be caught
        # and the queue polled again in a moment
        print q.get(False)
    except Queue.Empty:
        pass
{{< / highlight >}}

Implementing Non-Blocking Output
--------------------------------

That last example came very close to having the functionality we desired of our
non-blocking output. We now need to take the principles explored in
`simple_threading_queue_eg.py` and apply them to putting output from process A
and B's `stdout` onto a queue, rather than just any old string.

We want the target function of our output reading threads to keep attempting to
read from their target pipes, whenever they manage to read a whole line of
something we want them to put this line onto a queue:

{{< highlight python >}}
def read_output(pipe, q):
    """reads output from `pipe`, when line has been read, puts
line on Queue `q`"""

    while True:
        l = pipe.readline()
        q.put(l)

{{< / highlight >}}

A thread that reads from the `stdout` of procedure `A` and put any output it
finds into a queue, `pa_q` can be started like this:

{{< highlight python >}}
# queue for storing output lines
pa_q = Queue.Queue()

# start a pair of thread to read output from procedures A
pa_t = threading.Thread(target=read_output, args=(proc_a.stdout, pa_q))
pa_t.daemon = True
pa_t.start()
{{< / highlight >}}

with a similar sequence needed for procedure `B`.

With these threads busily running in the background, putting any output they get
onto a queue named `pa_q` for process `A`, or `pb_q` for `B`, we want our main
thread to loop, periodically checking the queues to see if process `A` or `B` has
produced any output. Upon finding some, we just prepend the letter of the
producing process and print the message:

{{< highlight python >}}
while True:
    # ...

    # write output from procedure A (if there is any)
    try:
        l = pa_q.get(False)
        sys.stdout.write("A: ")
        sys.stdout.write(l)
    except Queue.Empty:
        pass

    # write output from procedure B (if there is any)
    try:
        l = pb_q.get(False)
        sys.stdout.write("B: ")
        sys.stdout.write(l)
    except Queue.Empty:
        pass
{{< / highlight >}}

And that should be it! Putting this all together into a working script:

{{< highlight python >}}
# two_subprocesses_with_output.py

import subprocess
import threading
import sys
import Queue


def read_output(pipe, q):
    """reads output from `pipe`, when line has been read, puts
line on Queue `q`"""

    while True:
        l = pipe.readline()
        q.put(l)

# start both `proc_a.py` and `proc_b.py`
proc_a = subprocess.Popen(["stdbuf", "-o0", "python2", "proc_a.py"],
    stdin=subprocess.PIPE, stdout=subprocess.PIPE)
proc_b = subprocess.Popen(["stdbuf", "-o0", "python2", "proc_b.py"],
    stdin=subprocess.PIPE, stdout=subprocess.PIPE)

# queues for storing output lines
pa_q = Queue.Queue()
pb_q = Queue.Queue()

# start a pair of threads to read output from procedures A and B
pa_t = threading.Thread(target=read_output, args=(proc_a.stdout, pa_q))
pb_t = threading.Thread(target=read_output, args=(proc_b.stdout, pb_q))
pa_t.daemon = True
pb_t.daemon = True
pa_t.start()
pb_t.start()

while True:
    # check if either sub-process has finished
    proc_a.poll()
    proc_b.poll()

    if proc_a.returncode is not None or proc_b.returncode is not None:
        break

    # write output from procedure A (if there is any)
    try:
        l = pa_q.get(False)
        sys.stdout.write("A: ")
        sys.stdout.write(l)
    except Queue.Empty:
        pass

    # write output from procedure B (if there is any)
    try:
        l = pb_q.get(False)
        sys.stdout.write("B: ")
        sys.stdout.write(l)
    except Queue.Empty:
        pass
{{< / highlight >}}

Running this will result in the output of both sub procedures being multiplexed
and printed out together:

{{< highlight c >}}
A: what should proc A say?
B: what should proc B say?
{{< / highlight >}}

Non-blocking Input
------------------

The way we want input for our test harness to work is like this, we want to have
two named pipes in the file system, `proc_a_input` and `proc_b_input`, for which
attempts are constantly made to open and read from them. Whenever anything is
read from either it can be passed directly to the `stdin` of the appropriate
process.

This case is actually a little simpler than the output case, since we don't have
to make any communication back to the main thread.

The target function of our input threads will look like what we had in
[external_pipe_say_my_name_constant.py](/2017/03/python-and-pipes-part-5-subprocesses-and-pipes#external_pipe_say_my_name_constant),
i.e.:

{{< highlight python >}}
def read_input(write_pipe, in_pipe_name):
    """reads input from a pipe with name `read_pipe_name`,
writing this input straight into `write_pipe`"""
    while True:
        with open(in_pipe_name, "r") as f:
            write_pipe.write(f.read())
{{< / highlight >}}

where `write_pipe` will be the `stdin` of our processes `A` and `B`, and
`in_pipe_name` will be the name of the external pipes in our file system,
`proc_a_input` and `proc_b_input`. For procedure `A`, for example:

{{< highlight python >}}
# start a thread to read input into procedure A
pa_input_thread = threading.Thread(target=read_input, args=(proc_a.stdin, "proc_a_input"))
pa_input_thread.daemon = True
pa_input_thread.start()
{{< / highlight >}}

With everything mashed together you'll get a program like this:

{{< highlight python >}}
# two_subprocesses_with_output_and_input.py

import subprocess
import threading
import sys
import Queue


def read_output(pipe, q):
    """reads output from `pipe`, when line has been read, puts
line on Queue `q`"""

    while True:
        l = pipe.readline()
        q.put(l)

def read_input(write_pipe, in_pipe_name):
    """reads input from a pipe with name `read_pipe_name`,
writing this input straight into `write_pipe`"""
    while True:
        with open(in_pipe_name, "r") as f:
            write_pipe.write(f.read())

# start both `proc_a.py` and `proc_b.py`
proc_a = subprocess.Popen(["stdbuf", "-o0", "python2", "proc_a.py"],
    stdin=subprocess.PIPE, stdout=subprocess.PIPE)
proc_b = subprocess.Popen(["stdbuf", "-o0", "python2", "proc_b.py"],
    stdin=subprocess.PIPE, stdout=subprocess.PIPE)

# lists for storing the lines of output generated
pa_line_buffer = [] 
pb_line_buffer = [] 

# queues for storing output lines
pa_q = Queue.Queue()
pb_q = Queue.Queue()

# start a pair of threads to read output from procedures A and B
pa_t = threading.Thread(target=read_output, args=(proc_a.stdout, pa_q))
pb_t = threading.Thread(target=read_output, args=(proc_b.stdout, pb_q))
pa_t.daemon = True
pb_t.daemon = True
pa_t.start()
pb_t.start()

# start a pair of threads to read input into procedures A and B
pa_input_thread = threading.Thread(target=read_input, args=(proc_a.stdin, "proc_a_input"))
pb_input_thread = threading.Thread(target=read_input, args=(proc_b.stdin, "proc_b_input"))
pa_input_thread.daemon = True
pb_input_thread.daemon = True
pa_input_thread.start()
pb_input_thread.start()

while True:
    # check if either sub-process has finished
    proc_a.poll()
    proc_b.poll()

    if proc_a.returncode is not None or proc_b.returncode is not None:
        break

    # write output from procedure A (if there is any)
    try:
        l = pa_q.get(False)
        sys.stdout.write("A: ")
        sys.stdout.write(l)
    except Queue.Empty:
        pass

    # write output from procedure B (if there is any)
    try:
        l = pb_q.get(False)
        sys.stdout.write("B: ")
        sys.stdout.write(l)
    except Queue.Empty:
        pass
{{< / highlight >}}

Running the Suite
-----------------

Now all that is left to do is give the suite a test-drive an check that it
works. In the same directory that you're going to be running the script make
sure you've got two named pipes `proc_a_input` and `proc_b_input`:

{{< highlight c >}}
mkfifo proc_a_input
mkfifo proc_b_input
{{< / highlight >}}

then you can run `two_subprocesses_with_output_and_input.py`. In another
terminal, by piping input into `proc_a_input` or `proc_b_input` you should see
the consequences of that input reflected in the output of the suite, for example:

{{< highlight c >}}
    echo hi there > proc_a_input
{{< / highlight >}}

in another terminal should give you the output:

{{< highlight c >}}
A: what should proc A say?
B: what should proc B say?
A: Proc A says, "hi there"
A: what should proc A say?
{{< / highlight >}}

and following this with:

{{< highlight c >}}
    echo hi from b > proc_b_input
{{< / highlight >}}

will give you the output:

{{< highlight c >}}
A: what should proc A say?
B: what should proc B say?
A: Proc A says, "hi there"
A: what should proc A say?
B: Proc B says, "hi from b"
B: what should proc B say?
{{< / highlight >}}

Naturally this is a simple example, but if you are trying to test a
server-client architecture it can be very powerful and labour saving. 
Since all the output goes through the main thread, you can check that your
server and client are in appropriate states before handing interactive control
over to the user.

As an illustration, I needed to prompt my client to send a couple of messages to
the server before I wanted to do interactive testing. This was a trivial task of
sending the appropriate messages through the client's `stdin` and checking for
the correct responses from the server's `stdout` before starting the threaded
input and the event loop of the main process.

I hope this exploration of Python pipes and subprocesses will save you a similar
amount of time in the future!
