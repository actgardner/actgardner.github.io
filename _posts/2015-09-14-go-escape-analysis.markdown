---
layout: post
title:  "Golang Escape Analysis"
date:   2015-09-20 00:00:00
categories: golang garbage collection gc escape analysis
---

Anyone who's written or heard of Go is probably aware that it's garbage-collected. However, it's worth knowing that every variable in Go is not allocated on the heap, and doesn't need to be garbage collected. The Go compiler is smart enough to perform escape analysis to determine whether a variable can be allocated on the stack, or whether it must be heap-allocated - see (this post)[https://scvalex.net/posts/29/] for a quick introduction to escape analysis as performed by the Go compiler. There are many (documented, pathological cases as of Go 1.5)[https://docs.google.com/document/d/1CxgUBPlx9iJzkz9JWkb6tIpTe5q32QDmz8l0BouG0Cw/preview] where variables may be needlessly heap allocated.

If you've (profiled heap usage)[http://blog.golang.org/profiling-go-programs] and identified hot spots, you may be able to eliminate some of them by working with the compiler to heap allocate variables. Passing the argument `-gcflags '-m'` to any `go` command (`build`, `test`, `install`) makes the compiler print detailed escape analysis output. Let's walk through a few examples.

Let's start simple:
```
package main

type S struct {}

func main() {
  var x S
  _ = identity(x)
}

func identity(x S) S {
  return x
}
```

You'll have to build this with `go run -gcflags '-m -l'` - the `-l` flag prevents the method `identity` from being inlined (that's a topic for another time). The output is: nothing! Go uses pass-by-value semantics, so the `x` variable from `main` will always be copied into the stack of `identity`. In general code without references always uses stack allocation, trivially. There's no escape analysis to do. Let's try something harder:

```
package main

type S struct {}

func main() {
  var x S
  y := &x
  _ = *identity(y)
}

func identity(z *S) *S {
  return z
}
```

And the output:

```
./escape.go:11: leaking param: z to result ~r1
./escape.go:7: main &x does not escape
```

The first line shows that the variable "flows through": the input variable is returned as an output.  When `identity` returns `z` can be copied onto the stack of the calling function. The second line shows that the reference to `x` we take on line 7 never escapes `main`. No variables are allocated on the heap. 

A third experiment:

```
package main

type S struct {}

func main() {
  var x S
  _ = *ref(x)
}

func ref(z S) *S {
  return &z
}
```

And the output:

```
./escape.go:10: moved to heap: z
./escape.go:11: &z escapes to heap
```

Now there's some escaping going on. Since we return a reference to `z`, it can't be allocated on the stack - where would the reference point when `identity` returns? Instead it escapes to the heap. This is a toy example - `ref` would be inlined by the compiler if we weren't stopping it - but it demonstrates how to interpret some `-m` output.

Finally, a slightly more insidious example: 

```
package main

type S struct {
  M *int
}

func main() {
  var x S
  var i int
  ref(&i, &x)
}

func ref(y *int, z *S) {
  z.M = y
}
```

Output:

```
./escape.go:13: leaking param: y
./escape.go:13: ref z does not escape
./escape.go:9: moved to heap: i
./escape.go:10: &i escapes to heap
./escape.go:10: main &x does not escape
```

This is an example of "Flow through function arguments" from the document linked above: Go can only handle arguments which flow through from input to output, not input to other input.  
