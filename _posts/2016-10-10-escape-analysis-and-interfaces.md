---
layout: post
title:  "golang: Escape analysis and interfaces"
date:   2016-10-10 03:30:00
categories: programming
comments: yes
---

Escape what? Well, "escape analysis" is the process by which the
compiler decides if a variable is to be allocated on the stack or on
the heap. In Go (unlike in C) the fact that a variable is locally
declared does *not* mean that it will be stack-allocated. Why? Because
the following is a perfectly valid program in Go (you can
[run it](https://play.golang.org/p/Zt2L1LdsPU) in the playground):

    package main
    
    import "fmt"
    
    var VP *int // a global pointer
    
    func f(vp *int) {
        VP = vp
    }
    
    func g() {
        var v int // a local variable
        v = 40
        f(&v)
        *VP = 42
        fmt.Println(v) // prints 42
    }
    
    func main() {
        g()
        fmt.Println(*VP) //prints 42
    }

Obviously, the variable `v` declared in function `g()` cannot be stack
allocated, since a pointer to it is stored to global `VP` by `f()`. Or
as we say: Variable `v` *escapes* it's local scope; hence **escape
analysis**.

​Now take the following two functions:

    func Ok(f os.File) []byte {
            var x [128]byte
            b := x[:]
            n, _ := f.Read(b)
            r := make([]byte, n)
            copy(r, b[:n])
            return r
    }
    
    func NotOk(c net.Conn) []byte {
            var x [128]byte
            b := x[:]
            n, _ := c.Read(b)
            r := make([]byte, n)
            copy(r, b[:n])
            return r
    }

They look almost identical. There is an unexpected difference,
though. In `Ok()` the array `x` is allocated on the stack, while in
`NotOk()`, `x` is allocated on the heap. That is: `NotOk()` generates
128 bytes of garbage every time it's called; `Ok()` does not.

This can be easily verified by building with:

    go build -gcflags='-m'

Why this happens. If I'm not mistaken: the problem here is that
`net.Conn` is an interface, while `os.File` is not (it's a concrete
type). When the compiler sees a call to a method of net.Conn there is
no way to know (at compile time) if the method's implementation
somehow retains a copy of it's reference argument. I could pass to
`NotOk()` a net.Conn implementation whose `Read()` saves its argument
to a global variable, as shown in the example at the top. Or I could
not. The compiler has no way to know which, so it has to be
conservative.

Sadly, there's nothing that can be done about this, no matter how
clever the compiler becomes. Consider that `NotOk()` could be in a
different package than its callers. As a result: *all* pointers or
reference types passed to methods through interfaces will point to
(reference) heap-allocated objects. Or: There is no way to read from
net.Conn to a stack-allocated buffer.

A bit disappointing, but what can you do...

**UPDATE:**

It turns out, there *is* something that the compiler can do about it,
and it's called "whole program analysis". Whether this could
realistically help in this case, and whether avoiding heap allocation
is worth the cost, is debatable. Some argue that with a very good
allocator and garbage collector, being too focused on stack allocation
is not worth the cost. Naturally, by "very good garbage collector"
they mean a compacting, generational one, and by "good allocator" they
presumably mean a "bump the pointer" allocator. Anyway, I won't get
into this discussion here...

Finally, another example where an interface is causing a heap
allocation, one that would (I think) be easier to avoid by a smart
compiler is the following:

~~~~

type S struct {
    s1 int
}

func (s *S) M1(i int) { s.s1 = i }

type I interface {
    M1(int)
}

func g() {
    var s1 S  // this escapes
    var s2 S  // this does not
        
    f1(&s1)
    f2(&s2)
}

func f1(s I) { s.M1(42) }
func f2(s *S) { s.M1(42) }

~~~~
        
Here the compiler has to decide whether to produce code for `g()` that
allocates `s1` and `s2` on the heap or on the stack. In both cases,
the compiler can see that both `s1` and `s2` are leaked by `g()` to
`f1()` and `f2()` respectively. Now f2 leaks `s` (i.e. `s1`) to method
`S.M1()`, which does not leak `s` anywhere; so `s2` can be safely
allocated on the stack by `g()`. On the other hand, by looking `f1()`
we know that it leaks its `s` argument to *a method*, but by looking
at `f1()` alone we cannot tell to which specific method, and therefore
cannot tell if `s` escapes via that method. A more clever compiler
would observe that `f1()` is called by `g()` and in `g()` we know the
concrete type passed to the interface argument of `f1()` so,
in-effect, we *know* the specific method to which `f1()` leaks it's
argument, and we can see that s1 does not escape this
way. Unfortunately the Go compiler is not currently that clever.

