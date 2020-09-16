# CSAW CTF Quals 2020
# roppity
Simple ret2libc problem. There are a binary rop and a library libc-2.27.so.

Use gdb or ghidra to look into rop binary, and found that there is no useful function to use, then the useful functions will be in libc.

We find the function system(), which can execute command, therefore, if we can know the address of this function, we can jump to this functionand execute command through it, and we also find a useful string "/bin/sh" in library. Same as function system, if we know the address of the string, we can use it as our command.

The main problem will be finding the correct address of the libc. According to the dynamic loading, the address of the library will be a random base address plus the offset of the function in library, which means that we only need to know the random base address, since the offset of the function is already known.

Due to ASLR, the base address will be randomized every execution. Therefore, we need to find the base address and use it as the same execution.

The simplest way to find the base address is to find the real address of the function in got table, we need to obtain this address, and this address is constructed by real address = base address + function offset, then base address = real address - function offset.

In binary rop plt table, there is a function puts, which can be choosed as the target and also output the address.

Since this is a x86_64 structure, function argument is strored in rdi, rsi... we put the got address of the puts function into rdi, and call puts function, then the process will print the address of puts.
```
#gadget for pop value into rdi
gadget1 = 0x0400683 # pop rdi ; ret

```
We can obtain plt, and got throught ELF:
```
elf_prog = ELF("./rop")
got_puts = elf_prog.got['puts']
plt_puts = elf_prog.plt['puts']
```
We know that the buffer is 32 bytes, and there is also a 8 bytes rbp.

There is also a stack alignment problem for ubuntu 18.04, and we need to add one more ret gadget for rop chain.

After obtaining the address of puts, we need to jump back to the beginning of the main function.

The first payload will be:
```
b'A'*40 + align_gadget + p64(gadget1) + p64(got_puts) + p64(plt_puts) + p64(main) 
```
Next, we can use this address to get base address, and use this base address to calculate system and "/bin/sh" address.
```
...
base = proc.recvline()
libc_puts = u64(base[:-1].ljust(8, b'\x00'))
#obtain the base address of libc, calculate correct system function address and string "/bin/sh" address
libc_base = libc_puts - libc.symbols['puts']
libc_system = libc_base + libc.symbols['system']
libc_binsh = libc_base + libc.search('/bin/sh').next()
...
```
Finally, we can pop the "/bin/sh" into rdi and call system function.
```
b'A'*40 + p64(gadget1) + p64(libc_binsh) + p64(libc_system)
```
