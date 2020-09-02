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
x86 use stack to pass function arguments, therefore, we need to store 
