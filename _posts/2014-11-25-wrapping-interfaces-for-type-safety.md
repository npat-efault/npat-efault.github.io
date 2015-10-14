---
layout: post
title:  "Wrapping Interfaces for Type Safety"
date:   2014-11-25 21:01:59
categories: golang
---

Go(lang) has no support for generics. This introduces a dilemma when
you want to code container data structures since there is no way to
define something like `Tree<T>` and have a type-parametric tree
implementation. To get around this problem there are three possible
practical solutions (that I can think-of):

1. If the implementation of the data structure is short and trivial,
you can open-code it.

2. You can copy-paste the implementation and replace every occurrence
of the type (along with the operations on it) with the one you want. A
more elaborate version of this is to have the copying and replacement
performed mechanically by some custom code-generation method or
templating scheme.

3. You can implement the data structure to support any type that
satisfies a specific interface. This interface could be just
`interface{}` (i.e. any type at all) if you don't need the type to
support any operations, or an interface that includes the operations
you want the type to support.

We will talk about this third approach here. Two are its main
drawbacks:

(a) It introduces boxing and unboxing overheads. These overheads are
*not* more than what you'd get from *any* dynamic, garbage collected,
language, or from any language (dynamic or statically typed) that only
supports boxed types. It's the fact that Go(lang) supports, and makes
frequent use of, "plain", unboxed types that makes some see the
difference as dramatic. Anyway, these overheads may of may-not be
important (or acceptable) depending on the specific application (they
tend, though, to get overemphasized).

(b) It makes you surrender some of the language's static-type
safety. This is the issue we will focus on for the rest of this
discussion.

Imagine you have a tree implementation (binary search tree, balanced
tree, AVL tree, whatever) that provides an API like this:

    package tree
  
    // Interface must be implemented by elements (values) inserted in
    // the tree (i.e. tree elements must be comparable to each other). 
    type Interface interface {
        // The Less method returns true if the receiver is less 
        // than the argument, false otherwise.
        Less(Interface) bool
  
        // The Equal method returns true if the receiver is equal
        // to the argument, false otherwise.
        Equal(Interface) bool
    }
  
    // Creates and returns a new tree.
    func New() *Tree
  
    // Insert adds v to the tree. Returns false if v is already in 
    // the tree (insertion failed)
    func (t *Tree) Insert(v Interface) bool
    
    // Remove removes v from the tree. Returns true if v was removed,
    // false if there is no v in the tree.
    func (t *Tree) Remove(v Interface) bool
  
    // Lookup searches the tree for a node with value equal to s 
    // and returns that node's value. Returns ok == false if no 
    // such node is found (ok == true if it is).
    func (t *Tree) Lookup(s Interface) (v Interface, ok bool)

Don't fret too much over the specific API as it's just
demonstrative. Assume you want to put to the tree pointers to values
of this type:

    type Thing struct {
        Id int
        Val string
    }

You could do it like this. First you create your tree:
  
    thingsTree := tree.New()

Then you insert values to it:

    ok := thingsTree.Insert(&Thing{Id: 10, Val: "Thing with id 10"})
    if ! ok {
        fmt.Printf("Thing already in the tree")
        // Do what's necessary
    }

And you lookup values like this:

    v, found := thingsTree.Lookup(&Thing{id: 10})
    if ! found {
        fmt.Printf("Thing not found")
    } else {
        fmt.Printf("Found thing! Id: %d, Val: %s", v.Id, v.Val)
    }

The problem with this approach is that, if you use several trees (say
one for things, one for persons, and so on), somewhere in you code,
when you are tired and sleep deprived, you might try to insert a
person to the things-tree or vice-versa. No-one will stop you until
the code actually runs and reaches the offending call (and for some
data-structures, not for our tree though, maybe not even then)

    thingsTree.Insert(&Person{Name: "a name", Phone: 4242424})

Assuming `Person` implements `tree.Interface` (which it would, since
there is a `personsTree` somewhere in your code), the compiler won't
stop you. We seem to have defeated the protections offered by the
static type-system of the language, which is bad.

The thing is, you can easily restore (much of) the sanity, world
order, and the cozy feeling of static type safety, with... a little
extra code, all in one place. When you define your things-tree (or
your persons-tree, or whatever) you can go the extra mile and *wrap
the generic tree methods in type-specific ones*, like this:

    type ThingsTree struct {
        tr *tree.Tree
    }
  
    func NewThingsTree() *ThingsTree {
         t := ThingsTree{}
         t.tr = tree.New()
  	   return t
    }
  
    func (t *ThingsTree) Insert(thing *Thing) ok bool { 
        return t.tr.Insert(thing) 
    }
  
    func (t *ThingsTree) Remove(thing *Thing) ok bool { 
        return t.tr.Remove(thing) 
    }
  
    func (t *ThingsTree) Lookup(k *Thing) (t *Thing, ok bool) { 
        v, ok := t.tr.Lookup(k)
        if ! ok {
           return nil, false
        }
        return v.(*Thing) 
    }

This way you reduce the "surface area" of type unsafe code to these
few and rather trivial functions. The remaining of your code accesses
the data-structure through these type-safe wrappers. Yes, it *does*
take some more typing, but the extra code is trivial (it's hard to get
it wrong or introduce bugs) and for the cases where you should really
worry about messing things up (large complex code-bases with many
different object types touched by many developers) it practically
amounts to very little additional effort.

Is it bullet-proof? For code outside the package defining the
wrapper-type, it is. For code in the same package, one could
conceivably do `thingsTree.tr.Insert()` (instead of
`thingsTree.Insert()`) and mess things up. Again, the more complex and
large your code-base gets, the more you need the protection.

It's *not* an ideal solution (especially seen with the eyes of a
language purist), but, practically, it's a *pretty decent
compromise*. It's up to you to decide if you need the extra protection
(at the cost of extra verbosity, complexity, effort) according to the
needs of your project.

After all, life is a set of compromises.
