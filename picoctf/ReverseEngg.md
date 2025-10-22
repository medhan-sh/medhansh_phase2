# 1. GDB baby step 1

> Can you figure out what is in the eax register at the end of the main function? Put your answer in the picoCTF flag 
format: picoCTF{n} where n is the contents of the eax register in the decimal number base. If the answer was 0x11 your 
flag would be picoCTF{17}.
Disassemble this.

## Solution:

- I first checked for the type of file `debugger0_a` was using the command `file` 
```
  vishalsharan@Vishals-MacBook-Air Downloads % file debugger0_a
debugger0_a: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=15a10290db2cd2ec0c123cf80b88ed7d7f5cf9ff, for GNU/Linux 3.2.0, not stripped
```
- After analysing that this is an `ELF` file of `x86-64` architecture i started a docker container running ubuntu x86-64 and insstalled the neccesary dependencies to use the command `string` the syntax for which i got from gemini after i asked it why wasn't the command working 
<img width="1280" height="832" alt="Screenshot 2025-10-22 at 8 01 17 PM" src="https://github.com/user-attachments/assets/646391e1-4019-4f68-b0b5-3106d911a74a" />
- Using strings was no help so as per the hint i installed `gdb` the syntax for which i got from a tutorial video i got on youtub about `gdb` the link to which i have provided in the refferences
```
root@cc71042c3bbd:/shared# apt-get install gdb
```
- The tutorial crafted my thought process i first used `info funtion` to analyse all the functions inside the file there I found `main` function as instructed in the challenge.
<img width="399" height="214" alt="Screenshot 2025-10-22 at 8 19 05 PM" src="https://github.com/user-attachments/assets/6ac5354b-841f-4c67-a1c3-91b92cc4b991" />

- Then used the command `disassemble main` to disassemble the main function

```
(gdb) disassemble main 
Dump of assembler code for function main:
   0x0000000000001129 <+0>:	endbr64
   0x000000000000112d <+4>:	push   %rbp
   0x000000000000112e <+5>:	mov    %rsp,%rbp
   0x0000000000001131 <+8>:	mov    %edi,-0x4(%rbp)
   0x0000000000001134 <+11>:	mov    %rsi,-0x10(%rbp)
   0x0000000000001138 <+15>:	mov    $0x86342,%eax
   0x000000000000113d <+20>:	pop    %rbp
   0x000000000000113e <+21>:	ret
End of assembler dump.

```
- Over there I found the eax required to solve the challenge the number was in hexadecimal so coverted it to decimal to decimal which gave the number `549698` Employed the number in the flag format as instructed to solve the challenge.

## Flag:

```
picoCTF{549698}
```

## Concepts learnt:

- Include the new topics you've come across and explain them in brief
- 

## Notes:

- Include any alternate tangents you went on while solving the challenge, including mistakes & other solutions you found.
- 

## Resources:

- Include the resources you've referred to with links. [example hyperlink](https://google.com)


***

# 2. Challenge name

> Description

.
.
.

