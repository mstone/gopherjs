GopherJS - A compiler from Go to JavaScript
---------------------------------------------

[![Circle CI](https://circleci.com/gh/gopherjs/gopherjs.svg?style=svg)](https://circleci.com/gh/gopherjs/gopherjs)

GopherJS compiles Go code ([golang.org](http://golang.org/)) to pure JavaScript code. Its main purpose is to give you the opportunity to write front-end code in Go which will still run in all browsers. Give GopherJS a try on the [GopherJS Playground](http://gopherjs.github.io/playground/).

### What is supported?
Nearly everything, including Goroutines ([compatibility table](doc/packages.md)). Performance is quite good in most cases, see [HTML5 game engine benchmark](http://ajhager.github.io/enj/).

### Installation and Usage
Get or update GopherJS and dependencies with:
```
go get -u github.com/gopherjs/gopherjs
```
Now you can use  `gopherjs build [files]` or `gopherjs install [package]` which behave similar to the `go` tool. For `main` packages, these commands create a `.js` file and `.js.map` source map in the current directory or in `$GOPATH/bin`. The generated JavaScript file can be used as usual in a website. Use `gopherjs help [command]` to get a list of possible command line flags, e.g. for minification and automatically watching for changes. If you want to run the generated code with Node.js, see [this page](doc/syscalls.md).

*Note: GopherJS will try to write compiled object files of the core packages to your $GOROOT/pkg directory. If that fails, it will fall back to $GOPATH/pkg.*

### Performance Tips

- Use the `-m` command line flag to generate minified code.
- Apply gzip compression (http://en.wikipedia.org/wiki/HTTP_compression).
- Use `int` instead of `(u)int8/16/32/64`.
- Use `float64` instead of `float32`.

### Community
- [Google Group](https://groups.google.com/d/forum/gopherjs)
- [Bindings to JavaScript APIs and libraries](https://github.com/gopherjs/gopherjs/wiki/bindings)
- [GopherJS on Twitter](https://twitter.com/GopherJS)

### Getting started
#### Interacting with the DOM
The package `github.com/gopherjs/gopherjs/js` (see [documentation](http://godoc.org/github.com/gopherjs/gopherjs/js)) provides functions for interacting with native JavaScript APIs. For example the line
```js
document.write("Hello world!");
```
would look like this in Go:
```go
js.Global.Get("document").Call("write", "Hello world!")
```
You may also want use the [DOM bindings](http://dominik.honnef.co/go/js/dom), the [jQuery bindings](https://github.com/gopherjs/jquery) (see [TodoMVC Example](https://github.com/gopherjs/todomvc)) or the [AngularJS bindings](https://github.com/gopherjs/go-angularjs). Those are some of the [bindings to JavaScript APIs and libraries](https://github.com/gopherjs/gopherjs/wiki/bindings) by community members.

#### Providing library functions for use in other JavaScript code
Set a global variable to a map that contains the functions:
```go
package main

import "github.com/gopherjs/gopherjs/js"

func main() {
  js.Global.Set("myLibrary", map[string]interface{}{
    "someFunction": someFunction,
  })
}

func someFunction() {
  [...]
}
```
For more details see [Jason Stone's blog post](http://legacytotheedge.blogspot.de/2014/03/gopherjs-go-to-javascript-transpiler.html) about GopherJS.

### Architecture

#### General
GopherJS emulates a 32-bit environment. This means that `int`, `uint` and `uintptr` have a precision of 32 bits. However, the explicit 64-bit integer types `int64` and `uint64` are supported. The `GOARCH` value of GopherJS is "js". You may use it as a build constraint: `// +build js`.

#### Goroutines
Goroutines are fully supported by GopherJS. The only restriction is that you need to start a new goroutine if you want to use blocking code called from external JavaScript:

```go
js.Global.Get("myButton").Call("addEventListener", "click", func() {
  go func() {
    someBlockingFunction()
  }()
})
```

How it works:

JavaScript has no concept of concurrency (except web workers, but those are too strictly separated to be used for goroutines). Because of that, instructions in JavaScript are never blocking. A blocking call would effectively freeze the responsiveness of your web page, so calls with callback arguments are used instead.

GopherJS does some heavy lifting to work around this restriction: Whenever an instruction is blocking (e.g. communicating with a channel that isn't ready), the whole stack will unwind (= all functions return) and the goroutine will be put to sleep. Then another goroutine which is ready to resume gets picked and its stack with all local variables will be restored. This is done by preserving each stack frame inside a closure.
