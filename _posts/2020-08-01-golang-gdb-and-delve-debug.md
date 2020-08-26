---
layout: post
title: golang调试工具gdb和delve
tags: [golang, debug]
---

   编译时使用参数-gcflags “-N -l”,这样可以忽略Go内部做的一些优化，聚合变量和函数等优化，关闭内联优化；
当然如果发布的话不需要关闭内联优化，可以-ldflags “-s -w”加这个参数去掉符号信息和DWARF调试信息。   
版本信息：  
```
gdb --version
GNU gdb (GDB) 9.1

dlv version
Delve Debugger
Version: 1.5.0

go version
go version go1.13.8 darwin/amd64
```
hello world示例:
```golang
package main

import "fmt"

func main() {
    varTest := "hello world!"
    fmt.Println(varTest)
}
```

## gdb调试golang
  官方不推荐用gdb调试，更推荐golang开发的delve，比gdb更容易理解golang的数据结构及runtime,当然用gdb调试go runtime貌似也是可以的；如果只是撸业务逻辑，我一般都是用IDE来调试，主要是更方便；当然是debug不了runtime的。  
  在mac上用gdb调试,由于golang1.11版本后为了减小bin文件的大小，对调试信息做了压缩，而mac上的gdb版本识别不了dwarf压缩后的调试信息，编译的时候需要加-ldflags=-compressdwarf=false，关闭压缩。  
  官方文档https://golang.org/doc/gdb   
  参考这偏博文的gdb命令行解释：https://blog.csdn.net/liigo/article/details/582231?utm_source=copy
```bash
go build -gcflags "-N -l" -ldflags "-compressdwarf=false" -o hello main.go
```  

```bash
sudo gdb -tui hello   #加上-tui参数可以查看到代码中的断点，非常方便。
(gdb) info files
Symbols from "/Users/zhouxiaoming/worker/gohello/hello".
Local exec file:
	`/Users/zhouxiaoming/worker/gohello/hello', file type mach-o-x86-64.
	Entry point: 0x1054b20
	0x0000000001001000 - 0x00000000010995e3 is .text
	0x0000000001099600 - 0x00000000010ed24a is __TEXT.__rodata
	0x00000000010ed260 - 0x00000000010ed374 is __TEXT.__symbol_stub1
	0x00000000010ed380 - 0x00000000010ee084 is __TEXT.__typelink
	0x00000000010ee088 - 0x00000000010ee0f8 is __TEXT.__itablink
	0x00000000010ee0f8 - 0x00000000010ee0f8 is __TEXT.__gosymtab
	0x00000000010ee100 - 0x0000000001163325 is __TEXT.__gopclntab
	0x0000000001164000 - 0x0000000001164020 is __DATA.__go_buildinfo
	0x0000000001164020 - 0x0000000001164190 is __DATA.__nl_symbol_ptr
	0x00000000011641a0 - 0x00000000011712f8 is __DATA.__noptrdata
	0x0000000001171300 - 0x0000000001177f90 is .data
	0x0000000001177fa0 - 0x00000000011938f0 is .bss
	0x0000000001193900 - 0x0000000001195ee8 is __DATA.__noptrbss
(gdb) b *0x1054b20
Breakpoint 1 at 0x1054b20: file /Users/zhouxiaoming/.gvm/gos/go1.13.8/src/runtime/rt0_darwin_amd64.s, line 8.
```
可以通过断点查看到entry并不是我们的main.main,而是一个golang源码中对应硬件和系统的汇编文件。
```bash
#include "textflag.h"

TEXT _rt0_amd64_darwin(SB),NOSPLIT,$-8
	JMP	_rt0_amd64(SB)

// When linking with -shared, this symbol is called when the shared library
// is loaded.
TEXT _rt0_amd64_darwin_lib(SB),NOSPLIT,$0
	JMP	_rt0_amd64_lib(SB)
