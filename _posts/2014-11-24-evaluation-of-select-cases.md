---
layout: post
title:  "Evaluation of Select Cases"
date:   2014-11-24 21:01:59
categories: golang
comments: yes
---

Assume you have a goroutine and two channels: *cin* and *cout*. The
goroutine receives stuff from *cin* and pushes them to a queue. It
then pops stuff from the queue and sends them to *cout*. The queue
could be there for dynamic buffering, or for reordering (prioritizing
request), etc. This is a pretty common scenario. One would imagine
that you could write this along these lines:

    var out chan stuff
    out = nil
    for {
        select {
        case v := <-cin:
            q.Push(v)
            out = cout
        case out <- q.Pop():
            if q.Empty() {
                out = nil
            }
        }
    }

Unfortunatelly, though certainly elegant, this *doesn't work*. The
problem is that, for send-cases, the right-hand-side of the channel
operation is evaluated upon entering the select statement, *before*
the case is actually selected (and even if it's not). Copying from the
[language spec](http://golang.org/ref/spec#Select_statements):

> For all the cases in the statement, the channel operands of receive
> operations and the channel **and right-hand-side expressions of send
> statements** are evaluated exactly once, in source order, upon
> entering the "select" statement. The result is a set of channels to
> receive from or send to, **and the corresponding values to send**.

I admit that this caught me by surprise. I would expect the
right-hand-side (RHS) part of send statements to be evaluated *only*
when the case was selected (i.e. to be evaluated lazily). This would
allow someone to write code like the above. Why did the language
designers decide to have the RHS of send statements evaluated so
early?

Turns out, the answer is rather obvious (and has been [discussed on
golang-nuts](https://groups.google.com/d/topic/golang-nuts/qZO4kldyWak/discussion)
some time ago). Think of what would happen if someone did:

    case cout <- TakesLotsOfTimeToCompute():

or:

    case cout <- BlockingCall():

or maybe even:

    case cout <- (<-foo)

If the RHS of the send operation was evaluated *after* the case was
selected, the channel would have to be locked, and remain locked
(waiting to send to), until the lengthy function completes, the
blocking call returns, or the receive from ``foo`` can proceed. It is
very inefficient to keep a channel locked for so long (arbitrarily
long), so the language designers decided to have the values to be sent
computed *before* the operation is selected. This way we can be
certain that when (and if) a case is selected, the channel will not
have to be kept locked for as long as the RHS evaluation is performed.

Where does this leave us? Well, the code above doesn't work and must
be rewritten along these lines:

    var out chan stuff
    var vo stuff
    out = nil
    for {
        select {
        case v := <-cin:
            if out == nil {
                vo = v
                out = cout
            } else {
                q.Push(v)
            }
        case out <- vo:
            if q.Empty() {
                out = nil
                break
            }
            vo = q.Pop()
        }
    }

It is certainly uglier, but what can you do? Everything has a cost!
