# link battle

Use ghidra to see the library libflaggen.so, we found a function getflag.
```
undefined8 getflag(int param_1)
...
if (param_1 != 0x1a0a) {
  return 0;
}
...
putchar(uVar5 & 0xff ^ (int)*pcVar1);
...
```
The parameter need to be 0x1a0a, and getflag function will generate flag and print it.

We can use ctypes module in python, use it to load shared library .so to program.
```
...
import ctypes
flag = ctypes.cdll.LoadLibrary("./libflaggen.so")
flag.getflag(0x1a0a)
...
```
Get the flag:
```
flag{pl34s3_sp34k_3ngl1sh_m1n10n!_1v3_been_b4k1ng_und3r_th0s3_st00d1o_l1ghts!}
```
