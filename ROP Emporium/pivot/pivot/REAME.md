# pivot
In this problem, we need to practice two important concept, stack pivot, and lib base address leak. Although the binary's name is called pivot, in fact the binary has already allocate an area of memory, which allowed user to write rop chain directly. The only thing we need to do is stack pivot. 

The binary will read user input to a specific address area, then we can directly write our rop chain into the space.

So the most difficult problem will be that we need to call a function ret2win, and this function is in libpivot, its address is randomized and is not in the plt table. We need to first use a function foothold_function, which is also in libpivot and is in plt table to leak the random base address of libpivot library, then we can calculate the correct address of the ret2win and call it.

first, check the security:
```
CANARY    : disabled
FORTIFY   : disabled
NX        : ENABLED
PIE       : disabled
RELRO     : Partial
```
Since PIE is disabled, we can directly use the address in binary, and obtain the got, plt table.

In this binary, there are two read parts, the first one is used to write rop chain in stack pivot area, and the second one is used to do buffer overflow and stack pivot.

This binary will tell us the stack pivot area address, we need to store this address and use it to do stack pivot.

Run the program you will see the address of area:
```
...
The Old Gods kindly bestow upon you a place to pivot: 0x7ffff77e0f10
Send a ROP chain now and it will land there
...
```
We can do stack pivot by doing leave; ret twice.

When we do buffer overflow, we will do leave; ret defaultly, and we can use a gadget to do it second time. We need to store the new address we obtain before to rbp, then in the first time leave, we pop the address into rbp, which means that we transfer the stack base to the new space, the then we apply the second leave will set rsp to new address, finally we tranfer the whole stack to the new place.

the payload will be:
```
...
gadget1 = 0x04008ef # leave; ret;
...
payload = 'A'*32 + p64(rop_add) //set rbp value
payload += p64(gadget1) // leave; ret
..
```

Secondly, we need to leak lib base address, in this problem we can use foothold_functino and puts to do this.

we need to call foothold function first, since it will not be used in this program, we need to call it once to let its address written to got table. Then we can puts its got address, and we can use it to calculate correct address of the ret2win function.