```
查看源码发现实际上是jmp到_rt0_amd64了，再通过断点查看_rt0_amd64在哪里。
```bash
b *_rt0_amd64
Breakpoint 2 at 0x1051170: file /Users/zhouxiaoming/.gvm/gos/go1.13.8/src/runtime/asm_amd64.s, line 15.
```

通过gdb run的时候发现报错了Unable to find Mach task port for process-id 6937，这个应该是没有权限，所以用gdb的时候加上sudo就好了。  
实际还有其他解决方法如https://stackoverflow.com/questions/10221448/emacs-24-and-gdb-6-3-on-mac-os-x/10441587#10441587；   

以上是可以追踪runtime的源码方式，这里就不做详细讨论了，接下来给main打个断点跑一下，然后打印下golang runtime相关的一些信息。

```bash
(gdb) b main.main
Breakpoint 1 at 0x1099550: file /Users/zhouxiaoming/worker/gohello/main.go, line 5.
(gdb) run
```
郁闷咋又报错了
```bash
During startup program terminated with signal ?, Unknown signal.
```
赶紧google了一波，还是mac上的问题，要在使用目录下新建一个文件
```bash
echo "set startup-with-shell off" >> ~/.gdbinit
```
重跑一下，这次跟代码具体第几行打个断点，再跑
```bash
(gdb) list
1	package main
2
3	import "fmt"
4
5	func main() {
6		varTest := "hello world!"
7		fmt.Println(varTest)
8	}
(gdb) break main.go:7
Breakpoint 1 at 0x1099586: file /Users/zhouxiaoming/worker/gohello/main.go, line 7.
(gdb) run
Starting program: /Users/zhouxiaoming/worker/gohello/hello
[New Thread 0xe03 of process 8253]
[New Thread 0x2903 of process 8253]
warning: unhandled dyld version (16)
[New Thread 0x1507 of process 8253]
[New Thread 0x1703 of process 8253]
[New Thread 0x1803 of process 8253]
[New Thread 0x2803 of process 8253]

Thread 2 hit Breakpoint 1, main.main () at /Users/zhouxiaoming/worker/gohello/main.go:7
7		fmt.Println(varTest)
```
debug一下基础信息，参考官方文档。
```bash
Thread 2 hit Breakpoint 1, main.main () at /Users/zhouxiaoming/worker/gohello/main.go:7
(gdb) info goroutines
* 1 running  runtime.systemstack_switch
  2 waiting  runtime.gopark
  3 waiting  runtime.gopark
  4 waiting  runtime.gopark
  17 waiting  runtime.gopark
(gdb) p varTest
$1 = 0x10d0f64 "hello world!"
```

## delve调试golang
 delve是golang官方推荐的debug工具，所以如果是golang程序，就用这个工具比较好，可以不用gdb；  
 实际上是golang写的一个开源工具github上有详细的文档可以查看https://github.com/go-delve/delve/tree/master/Documentation       
安装
```bash
go get github.com/go-delve/delve/cmd/dlv

Available Commands:
  attach      Attach to running process and begin debugging.
  connect     Connect to a headless debug server.
  core        Examine a core dump.
  dap         [EXPERIMENTAL] Starts a TCP server communicating via Debug Adaptor Protocol (DAP).
  debug       Compile and begin debugging main package in current directory, or the package specified.
  exec        Execute a precompiled binary, and begin a debug session.
  help        Help about any command
  run         Deprecated command. Use 'debug' instead.
  test        Compile test binary and begin debugging program.
  trace       Compile and begin tracing program.
  version     Prints version.
```
本文专注于了解dlv debug命名的使用，默认执行dlv debug会debug 当前目录下的main.go文件。help中有详细的介绍详细，功能还是很强大的。
```bash
dlv debug

Type 'help' for list of commands.
(dlv)
(dlv) help
The following commands are available:

Running the program:
    call ------------------------ Resumes process, injecting a function call (EXPERIMENTAL!!!)
    continue (alias: c) --------- Run until breakpoint or program termination.
    next (alias: n) ------------- Step over to next source line.
    rebuild --------------------- Rebuild the target executable and restarts it. It does not work if the executable was not built by delve.
    restart (alias: r) ---------- Restart process.
    step (alias: s) ------------- Single step through program.
    step-instruction (alias: si)  Single step a single cpu instruction.
    stepout (alias: so) --------- Step out of the current function.

Manipulating breakpoints:
    break (alias: b) ------- Sets a breakpoint.
    breakpoints (alias: bp)  Print out info for active breakpoints.
    clear ------------------ Deletes breakpoint.
    clearall --------------- Deletes multiple breakpoints.
    condition (alias: cond)  Set breakpoint condition.
    on --------------------- Executes a command when a breakpoint is hit.
    trace (alias: t) ------- Set tracepoint.

Viewing program variables and memory:
    args ----------------- Print function arguments.
    display -------------- Print value of an expression every time the program stops.
    examinemem (alias: x)  Examine memory:
    locals --------------- Print local variables.
    print (alias: p) ----- Evaluate an expression.
    regs ----------------- Print contents of CPU registers.
    set ------------------ Changes the value of a variable.
    vars ----------------- Print package variables.
    whatis --------------- Prints type of an expression.

