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
- After analysing that this is an `ELF` file of `x86-64 architecture i started a docker container running ubuntu x86-64 and insstalled the neccesary dependencies 
<img width="1280" height="832" alt="Screenshot 2025-10-22 at 8 01 17 PM" src="https://github.com/user-attachments/assets/646391e1-4019-4f68-b0b5-3106d911a74a" />
- I ran

```
put codes & terminal outputs here using triple backticks

you may also use ```python for python codes for example
```

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
