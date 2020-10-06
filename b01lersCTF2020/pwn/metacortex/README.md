# metacortex
In this problem, there is a system("/bin/sh") right in main function, but there is a condition to execute system, the condition will always be false, we need to find way to bypass the condition.

Use ghidra to disassembly.

The condition is:
```
...
if (lVar1 >> 0x20 == (long)iVar2) {
  ...
  system("/bin/sh",puVar4[-0x58]);
  ...
```
Look through the code, we find that there is a read function and buffer size is 8 bytes, but read bytes length is 0x80, therefore, we are able to overflow the stack.
```
...
read(0,local_28,0x80,puVar4[-0x58]);
...
```
Check the security of the binary:
```
CANARY    : ENABLED
FORTIFY   : disabled
NX        : ENABLED
PIE       : ENABLED
RELRO     : FULL
```

The security level is almost the top, which means that we cannot modify return address since canary is enabled. We can only modify certain content in stack, maybe we can modify the variable value used for condition.

```
...
lVar1 = *local_30; //cannot be modified
...
iVar2 = atoi(local_28,puVar4[-0x58]); //obtain from read
...
```
iVar2 value is obtained from read, which can be controlled, the problem is lVar1, this value cannot be modified directly, we look at the position of variables in stack.
```
...
long lVar1;
int iVar2;
...
ulong local_28;
...
```
We found that the read bufferr local_28 is lower than variable lVar1, which means that we may able to overflow its value, and iVar2 is decided by our input, so we can pass the condition. 

read syscall will read length bytes, even through it meets bytes '\x00'. atoi will stop when it meets '\x00'. Then we can use the value before '\x00' to control iVar2, and use bytes after it to overflow lVar1 value.

We can set 0x4141414141414141 + '\x00' to variable iVar2, and use 'A'*n to overflow the stack, if we overflow the variable lVar1 correctly, the value of lVar1 will be 'A'*8 = 0x4141414141414141.

The payload will be:
```
...
#0x4141414141414141 = 4702111234474983745
payload = b'4702111234474983745\x00' + b'A'*70 # set value to iVar2, and overflow the value of lVar1
...
```
Execute exploit.py, we can get the shell.
