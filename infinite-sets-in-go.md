# Infinite Sets in Go

__Sven Haardiek, 2017-01-07__

If you are too lazy to read, but interested in using finite and infinite sets
in Go, just take a look at the [documentation][godoc] or directly at the
[source][source] of the library which inspired this article.

Some time ago I was working with Go and was missing the concept of set-like
objects like I had in
[Python](https://docs.python.org/3/tutorial/datastructures.html#sets). After
some very very short investigation I found an article about [go maps in
action][gomapsinaction] which was describing how to build such sets very simple
by using maps.

Inspired by that, I was thinking about the concept of sets in programming
languages and realized that all the sets were defined explicitly by their
elements. This is, in general, a good idea, since it makes such sets very fast
and leads to a very easy structure definition.

But I wanted to have an implementation of sets which was more like humans
define sets, especially infinite sets. The goal was not to implement something
that is fast, but easy to use them like real sets.

This article is about the steps that followed.


## Finite Sets

To begin with a set data structure, I was first implementing a struct `set`
defined explicitly by its elements. I followed the [go maps in
action][gomapsinaction] article I mentioned earlier.

```go
type elementSet struct {
    elements map[interface{}]bool
}
```

The most basic method defines which elements are contained in the set.

```go
func (set elementSet) Contains(x interface{}) (bool, error) {
    return set.elements[x], nil
}
```

Why this function returns a tuple of boolean and error although this error is
obviously never set, will be explained later.

Also, I implemented the possibility to count the elements of a set and a helper
function to create an explicit list of a function to make iteration a lot
easier.

```go
func (set elementSet) Cardinality() (uint64, error) {

var number uint64
for _, contained := range set.elements {
    if contained {
        number++
    }
}

    return number, nil
}

func (set elementSet) List() ([]interface{}, error) {

    number, err := set.Cardinality()
    if err != nil {
        return []interface{}{}, err
    }

    var list = make([]interface{}, 0, number)
    for element, contained := range set.elements {
        if contained {
            list = append(list, element)
        }
    }
    return list, nil
}
```

Again here are some errors returned which are always `nil`. And again, this
will be explained later.

## Infinite Sets

After I had implemented the finite sets, I wanted to work with infinite
sets, but defining infinite sets explicitly via the elements is obviously
impossible. Mathematicians which are using pen and paper to define sets would,
for example, define the set of all integers which are divisible by 10 like this

<script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-chtml.js"></script>
<p>$$\{ n \in \Bbb{N} : n \mod 10 = 0 \}$$</p>

So my idea was to define the infinite sets in an equal way and define an
infinite set via a function which describes which elements are contained. So I
defined them as followed

```go
type functionSet struct {
    contains func(interface{}) (bool, error)
}

func (set functionSet) Contains(x interface{}) (bool, error) {
    return set.contains(x)
}
```

I have to admit that this definition is not very practical and very generic,
but this is also the case by the definition of sets in math. So it fits.

Also, you can see that this struct can not only hold infinite sets but also
finite sets as well. There are no real limits to the defining function at all.

## The Interface

So now we have two different definitions of sets. To be able to use both of
them the same way and be able for example to create a union from a finite set
defined by `elementSet` and an infinite set defined by `functionSet`, I created
an interface for these implementations. Also after defining an interface we can
add other definitions of sets very easily as well. It's Go, right?

```go
type Set interface {
    Contains(x interface{}) (bool, error)
    Cardinality() (uint64, error)
    List() ([]interface{}, error)
}
```

Now the problem is that we can not define `Cardinality()` and `List()` for
`functionSet` that easy, since this information are very hard to extract from
the defining function and for real infinite set it would be even impossible to
define a list of elements explicitly. I solved this by implementing some dummy
functions for these which only have `err != nil`.

To make things easier I added a function to the interface to see if you are
working with a set implementing `Cardinality()` and `List()` or not. So,
therefore, the distinction is not any longer between infinite and finite sets,
but countable and uncountable sets.

```go
type Set interface {
    Contains(x interface{}) (bool, error)
    Cardinality() (uint64, error)
    Countable() bool
    List() ([]interface{}, error)
}
```

Now it should be clear why there is returned some errors which are always `nil`
in the previous sections. This is to fit the interface which is written much
more generic than needed for the two existing implementations.

## Adapt Structs to fit the Set Interface

To fulfill the interface `Set` I had to add some methods to the structs
`elementSet` and `functionSet`.

For `elementSet` I only needed to add the `Countable()` function which simply
says `true`.

```go
func (set elementSet) Countable() bool {
    return true
}
```

For `functionSet` I had to add the dummy function I talked about previously for
`Cardinality()` and `List()` and add a `Countable()` function which simply says
`false`.

```go
func (set functionSet) Countable() bool {
    return false
}

func (set functionSet) Cardinality() (uint64, error) {
    return 0, errors.New("Not countable by design")
}

func (set functionSet) List() ([]interface{}, error) {
    return []interface{}{}, errors.New("Not listable by design")
}
```

Now our structs should fit the interface `Set` and we can start defining
functions on the sets.

## Functions on Sets

You can define a lot of functions on the `Set` interface, but I will only
describe a small amount of them. If you want to know more about those
functions, look at the [source][source] or the [documentation][godoc] of the
library.

### Creating Sets

To be able to create sets I added two functions which simply takes either an
explicit list or a function and create an `elementSet` or `functionSet` out of
it.

```go
func CreateFromArray(list []interface{}) Set {
    set := elementSet{make(map[interface{}]bool)}
    for _, element := range list {
        set.elements[element] = true
    }
    return set
}

func CreateFromFunc(f func(interface{}) (bool, error)) Set {
    return functionSet{f}
}
```

### Intersecting Sets

One of the most basic operation on sets is to intersect them with each other.
Due to the possibility to differ countable sets and not countable sets, we are
able to distinct two different situations.

#### One countable Set

Obviously, if one of the sets to intersect is countable the intersection should
also be countable as a subset of a countable set. The following function is to
create such an intersection. Since intersections are commutative, let's assume
that the set `a` is countable.

```go
func countableIntersection(a Set, b Set) (Set, error) {
    // Create new countable set
    newSet := elementSet{make(map[interface{}]bool)}
    // Explicit list of the elements of a
    elements, err := a.List()
    if err != nil {
        return newSet, err
    }
    // Check if the elements are also in b and therefor in the intersection
    for _, element := range elements {
        yes, err := b.Contains(element)
        if err != nil {
            return newSet, err
        }
        if yes {
            newSet.elements[element] = true
        }
    }
    return newSet, nil
}
```

#### No countable Set

If both sets to intersect are not countable the intersection *could* be
implemented as a countable set, but this is very hard to proof and therefore it
is defined as the combination of the defining `Contains(x interface{})`
functions.

```go
func notCountableIntersection(a Set, b Set) (Set, error) {
    // both not countable
    newContains := func(x interface{}) (bool, error) {
        if yes, err := a.Contains(x); err != nil {
            return false, err
        } else if !yes {
            return false, nil
        }
        // If here, x is in a
        if yes, err := b.Contains(x); err != nil {
            return false, err
        } else if !yes {
            return false, nil
        }
        // if here, x is in b
        return true, nil
    }
    return functionSet{newContains}, nil
}
```

Now we only need some glue code to call one of these functions.

```go
func Intersection(a Set, b Set) (Set, error) {
    // a is countable
    if a.Countable() {
        return countableIntersection(a, b)
    }
    // b is countable
    if b.Countable() {
        return countableIntersection(b, a)
    }
    // Both not countable
    return notCountableIntersection(a, b)
}
```

Other functions are following the same chain of thoughts.

## Examples

To show what you can do with this library here some examples.

### Natural numbers

We can now, for example, define the set of natural numbers as a subset of all
strings. It is a little bit like you see it as a human.

```go
import (
    "regexp"

    "github.com/shaardie/set"
)

var NatualNumbers = set.CreateFromFunc(func(x interface{}) (bool, error) {

    // Is a string
    str, ok := x.(string)
    if !ok {
        return false, nil
    }

    // Match what you would "see" as a natural number
    return regexp.MatchString("0|[1-9][0-9]*", str)
})
```

Also, we define a set of very evil numbers.

```go
var evilNumbers = set.CreateFromArray([]interface{}{"1", "3", "21", "999"}
```

Now we can create the subset of the natural numbers which are good by creating
the difference.

```go
goodNumbers, _ := set.Difference(NatualNumbers, evilNumbers)
```


Here the full example

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "regexp"

    "github.com/shaardie/set"
)

func main() {
    var NatualNumbers = set.CreateFromFunc(func(x interface{}) (bool, error) {

        // Is a string
        str, ok := x.(string)
        if !ok {
            return false, nil
        }

        // Match what you would "see" as a natural number
        return regexp.MatchString("0|[1-9][0-9]*", str)
    })

    var evilNumbers = set.CreateFromArray([]interface{}{"1", "3", "21", "999"})

    goodNumbers, _ := set.Difference(NatualNumbers, evilNumbers)

    fmt.Println("Please type a natural number")
    b, _, _ := bufio.NewReader(os.Stdin).ReadLine()
    number := string(b)

    if in, _ := goodNumbers.Contains(number); in {
        fmt.Printf("%v is a good number\n", number)
    } else {
        fmt.Printf("%v is an evil number\n", number)
    }
}
```

## Conclusion

Creating finite and infinite sets in Go was canonical and due to the interface
logic it is easy to expand this further with more complex solutions as well.

Creating other math related packages, for example, a package of different
number set is something I would definitely do. Also creating some kind of
algebra on top of these sets is something very interesting.

[gomapsinaction]: https://blog.golang.org/go-maps-in-action
[godoc]: https://godoc.org/github.com/shaardie/set
[source]: https://github.com/shaardie/set
