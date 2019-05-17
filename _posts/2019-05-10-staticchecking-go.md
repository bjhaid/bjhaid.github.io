---
layout: post
title: Static Checking Go
modified: 2019-05-10
published: true
comments: true
tags: [golang, go, kubernetes, vim, staticcheck, static, check]
---

I like Golang for being a very opinionated language, _variables declared and not
used_ will result in code not compiling, same with unused imports etc. To give a
quick example, the code below will not compile:

```go
package main
import "fmt"
func main() {
  fmt.Println("Does not compile")
  var willNotCompile int
}
```

An attempt to run it gives:

```bash
$> go run will_not_compile.go
# command-line-arguments
./will_not_compile.go:4:6: willNotCompile declared and not used
```

which is great as golang does not want us declaring variables we will not use,
however if we move the `willNotCompile` variable into the [file
block](https://golang.org/ref/spec#Blocks):

```go
package main
import "fmt"
var willNotCompile int
func main() {
  fmt.Println("Compiles!?")
}
```

and run it:

```bash
$> go run will_not_compile_file_scope.go
Compiles!?
```

it compiles, this is surprising!? The variable here is also declared but not
used, it could be part of the public API of the package, or truly in this case
unused. ~~(I have searched online and can not find an explanation for this
behavior, if you know why please drop me a comment)~~. In my quest to reinforce
opinions similar to that of golang, I found a cool tool
[staticcheck](https://staticcheck.io/docs/) which will catch unused variables
of this nature and does even much more.

```bash
$> staticcheck will_not_compile_file_scope.go
will_not_compile_file_scope.go:5:5: var willNotCompile is unused (U1000)
```

This catches the unused variable in the file scope.

[staticcheck](https://staticcheck.io/docs/) catches a host of other things, I
will cover a few of other nice things staticcheck catches, the documentation is
more thorough on this:

- [Unused Constants](#unused-constants)
- [Unused Functions](#unused-functions)
- [Unused Embedded Functions](#unused-embedded-functions)
- [Unused Struct Fields](#unused-struct-fields)
- [Unnecessary Nil Checks](#unnecessary-nil-checks)

### Unused Constants

```go
package main
const unusedConstant int = 30
func main() {}
```

```bash
$> staticcheck unused_constants.go
unused_constants.go:3:7: const unusedConstant is unused (U1000)
```

[staticcheck](https://staticcheck.io/docs/) is able to detect that the constant
`unusedConstant` is truly not used.

### Unused Functions

```go
package main
func unused() {}
func main() {}
```

```bash
$> staticcheck unused_function.go
unused_function.go:5:6: func unused is unused (U1000)
```

The function `unused` is truly not used and staticcheck correctly reports this.

### Unused Embedded Functions

```go
package main
import "fmt"
type Foo struct {}
func (*Foo) ancestory() {
  fmt.Println("Ancestor is foo")
}
func (*Foo) t() {
  fmt.Println("Type is Foo")
}
type Bar struct {
  Foo
}
func (*Bar) t() {
  fmt.Println("Type is Bar")
}
func main() {
  bar := &Bar{}
  bar.ancestory()
  bar.t()
}
```

```bash
$> staticcheck unused_embedded_functions.go
unused_embedded_functions.go:11:13: func (*Foo).t is unused (U1000)
```

staticcheck correctly detects that the function `(*Foo).t` is unused.

### Unused Struct Fields:

```go
package main
import "fmt"
func main() {
  foo := struct{
    a int
    b int
  }{a: 1}
  fmt.Printf("%v\n", foo)
}
```

```bash
$> staticcheck unused_struct_fields.go
unused_struct_fields.go:8:3: field b is unused (U1000)
```

### Unnecessary Nil Checks

```go
package main
func main() {
  var foos []int
  if foos != nil {
    for i := range foos {
      _ = i + 2
    }
  }
  var bars map[string]int
  if _, ok := bars["bar"]; ok {
    delete(bars, "bar")
  }
}
```

```bash
$> staticcheck unnecessary_nil_checks.go
unnecessary_nil_checks.go:6:2: unnecessary nil check around range (S1031)
unnecessary_nil_checks.go:14:2: unnecessary guard around call to delete (S1033)
```

[staticcheck](https://staticcheck.io/docs/) is able to way more than this, do
read the documentation for its capabilities.

## VIM Tip

If you are a vim user, staticcheck output is in the
[quickfix](http://vimdoc.sourceforge.net/htmldoc/quickfix.html) format so you
can, use `:cexpr system('staticcheck ./...')` in normal mode to make vim
populate your quickfix list with staticcheck warnings then navigate to the
warnings with your standard `c{n,p}{f}`

## Real world example.

I ran it against [kubernetes](https://github.com/kubernetes/kubernetes), it did
find a bunch of things and I opened a couple of PRs, but this PR:
[https://github.com/kubernetes/kubernetes/pull/77325/files](https://github.com/kubernetes/kubernetes/pull/77325/files)
highlights its capabilities.
