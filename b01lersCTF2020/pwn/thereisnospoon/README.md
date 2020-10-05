# thereisnospoon

The main idea in this problem, is linux read syscall's attribute.
```
...
read() attempts to read up to count bytes from file descriptor fd into the buffer starting at buf.(including '\x00')
...
```
strlen() will end when meet '\x00', therefore, strlen() and len can be different.

In this problem, we need to modify changeme value to get shell, but changeme is a malloc pointer, and is hard to modify.

We can find that 
```
...
len = read(0, buffer, len);

char * buffer2 = malloc(strlen(buffer));
...
len = read(0, buffer2, len); # if stdin include '\x00' len can be larger than strlen()
...
```
  
