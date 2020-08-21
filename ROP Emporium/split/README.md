### split
The problem guides has told you that same as ret2win, there is a 0x20 buffer, can be overflowed, and there is still a syscall in a function named usefulFunction.

Use ghidra to track the code.

There is indeed a 0x20 buffer, and indeed a syscall in function usefulFunction.

However, the syscall command is not /bin/cat flag.txt, but /bin/ls.
```
...
00400746 bf 4a 08        MOV        EDI=>s_/bin/ls_0040084a,s_/bin/ls_0040084a       = "/bin/ls"
         40 00
0040074b e8 10 fe        CALL       system                                           int system(char * __command)
         ff ff
...
```
