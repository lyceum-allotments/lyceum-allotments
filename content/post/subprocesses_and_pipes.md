+++
date = "2016-12-20T22:09:41Z"
draft = true
title = "Python and Pipes Part 5: Subprocesses and Pipes"

+++

The Python [`subprocess`](https://docs.python.org/2/library/subprocess.html)
module (used for starting subprocesses) is one module that provides scope for
heavy usage of pipes. Here we'll look at this module and how you can use pipes
to manipulate the input and output of the spawned subprocess.

A Crash Course in the subprocess Module
---------------------------------------

Let's have a program, for example the Python program detailed below that queries
a person for their name and then echos it with a greeting (note this example is
a Python program, but we can, in principle, use any program)

{{< highlight python >}}
# say_my_name.py
import sys
print "what's your name?"
for name in iter(sys.stdin.readline, ''):
    name = name[:-1]
    if name == "exit":
        break
    print "Well how do you do {0}?".format(name)
    print "what's your name?"
{{< / highlight >}}

This program can be started from within a separate Python process by using the
`subprocess` module, like so:

{{< highlight python >}}
# run_say_my_name.py
import subprocess
import sys

proc = subprocess.Popen(["python", "say_my_name.py"])

while proc.returncode is None:
    proc.poll()
{{< / highlight >}}

`subprocess.Popen` creates a `Popen` object and kicks off a subprocess similar 
to the one that would be started by typing `python say_my_name.py` at a command
prompt. The subsequent `while` loop repeatedly polls the `Popen` object, and
makes sure that the `returncode` attribute is changed from being `None` when the
child process terminates, at which point the mother process will quickly also
terminate.

By default, the `stdin` and `stdout` of the child process are set to be the same
as the `stdin` and `stdout` of the mother, meaning that `say_my_name.py`
operates much as before. Next, we'll work at changing the `stdin` and `stdout`
of the child and exploring what possibilities this uncovers.

Controlling the Input and Output
--------------------------------

`subprocess.Popen` can take two optional named arguments, `stdin` and `stdout`,
that set the pipes that the child process uses as its `stdin` and `stdout`. By
passing the constant `subprocess.PIPE` as either of them you specify that you
want the resultant `Popen` object to have control of child proccess's `stdin`
and/or `stdout`, through the `Popen`'s `stdin` and `stdout` attributes.

In the next example, three names are passed to the `say_my_name.py` child
process before the `EOF` signal is sent to the child's input. The mother process
then waits for the child to finish, before reading whatever output the child
produced and printing it with a small piece of text prepended:

{{< highlight python >}}
# internal_pipe_say_my_name.py
import subprocess
import sys

proc = subprocess.Popen(["python", "say_my_name.py"],
    stdin=subprocess.PIPE, stdout=subprocess.PIPE)


proc.stdin.write("matthew\n")
proc.stdin.write("mark\n")
proc.stdin.write("luke\n")
proc.stdin.close()

while proc.returncode is None:
    proc.poll()

print "I got back from the program this:\n{0}".format(proc.stdout.read())
{{< / highlight >}}

It is easy to see how to extrapolate from this small program to develop an
end-to-end testing suite, starting a program,  passing it in some input and
checking that the output that is received is that expected. But what to if you
want a mix of scripted input and user input, say for a testing program when you
wish to get the test subject into a certain state before allowing interactive
input? That's what we'll look at next.

Mixing Scripted and Interactive Input
-------------------------------------

To expose the subprocess to a certain amount of scripted input, before reverting
to giving the subprocess input from `stdin`, we have to set up the subprocess to
accept input from a pipe, hand it our scripted input and then manually code to
read from the mother process's `stdin` passing whatever we read to the child... 

{{< highlight python >}}
# mixed_input_pipe_say_my_name.py
import subprocess
import sys

proc = subprocess.Popen(["python", "say_my_name.py"], stdin=subprocess.PIPE)

proc.stdin.write("matthew\n")
proc.stdin.write("mark\n")
proc.stdin.write("luke\n")

while proc.returncode is None:
    i = sys.stdin.read(1)
    if i == '':
        proc.stdin.close()
        break
    proc.stdin.write(i)
    proc.poll()

while proc.returncode is None:
    proc.poll()
{{< / highlight >}}

so this code will provide the names 'matthew', 'mark' and 'luke' to the
subprocess before switching to reading every byte from `stdin`. When `sys.stdin`
returns an empty string (''), that indicates that `stdin` has closed so we can
close the `stdin` of the child process and clean up.

Using External Pipes
--------------------

Another interesting trick with subprocesses that you might want to use from time
to time (we'll use it in the next section, in fact) is taking `stdin` and
`stdout` for the subprocess from a couple of external pipes. To do this we'll
first need to create a couple of pipes in our working directory where we will
pipe the input into and read the output out of:

{{< highlight c >}}
mkfifo input_pipe
mkfifo output_pipe
{{< / highlight >}}

Once these two pipes exist, our first stab at using external pipes with a
subprocess takes the following course:

* open the `input_pipe` (for reading) and `output_pipe` (for writing)
* start the subprocess, with `stdin` being `input_pipe` and `stdout` being
  `output_pipe`
* keep polling the subprocess until it returns

in code this looks like this:

{{< highlight python >}}
# external_pipe_say_my_name.py
import subprocess
import sys

with open("input_pipe", "r") as input_pipe:
    with open("output_pipe", "w") as output_pipe:
        proc = subprocess.Popen(["python", "say_my_name.py"],
            stdin=input_pipe, stdout=output_pipe)

        while proc.returncode is None:
            proc.poll()
{{< / highlight >}}

To test this out, we start `external_pipe_say_my_name.py` in one virtual
terminal. In another we pipe some input to `input_pipe`:

{{< highlight c >}}
echo john > input_pipe
{{< / highlight >}}

then, when we read from `output_pipe` in a third, say with `cat output_pipe`,
we retrieve the output of the `say_my_name.py` subprocess.

There is a small problem with this though, that being that once `echo john >
input_pipe` returns and closes the pipe, sending `EOF`, the child process closes
and so does the mother process. What we might like to be able to do is keep
piping names to `input_pipe` and have our mother process keep reading them and
passing them on to its child, without the child finishing.

Achieving this involves a slightly different flow from the one described
previously:

* open the `output_pipe`
* start the subprocess, using `output_pipe` as the output pipe and an internal
  pipe as `stdin`
* keep polling for the end of the child process
* try and open `input_pipe`
* write to the child's `stdin` what you read from the pipe

in code this looks like: 

{{< highlight python >}}
# external_pipe_say_my_name_constant.py
import subprocess

with open("output_pipe", "w") as output_pipe:
    proc = subprocess.Popen(["stdbuf", "-o0", "python", "say_my_name.py"],
        stdin=subprocess.PIPE, stdout=output_pipe)

    while True:
        proc.poll()
        if proc.returncode is not None:
            break
        with open("input_pipe", "r") as input_pipe:
            proc.stdin.write(input_pipe.read())
{{< / highlight >}}

to test this, we start `external_pipe_say_my_name_constant.py` in one virtual
terminal, start reading from `output_pipe` with `cat output_pipe` and in a third
terminal we can write to the input repeatedly with commands such as `echo greg >
input_pipe`, noting that the output is successfully deposited in the
`output_pipe`.

One subtlety that you might have noticed is that the command used to start the
subprocess is `stdbuf -o0 python say_my_name.py` as opposed to the usual
`python say_my_name.py`, what does this mysterious `stdbuf -o0` do?

A quick read of the flipping manual will tell you that this command turns off
buffering for `stdout` stream. This is necessary because by default Linux
buffers `stdout` through a pipe. What this means in our case is that
`say_my_name.py` will receive the names we input, and will process them and
produce an output, but this output will be buffered and not sent on immediately
to the mother process, meaning that when we view the output we see nothing
(at least until the buffer is filled). Try taking away that `stdbuf` argument
from the call to `Popen` and see what happens for yourself (the answer is not
much).

By now we've just about dealt with every permutation of putting stuff in pipes
from various sources and passing it to a sub-process, we're ready for a final
step, a piece de resistance, a program that spawns two subprocesses and allows
the user to send input/read output from both. But that's for the [next
section...](/2017/02/python-and-pipes-part-6-multiple-subprocesses-and-pipes/)
