---
layout: post
title: Debug dspsr segmentation fault when nbin is large on certain systems
date: 2023-02-23
author: Fxzjshm
category: Pulsar
tags: [Linux]
---

**TL;DR:** this is a stack overflow caused by large array on stack, so enlarge stack size using `ulimit -s <stack size in KB>`

## Incident
An error occurred when processing raw baseband data of a radio telescope using dspsr:
```
$ dspsr -b 4194304 -D 56.716 -A -L 1.073741824 -c 1.073741824  -O ${file}_128 -e rf -F 128:D ${file}.bin -U 4096

Only single polarization detection available
dspsr: Single archive with multiple sub-integrations
dspsr: dedispersion filter length=131072 (minimum=8192) complex samples
dspsr: 128 channel dedispersing filterbank requires 33554432 samples
dspsr: blocksize=330382096 samples or 4096 MB
dsp::Fold::choose_nbin WARNING Requested nbin=4194304 > sensible nbin=2097152.  Where:
  sampling period     = 0.000256 ms and
  requested bin width = 0.000256 ms

dsp::Archiver::finish archive '13835058401541322426_128.rf' with 1 integrations
62305 Segmentation Fault (Core dumped)
```

## Analyze
<!-- more -->
Recompile dspsr & psrchive with debug info (`CFLAGS=-g`, `CXXFLAGS=-g`), then run with gdb, segmentation fault is again triggered:
```
Program received signal SIGSEGV, Segmentation fault.
0x00000000008188d4 in fcompwrite (nvals=4194304, vals=0x7fffd193c010, fptr=0xcb5be0) at fcomp.C:103
103       if (scale == 0.0 || isnan(scale)) {
```
```
=> 0x00000000008188d4 <+433>:   call   0x4fdc93 <_ZSt5isnanf>
```
with registers
```
(gdb) info reg
rax            0x4a3c0d03          1245449475
rbx            0x7fffffffd500      140737488344320
rcx            0x10                16
rdx            0xfffffc            16777212
rsi            0x7fffd193c010      140736709509136
rdi            0x400000            4194304
rbp            0x7fffffffd560      0x7fffffffd560
rsp            0x7fffff7fd500      0x7fffff7fd500
r8             0x400000            4194304
r9             0x0                 0
r10            0x400000            4194304
r11            0x0                 0
r12            0x0                 0
r13            0x7fffffffdc50      140737488346192
r14            0x0                 0
r15            0x0                 0
rip            0x8188d4            0x8188d4 <fcompwrite(unsigned int, float const*, _IO_FILE*)+433>
eflags         0x10202             [ IF RF ]
cs             0x33                51
ss             0x2b                43
...
```

Weird, why calling `isnan()` gives segmentation fault? But it seems indeed faulty:
```
(gdb) print isnan(scale)
Cannot access memory at address 0x7fffff7fd47f
```
the only thing that may link to this address is `rsp = 0x7fffff7fd500`, but then what?

### ... if know little about x86 assembly
Scanning the code, one array definition is noticed:
```c++
unsigned short int packed_buf [nvals];
```
where, in this context, `nvals = 4194304`. 

This array is on stack, and it is common that large array cannot fit into stack (depending on runtime configuration), so maybe this segmentation fault is actually a stack overflow.

### ... if know a little about x86 assembly
Register `rsp` is a pointer pointing to current location on stack and grows downward.
If `rsp - 0x81` cannot be accessed, then it means this position is beyond stack area, so possibly a stack overflow.

## Verify
This machine has a default stack size of 8 MiB:
```
$ ulimit -s
8192
```
but size of a `unsigned short int [nvals]` is 2 * 4194304 = 8 MiB, so another stack frame (like calling `isnan()` or whatever other function) gives segmentation fault immediately.

## Solution / Expedient
Enlarge stack size using `ulimit -s <stack size in KB>` before executing dspsr with these parameters.
