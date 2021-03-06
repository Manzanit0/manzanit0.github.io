---
layout: post
title: Escape analysis in Go
author: Javier Garcia
category: Go
tags: Go, GarbageCollection
---

In any kind of software, there is always two kind of memories: stack and heap.
The **stack** memory is alloted for each function execution and freed when the
function resolves, while the **heap** memory is system wide and must be
explicitely alloted and freed, except in garbage collected languages, like Go.

This is relevant to introduce what I learnt today: in Go it is important how you
use your memory because if the usage _escapes_ the function, it will be
allocated in the heap instead of the stack, thus having to be freed by the
garbage collector.  This is an issue because when the garbage collector runs,
the application routines get less share of the CPU, hence slowing down our
application

For a more in-depth explanation, I recommend [this post][0].

The second thing I learnt today is that you can actually analyse the memory
scapes of your application with the out of the box `escapenalysis.go`. To do
this, simply build with the flags `-gcflags -m` and it will print the full
analysis, like below.

*Note*: this example is from the [golarm repository at commit cb6bf1a012][1]

```bash
➜  golarm git:(master) go build -gcflags -m
# _/Users/manzanit0/repositories/golarm
./main.go:35:34: inlining call to os.Executable
./main.go:35:34: inlining call to os.executable
./main.go:35:34: inlining call to errors.New
./main.go:14:27: inlining call to flag.String
./main.go:15:24: inlining call to flag.String
./main.go:16:12: inlining call to flag.Parse
./main.go:28:21: ... argument does not escape
./main.go:30:12: ... argument does not escape
./main.go:34:18: leaking param: cron
./main.go:34:31: leaking param: message
./main.go:35:34: &errors.errorString literal escapes to heap
./main.go:35:34: os.initCwd + string("/") + os.ep·3 escapes to heap
./main.go:37:12: ... argument does not escape
./main.go:40:24: ... argument does not escape
./main.go:40:25: cron escapes to heap
./main.go:40:25: executable escapes to heap
./main.go:40:25: message escapes to heap
./main.go:41:21: ... argument does not escape
./main.go:43:12: ... argument does not escape
<autogenerated>:1: leaking param: .this
```

[0]: https://www.ardanlabs.com/blog/2017/05/language-mechanics-on-escape-analysis.html
[1]: https://github.com/Manzanit0/golarm/tree/cb6bf1a01201f6c1b8530a21e3f6c1717174f15b
