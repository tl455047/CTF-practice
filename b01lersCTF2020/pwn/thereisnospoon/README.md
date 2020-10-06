# thereisnospoon

In this problem, there is a system("/bin/sh") in main function, and there is also a condition:
```
...
if (*changeme != 0xff) {
  system("/bin/sh");
...
```
changeme is a malloc pointer, it seems that there is no way to modify the value in heap section. Look through the source code, we find that there are two read functions.
```
...
char buffer[256];
...
len = read(0, buffer, len);
...
char * buffer2 = malloc(strlen(buffer));
int * changeme = malloc(sizeof(int));
...
len = read(0, buffer2, len);
...
```
We find a strange thing that why buffer use strlen(buffer) as its buffer size? We know that read syscall will read len bytes even through it meets '\x00' or '\n', but strlen() will stop when it meets '\x00', therefore, if we input strings which contains '\x00', the len and strlen will be different. And in second read syscall, we are able to overflow the heap.

Chunks in heap section may not be continuous, but if we overflow enought length bytes, we may able to overflow the value of changeme.

The payload will be:
```
...
payload = b'\x00'*32 # for first read, len will be 33, and strlen will be 0
...
payload = b'A'*32 # for second read, since buffer2 size is 0, then the input will overflow the place in heap
...
``` 
Execute exploit.py, we get the shell.

