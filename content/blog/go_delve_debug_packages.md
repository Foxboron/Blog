---
title: "coredumpctl, delve and debug packages for Go"
date: "2022-11-19T19:00:00+02:00"
tags:
- golang
- delve
- archlinux
- english
---

I have spent a fair amount of time hacking on debug packages the past two years.
This work resulted in Arch Linux announcing the public [debuginfod
server](https://archlinux.org/news/debug-packages-and-debuginfod/) which allows
users to download symbols and source code to debug software running on their
system.

With this service users don't need to figure out what the debug packages are
called, installing them and maybe removing it afterwards. It also saves a fair
amount of data you need to download. Generally just a great service with a good
list of supported clients.

`coredumpctl` is a tool from `systemd` that effectively installs crash handlers
on your system. If a process dumps its core, it will keep track of it, store it
and make it easier for you to debug them through debuggers like `gdb` and
`lldb`.  The core dump contains a memory dump with variables that where defined,
a stack trace so you can see what was executed.

As a maintainer of the Go compiler on Arch I also wanted to make sure the Go
specific debugger `delve` also could make use of the tooling.

Thus in this example we are going to be using the debugger
[delve](https://github.com/go-delve/delve) to debug uh... delve!

The binary we are going to crash is from the `delve` package in Arch Linux. It
is stripped form all debug symbols, and there is no source code available for
the binary on this system. We want to use `coredumpctl` to simplify dealing with
the core dump, and we want to debug this with `delve` itself.

All the source and symbols is from `debuginfod.archlinux.org`.

{{< highlight sh >}}
$ GOTRACEBACK=crash dlv dap &
[1] 226038
$ kill -SEGV $!
SIGSEGV: segmentation violation
PC=0x563722bbb981 m=0 sigcode=0

[...]
[1]  + IOT instruction (core dumped)  GOTRACEBACK=crash dlv dap
{{< /highlight >}}

Here we just launch the [`dap`](https://microsoft.github.io/debug-adapter-protocol/) server in `delve` in the background. We use the
`$!` shorthand, which references the process ID, and simply kill it.
`GOTRACEBACK=crash` instructs the Go [runtime](https://pkg.go.dev/runtime) to
raise `SIGABRT` when it exists which
will create a core dump. As we have `systemd` installed we can
then use the `coredumpctl` to interact with the core dump.

{{< highlight bash >}}
$ coredumpctl list dlv
TIME                           PID  UID  GID SIG     COREFILE EXE          SIZE
Thu 2022-11-17 21:27:53 CET 226038 1000 1000 SIGABRT present  /usr/bin/dlv 2.1M
{{< /highlight >}}

Here we see when the core dump happened. What process it was for, the PID and
the user/group IDs.

Before we use `coredumpctl` to inspect this core dump we need to install
`debuginfod` so `delve` can use the `debuginfod-find` binary to download sources
for us.

{{< highlight bash >}}
# Note you need to re-exec the shell or source /etc/profile.d/debuginfod.sh
# after installing debuginfod.
$ pacman -S debuginfod
{{< /highlight >}}

Earlier this year I wrote up the patches needed for `delve` to understand source
listings from `debuginfod`. Along with a tiny bit of refactoring so code
de-duplication was possible.

https://github.com/go-delve/delve/pull/2885

This is the feature we are going to be using to actually inspect the symbols and
the source listing in the debugging session.

Please note I wrote a small patch so `coredumpctl` can use `delve`, as it expects
core dumps to be passed through a [`-c/-core`
switch](https://github.com/go-delve/delve/pull/3195). It should be part of the
next delve release.

{{< highlight bash >}}
$ coredumpctl debug --debugger=./dlv -A core dlv
           PID: 226038 (dlv)
           UID: 1000 (fox)
           GID: 1000 (fox)
        Signal: 6 (ABRT)
     Timestamp: Thu 2022-11-17 21:27:53 CET (16min ago)
  Command Line: dlv dap
    Executable: /usr/bin/dlv
  Size on Disk: 2.1M
       Message: Process 226038 (dlv) of user 1000 dumped core.

                Stack trace of thread 226048:
                #0  0x0000563722bbb401 n/a (dlv + 0x1a8401)
                #1  0x0000563722b9f805 n/a (dlv + 0x18c805)
                # ...
                #17 0x0000563722bb77a5 n/a (dlv + 0x1a47a5)
                ELF object binary architecture: AMD x86-64

Type 'help' for list of commands.
(dlv) bt
 0  0x000055799c40a401 in runtime.raise
    at /usr/lib/go/src/runtime/sys_linux_amd64.s:159
 1  0x000055799c3ede65 in runtime.dieFromSignal
    at /usr/lib/go/src/runtime/signal_unix.go:870
 2  0x000055799c3ee805 in runtime.sigfwdgo
    at /usr/lib/go/src/runtime/signal_unix.go:1086
 3  0x000055799c3ecb47 in runtime.sigtrampgo
    at /usr/lib/go/src/runtime/signal_unix.go:432
 4  0x000055799c40a6e9 in runtime.sigtramp
    at /usr/lib/go/src/runtime/sys_linux_amd64.s:359
 5  0x00007f16ee0d0a00 in ???
    at ?:-1
    [...snip....]
19  0x000055799c92502e in ???
    at ?:-1
    error: error while reading spliced memory at 0x7f15ae7fd260: EOF
(truncated)
(dlv)
{{< /highlight >}}


We have now asked `coredumpctl` to debug the last core dump of the `dlv` binary.
Here we can see the backtrace of the goroutine that failed. We can see
`runtime.raise` was used and we hit the function `dieFromSignal`. This makes
sense considering we killed the process.

The above stack trace contains code found locally installed.  Our interest is to
look at the `delve` source code though! Using `grs` we can list the goroutines
from the crash, and as `Goroutine 1` contains symbols to the `delve` source we
will take a peak at that one.

{{< highlight bash >}}
(dlv) grs
  Goroutine 1 - User: /usr/src/debug/delve/delve-1.9.1/cmd/dlv/cmds/commands.go:831 github.com/go-delve/delve/cmd/dlv/cmds.waitForDisconnectSignal (0x5637230d382c) [select]
  Goroutine 2 - User: /usr/lib/go/src/runtime/proc.go:364 runtime.gopark (0x563722b8baf6) [force gc (idle)]
  Goroutine 3 - User: /usr/lib/go/src/runtime/proc.go:364 runtime.gopark (0x563722b8baf6) [GC sweep wait]
  Goroutine 4 - User: /usr/lib/go/src/runtime/proc.go:364 runtime.gopark (0x563722b8baf6) [GC scavenge wait]
  Goroutine 5 - User: /usr/lib/go/src/runtime/proc.go:364 runtime.gopark (0x563722b8baf6) [finalizer wait]
  Goroutine 18 - User: /usr/lib/go/src/net/fd_unix.go:172 net.(*netFD).accept (0x563722c89a95) [IO wait]
  Goroutine 19 - User: /usr/lib/go/src/runtime/proc.go:364 runtime.gopark (0x563722b8baf6) [select]
  Goroutine 20 - User: /usr/lib/go/src/runtime/sigqueue.go:152 os/signal.signal_recv (0x563722bb62af) (thread 226044)
[8 goroutines]
(dlv) gr 1
Switched from 0 to 1 (thread 226048)
(dlv) bt
 0  0x0000563722b8baf6 in runtime.gopark
    at /usr/lib/go/src/runtime/proc.go:364
 1  0x0000563722b9af7c in runtime.selectgo
    at /usr/lib/go/src/runtime/select.go:328
 2  0x00005637230d382c in github.com/go-delve/delve/cmd/dlv/cmds.waitForDisconnectSignal
    at /usr/src/debug/delve/delve-1.9.1/cmd/dlv/cmds/commands.go:831
 3  0x00005637230d066f in github.com/go-delve/delve/cmd/dlv/cmds.dapCmd.func1
    at /usr/src/debug/delve/delve-1.9.1/cmd/dlv/cmds/commands.go:511
 4  0x00005637230cfede in github.com/go-delve/delve/cmd/dlv/cmds.dapCmd
    at /usr/src/debug/delve/delve-1.9.1/cmd/dlv/cmds/commands.go:513
 5  0x00005637230c50e3 in github.com/spf13/cobra.(*Command).execute
    at /usr/src/debug/delve/pkg/mod/github.com/spf13/cobra@v1.1.3/command.go:856
 6  0x00005637230c56dd in github.com/spf13/cobra.(*Command).ExecuteC
    at /usr/src/debug/delve/pkg/mod/github.com/spf13/cobra@v1.1.3/command.go:960
 7  0x00005637230d54aa in github.com/spf13/cobra.(*Command).Execute
    at /usr/src/debug/delve/pkg/mod/github.com/spf13/cobra@v1.1.3/command.go:897
 8  0x00005637230d54aa in main.main
    at /usr/src/debug/delve/delve-1.9.1/cmd/dlv/main.go:24
 9  0x0000563722b8b733 in runtime.main
    at /usr/lib/go/src/runtime/proc.go:250
10  0x0000563722bb9ae1 in runtime.goexit
    at /usr/lib/go/src/runtime/asm_amd64.s:1594
{{< /highlight >}}

This looks more interesting! We have references to `/usr/src/debug/delve` which
is the source code from the debug packages. We can take a peak at frame 4.
    
{{< highlight bash >}}
(dlv) frame 4
> runtime.gopark() /usr/lib/go/src/runtime/proc.go:364 (PC: 0x563722b8baf6)
Frame 4: /usr/src/debug/delve/delve-1.9.1/cmd/dlv/cmds/commands.go:513 (PC: 5637230cfede)
   508:			} else { // work with a predetermined client.
   509:				server.RunWithClient(conn)
   510:			}
   511:			waitForDisconnectSignal(disconnectChan)
   512:			return 0
=> 513:		}()
   514:		os.Exit(status)
   515:	}
   516:	
   517:	func buildBinary(cmd *cobra.Command, args []string, isTest bool) (string, bool) {
   518:		debugname, err := filepath.Abs(cmd.Flag("output").Value.String())
(dlv)
{{< /highlight >}}

Behind the scenes we have now fetched the file
`/usr/src/debug/delve/delve-1.9.1/cmd/dlv/cmds/commands.go` from debuginfod.
This works transparently and it's as if the source was always present on the
system. We can also look at module code! Lets check out frame 5.

{{< highlight bash >}}
(dlv) frame 5
> runtime.gopark() /usr/lib/go/src/runtime/proc.go:364 (PC: 0x563722b8baf6)
Frame 5: /usr/src/debug/delve/pkg/mod/github.com/spf13/cobra@v1.1.3/command.go:856 (PC: 5637230c50e3)
   851:		if c.RunE != nil {
   852:			if err := c.RunE(c, argWoFlags); err != nil {
   853:				return err
   854:			}
   855:		} else {
=> 856:			c.Run(c, argWoFlags)
   857:		}
   858:		if c.PostRunE != nil {
   859:			if err := c.PostRunE(c, argWoFlags); err != nil {
   860:				return err
   861:			}
{{< /highlight >}}

This shows us some code from the library cobra which is used for the flag and
command handling inside delve. This means we have full insight into the code we
ran through in the software that arrived from other modules.

And it just works!

I'm quite pleased how this tooling actually works, and hopefully people find it
useful. A lot of these things seems largely unexplored in the case of
minimalistic containers. Debug stripping is often done to save on space, both
for containers and Linux distributions.

Having a kubernetes cluster with crash handlers, and a debuginfod server with
delve support seems like a cool thing that should exist I guess.

I plan on writing a longer blog post how the infrastructure in Arch Linux was
implemented, along a longer post describing what I learned implementing better
debug package support in pacman.
