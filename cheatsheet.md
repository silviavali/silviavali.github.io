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


[GDB reference](/cheatsheet/#gdb-reference)<br>

Exercises:

* [Hello World! in .data section](/cheatsheet/#hello-world-in-data-section)
* [Hello World! JCP technique](/cheatsheet/#hello-world-jcp-technique)
* [Execve](/cheatsheet/#execve)
* [Buffer overflow](/cheatsheet/#buffer-overflow)

Homework:

* [Homework](/cheatsheet/#homework) - deadline 1st of December, 23:59

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


## [Hello World! in data section](#hello-world-in-data-section)

```
; ssize_t write(int fd, const void *buf, size_t count);
; Write a program which displays Hello World

;STEP 1 - global directive, program entry point, sections
global _start:
section .text

_start:
;STEP 2 - fd needs to be set to 1 (stdout)
	mov dil, 0x1

;STEP 3 - pointer to a buffer needs to point to a string we want to print to the stdout
	mov rsi, message
;STEP 4 - count needs to be set to number of bytes we want to print to stdout "Hello World!" + new line = 13 dec or 0xd in hex
	add rdx, 0xd

;STEP 5 - RAX needs to hold the syscall number we want to execute, call syscall
	add rax, 0x1
	syscall

;STEP 6 - exit() syscall
	xor rdi, rdi
	mov al, 0x3c
	syscall

section .data
	message: db 'Hello World!', 0xa
```

## [Hello World JCP technique](#hello-world-jcp-technique)

2.1 with NULL bytes
```
; JMP-CALL-POP technique
; Write a program to display Hello World!
; 0xa - creates a new line character after the Hello World! string
; remove null bytes

global _start
section .text

_start:

	jmp callshell

shellcode:
	pop rsi
	mov rdx, 0xd
	add rdi, 1

	mov rax, 0x1
	syscall

exit:
	mov rax, 0x3c
	syscall

callshell:
	call shellcode

msg: db 'Hello World!', 0xa

```

2.2 without NULL bytes, but it doesn't work in our test shellcode program, why? Find out in GDB, that some registers do not have expected values before the 1st syscall is executed.
```
; JMP-CALL-POP technique
; Write a program to display Hello World!
; 0xa - creates a new line character after the Hello World! string
; remove null bytes

global _start
section .text

_start:

	jmp callshell

shellcode:
	pop rsi
	mov dl, 0xd
	add dil, 1

	mov al, 0x1
	syscall

exit:
	mov al, 0x3c
	syscall

callshell:
	call shellcode

msg: db 'Hello World!', 0xa

```

2.3 Final shellcode using JCP technique:
```
; JMP-CALL-POP technique
; Write a program to display Hello World!
; 0xa - creates a new line character after the Hello World! string
; remove null bytes

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

	xor rax, rax
	mov al, 0x1
	syscall

exit:
	mov al, 0x3c
	syscall

callshell:
	call shellcode

msg: db 'Hello World!', 0xa
```

## [Execve](#execve)

```
global _start
section .text

_start:
	; int execve(const char *filename, char *const argv[],
        ; char *const envp[]);

	mov rbx, 0x68732f6e69622f2f
	xor rdx, rdx
	push rdx
	push rbx
	mov rdi, rsp

	push rdx
	push rdi
	mov rsi, rsp

	;syscall
	xor rax, rax
	mov al, 0x3b
	syscall
```

## [Buffer overflow](#bof)

**Disable ASLR:**<br>
`echo 0 | sudo tee /proc/sys/kernel/randomize_va_space`

**Compile the file**<br>
`gcc -fno-stack-protector -z execstack buffer.c -o buffer`

Commands:<br>

* nasm -felf64 shellcode.nasm -o shellcode.o
* ld shellcode.o -o shellcode
* gethex shellcode
* in GDB to calculate the number of bytes between 2 addresses `p/d 0xXXXXXXX-0xYYYYYYYY`

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

### Solution path

**Step 1**:<br>

* Shellcode, when run independently, should print out your name/nickname as follows:<br>
```
$ ./name
Silvia
```

* Use `objdump -d -M intel <filename>` to verify that your shellcode does not have any NULL bytes
* `gethex <filename>` to dump the shellcode as a hex string, to use in the exploit in step 2


**Step 2**:<br>

* Use the dumped shellcode in your payload to exploit the vulnerable buffer.c program

Rough bulletpoints what you should know to complete the task:<br>

- How is the user input consumed in the vulnerable program?
- Vulnerability. Which buffer is overflown?
	- Start address of that buffer?
- Address of the RIP to overwrite?

* Vulnerable program "buffer" should produce the following output when run:<br>
```
$ ./buffer
0x7fffffffdbb0
Silvia
```

**Step 3: Present your homework**

**Deadline (strict)**: 01.12.2019, 23.59<br>
**Send it to**: <silviavali14[at]gmail[dot]com><br>
**Title of email**: Student code, firstname lastname<br>

Email should contain:<br>

* Source code of the shellcode that prints out your name/nickname
* Source code of exploit code used to complete step 2

Rules !!!<br>
1. Any homework that does not arrive in my mailbox by the set deadline is counted as not solved.
2. If you submit your homework multiple times, last one is the one that counts.
3. In order to complete the homework, both tasks (shellcode and buffer overflow exploitation) have to be done and sent to my e-mail. No points are given for half a solution. It either works and you get 10 points or it doesn't and you get a 0.

*[FLIP]: First-Last-Invert-Play. A coding technique to achieve performant page transition animations.
