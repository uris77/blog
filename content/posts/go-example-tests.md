---
title: "Go Notes: Example Tests"
date: 2020-01-20T12:02:11-06:00
draft: false
---

Go's [Example Tests](https://golang.org/pkg/testing/#hdr-Examples) are a great tool for documenting the expectations of our code. They are conveniently integrated with _godocs_, making the docs have more context and actual runnable examples. These tests provide a quick overview of what the code is supposed to be doing in a manner that is easily digestible for the reader without needing to understand the inner workings.

## What are Example Tests?
In Example Tests, we print out the result of a function that we want to test, and we indicate the expected output as comments:

```go
func ExampleTest() {
    fmt.Println("Hello World!")
    // Output:
    // Hello World!
}
```

If we can't determine the order of the output of a test, we can use `Unordered output` instead of `Output`:

```go
func ExampleUnorderedTest() {
    unordered := afunc()
    fmt.Println(unordered)

    // Unordered output:
    // d
    // c
    // a
    // b

}

```

## Naming Conventions
The Go community seems to have settled on two naming conventions for the example file names. Both conventions can be found in the standard library. The first one follows the pattern `xxx_example_test.go`. This can bee seen in the example tests for `stringer`: [https://github.com/golang/go/blob/master/src/fmt/stringer_example_test.go](https://github.com/golang/go/blob/master/src/fmt/stringer_example_test.go).

The second pattern is `example_xxx_test.go`. This pattern is prevalent in the `sort` package of the standard library: [https://github.com/golang/go/tree/master/src/sort](https://github.com/golang/go/tree/master/src/sort).


 # Summary
Example tests are a useful tool for documenting our code. They are kept relevant because they are run as tests and are embedded in the "godocs" for easy consumption.