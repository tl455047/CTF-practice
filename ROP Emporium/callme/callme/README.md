# callme x86 64
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
gdb-peda$ plt
...
Breakpoint 1 at 0x400720 (callme_one@plt)
Breakpoint 2 at 0x4006f0 (callme_three@plt)
Breakpoint 3 at 0x400740 (callme_two@plt)
```
We obtain three function address, before calling them we need to set the arguments properly.

Hint has told us that these functions need three parameters, which are 0xdeadbeefdeadbeef, 0xcafebabecafebabe, and 0xd00df00dd00df00d.

x86_64 calling convention tells us that the arguments are stored in the registers rdi, rsi, rdx, ...... 

Therefore, we need to set three parameters in rdi, rsi, rdx accordingly.

We find the suitable gadgets in program.
```
...
0x000000000040093c : pop rdi ; pop rsi ; pop rdx ; ret
0x00000000004009a3 : pop rdi ; ret
0x000000000040093e : pop rdx ; ret
...
```
We use thers gadgets to set the arguments, and then jump to the plt function address, and repeated 3 times.
```

```
Now, we can build our payload.

First, same as problems before, the overflow buffer is 32 bytes, and rbp register 8 bytes.
```
'A' * 32 + 'A' * 8
```



