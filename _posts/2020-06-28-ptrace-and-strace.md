---
layout: post
title: ptrace and strace
---

# TL;DR

1. summary
    * [ptrace][ptrace-man] make tracee stop on ptrace hook and notify tracer [[1]][ptrace-syscall-enter][[2]][ptrace-syscall-exit]
    * tracer use [ptrace][ptrace-man] syscall to get tracee registers and memory [[3]][ptrace-get-reg][[4]][ptrace-get-memory]
    * strace pretty output [syscall name][print-syscall-name] and [argument][open-parser] according to tracee registers and memory
        * strace get tracee registers and memory at tracee enter and exit syscall

2. ptrace
    * tracee call [ptrace(PTRACE_TRACEME)][linux-ptrace-syscall] to set process status to [PT_PTRACED][ptrace-set-flags]
    * tracee enter syscall execute ptrace hook [tracehook_report_syscall_entry][ptrace-syscall-enter] notify tracer
    * tracee exit syscall execute ptrace hook [tracehook_report_syscall_exit][ptrace-syscall-exit] notify tracer

3. strace
    * tracee call [ptrace][strace-ptraceme] set process status to PT_PTRACED [[5]][ptrace-set-flags]
    * [strace call wait tracee resume][strace-wait] 
    * strace call ptrace to get tracee registers and memory [[6]][strace-get-reg][[7]][strace-get-memory]
    * strace parse [syscall name][print-syscall-name] according to register value
    * strace print syscall argument, such as: [open][open-parser]

**links:**

* [1] [ptrace syscall enter hook][ptrace-syscall-enter]
* [2] [ptrace syscall exit hook][ptrace-syscall-exit]
* [3] [ptrace get tracee registers][ptrace-get-reg]
* [4] [ptrace get tracee memory][ptrace-get-memory]
* [5] [ptrace PTRACE_TRACEME option][ptrace-set-flags]
* [6] [strace get tracee registers][strace-get-reg]
* [7] [strace get tracee memory][strace-get-memory]

[ptrace-man]: https://man7.org/linux/man-pages/man2/ptrace.2.html
[strace-fork]: https://github.com/strace/strace/blob/v5.7/strace.c#L1542
[strace-exec]: https://github.com/strace/strace/blob/v5.7/strace.c#L1553
[strace-ptrace]: https://github.com/strace/strace/blob/v5.7/strace.c#L1582
[strace-wait]: https://github.com/strace/strace/blob/v5.7/strace.c#L3134
[syscall-enter]: https://github.com/strace/strace/blob/v5.7/syscall.c#L584
[print-syscall-name]: https://github.com/strace/strace/blob/v5.7/syscall.c#L591
[syscall-exit]: https://github.com/strace/strace/blob/v5.7/syscall.c#L717
[open-parser]: https://github.com/strace/strace/blob/v5.7/open.c#L118
[ptrace-set-flags]: https://github.com/torvalds/linux/blob/v5.7/kernel/ptrace.c#L487
[strace-ptraceme]: https://github.com/strace/strace/blob/v5.7/strace.c#L1339
[ptrace-syscall-enter]: https://github.com/torvalds/linux/blob/v5.7/arch/x86/entry/common.c#L85
[ptrace-syscall-exit]: https://github.com/torvalds/linux/blob/v5.7/arch/x86/entry/common.c#L251
[linux-ptrace-syscall]: https://github.com/torvalds/linux/blob/v5.7/kernel/ptrace.c#L1242
[ptrace-get-reg]: https://github.com/torvalds/linux/blob/v5.7/arch/x86/kernel/ptrace.c#L788
[ptrace-get-memory]: https://github.com/torvalds/linux/blob/v5.7/kernel/ptrace.c#L1014
[strace-get-reg]: https://github.com/strace/strace/blob/v5.7/linux/x86_64/get_syscall_args.c#L10
[strace-get-memory]: https://github.com/strace/strace/blob/v5.7/ucopy.c#L285