Listing and switching between threads and goroutines:
    goroutine (alias: gr) -- Shows or changes current goroutine
    goroutines (alias: grs)  List program goroutines.
    thread (alias: tr) ----- Switch to the specified thread.
    threads ---------------- Print out info for every traced thread.

Viewing the call stack and selecting frames:
    deferred --------- Executes command in the context of a deferred call.
    down ------------- Move the current frame down.
    frame ------------ Set the current frame, or execute command on a different frame.
    stack (alias: bt)  Print stack trace.
    up --------------- Move the current frame up.

Other commands:
    config --------------------- Changes configuration parameters.
    disassemble (alias: disass)  Disassembler.
    edit (alias: ed) ----------- Open where you are in $DELVE_EDITOR or $EDITOR
    exit (alias: quit | q) ----- Exit the debugger.
    funcs ---------------------- Print list of functions.
    help (alias: h) ------------ Prints the help message.
    libraries ------------------ List loaded dynamic libraries
    list (alias: ls | l) ------- Show source code.
    source --------------------- Executes a file containing a list of delve commands
    sources -------------------- Print list of source files.
    types ---------------------- Print list of types

Type help followed by a command for full documentation.

```

简单使用下来，还是觉得很方便，信息很详细。
```shell
(dlv) r
Process restarted with PID 8834
(dlv) c
> main.main() ./main.go:7 (hits goroutine(1):1 total:1) (PC: 0x10bfa66)
     2:
     3:	import "fmt"
     4:
     5:	func main() {
     6:		varTest := "hello world!"
=>   7:		fmt.Println(varTest)
     8:	}
(dlv) locals
varTest = "hello world!"
(dlv) grs
* Goroutine 1 - User: ./main.go:7 main.main (0x10bfa66) (thread 158449)
  Goroutine 2 - User: /Users/zhouxiaoming/.gvm/gos/go1.13.8/src/runtime/proc.go:305 runtime.gopark (0x102fafb)
  Goroutine 3 - User: /Users/zhouxiaoming/.gvm/gos/go1.13.8/src/runtime/proc.go:305 runtime.gopark (0x102fafb)
  Goroutine 4 - User: /Users/zhouxiaoming/.gvm/gos/go1.13.8/src/runtime/proc.go:305 runtime.gopark (0x102fafb)
  Goroutine 5 - User: /Users/zhouxiaoming/.gvm/gos/go1.13.8/src/runtime/proc.go:305 runtime.gopark (0x102fafb)
[5 goroutines]
(dlv) s
> fmt.Println() /Users/zhouxiaoming/.gvm/gos/go1.13.8/src/fmt/print.go:273 (PC: 0x10b9b13)
   268:	}
   269:
   270:	// Println formats using the default formats for its operands and writes to standard output.
   271:	// Spaces are always added between operands and a newline is appended.
   272:	// It returns the number of bytes written and any write error encountered.
=> 273:	func Println(a ...interface{}) (n int, err error) {
   274:		return Fprintln(os.Stdout, a...)
   275:	}
   276:
   277:	// Sprintln formats using the default formats for its operands and returns the resulting string.
   278:	// Spaces are always added between operands and a newline is appended.
(dlv) tr
Command failed: you must specify a thread
(dlv) threads
* Thread 158449 at 0x10b9b13 /Users/zhouxiaoming/.gvm/gos/go1.13.8/src/fmt/print.go:273 fmt.Println
  Thread 158474 at :0
  Thread 158475 at :0
  Thread 158476 at :0
  Thread 158477 at :0
(dlv) gr 1
Switched from 1 to 1 (thread 158449)
(dlv) gr 1 bt
0  0x00000000010b9b13 in fmt.Println
   at /Users/zhouxiaoming/.gvm/gos/go1.13.8/src/fmt/print.go:273
1  0x00000000010bfae2 in main.main
   at ./main.go:7
2  0x000000000102f754 in runtime.main
   at /Users/zhouxiaoming/.gvm/gos/go1.13.8/src/runtime/proc.go:203
3  0x000000000105acc1 in runtime.goexit
   at /Users/zhouxiaoming/.gvm/gos/go1.13.8/src/runtime/asm_amd64.s:1357
```


总结：gdb是很强大的debug工具，delve是更适合golang的调试工具。