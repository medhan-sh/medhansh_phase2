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
- After confirming it was an ELF 64-bit executable, I launched an Ubuntu x86-64 Docker container and installed neccesary dependencies to use `file` and `string`. I reffered to Gemini for the syntax.
<img width="1280" height="832" alt="Screenshot 2025-10-22 at 8 01 17 PM" src="https://github.com/user-attachments/assets/646391e1-4019-4f68-b0b5-3106d911a74a" />

- I made the file executable using `chmod +x` but that was no help so as per the hint I installed `gdb` the syntax for which i got from a tutorial video I got on youtube about `gdb` the link to which I have provided in the refferences

```
root@cc71042c3bbd:/shared# apt-get install gdb
```
- The tutorial crafted my thought process i first used `info funtion` to analyse all the functions inside the file, there I found `main` function as instructed in the challenge.

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

<img width="1280" height="832" alt="Screenshot 2025-10-22 at 8 29 12 PM" src="https://github.com/user-attachments/assets/cf435b28-7946-46e1-9462-b1ce055075fb" />


## Flag:

```
picoCTF{549698}
```

## Concepts learnt:

- Basics of GDB workflow - Employing basic functions like `info function` and `disassemble` to understand assembly instructions
- GDB is a debugger which lets me stop the program at specific points and lets us examine what is happening at that exact moment.
- `disassemble` lets you read what exactly the program is telling the CPU to do i.e translating the low-level code to assembly which can be interepreted by us
- I dont really understand the output but the challenge wanted me to find `eax` which i found just lying there.
- Coverting Hex to decimal .

## Notes:

- rosetta error- the file was in `x86-64` and i use a mac so naturally i ran an ARM version of ubuntu on my terminal which gave me this error i fixed it by using the `x86-64` version of Ubuntu 

## Resources:

- Reverse Engineering -GDB( GNU debugger) by IronByte - https://www.youtube.com/watch?v=1iptoUKXrcI
- Geminini


***

# 2. ARMssembly 1

> For what argument does this program print `win` with variables 81, 0 and 3? File: chall_1.S Flag format: picoCTF{XXXXXXXX} -> (hex, lowercase, no 0x, and 32 bits. ex. 5614267 would be picoCTF{0055aabb})

## Solution:

- Used `file` to understand the type of the file then used `cat` to read the contents of the file .
- The file was in Assembler Source used the video given in refference to understand the assembly code
```
main:
	stp	x29, x30, [sp, -48]!
	add	x29, sp, 0
	str	w0, [x29, 28]
	str	x1, [x29, 16]
  ldr	x0, [x29, 16]
	add	x0, x0, 8
	ldr	x0, [x0]
	bl	atoi
	str	w0, [x29, 44]
```
- This program first stores stores our input by creating an array of string pointer `argv` then it skips argv[0] by adding `8` which is the no. of bytes in a pointer
- The string is then coverted to integer using `atoi`

 ```
	ldr	w0, [x29, 44]
	bl	func
	cmp	w0, 0
	bne	.L4
	adrp	x0, .LC0
	add	x0, x0, :lo12:.LC0
	bl	puts
	b	.L6
  ```
  - This calls the function `func` and checks for result if the output is 0 we win.
```
func:
	sub	sp, sp, #32
	str	w0, [sp, 12]
	mov	w0, 81
	str	w0, [sp, 16]
	str	wzr, [sp, 20]
	mov	w0, 3
	str	w0, [sp, 24]
	ldr	w0, [sp, 20]
	ldr	w1, [sp, 16]
	lsl	w0, w1, w0
	str	w0, [sp, 28]
	ldr	w1, [sp, 28]
	ldr	w0, [sp, 24]
	sdiv	w0, w1, w0
	str	w0, [sp, 28]
	ldr	w1, [sp, 28]
	ldr	w0, [sp, 12]
	sub	w0, w1, w0
	str	w0, [sp, 28]
	ldr	w0, [sp, 28]
	add	sp, sp, 32
	ret
	.size	func, .-func
	.section	.rodata
	.align	3

```
- The function already has 81,3,0 hardwired in the program and it is employing our input.
- It performs three mathematical operation in the fllowing order

```
ldr	w0, [sp, 20]
ldr	w1, [sp, 16]
lsl	w0, w1, w0
str	w0, [sp, 28]
```
```
ldr	w1, [sp, 28]
ldr	w0, [sp, 24]
sdiv	w0, w1, w0
str	w0, [sp, 28]
```
```
ldr	w1, [sp, 28]
ldr	w0, [sp, 12]
sub	w0, w1, w0
str	w0, [sp, 28]
```
- The first operation loads the two numbers stored at `w0, [sp, 20]`=0 and `w1, [sp, 16]`=81 and cheks for `81>>0` and then stores the output at `w0, [sp, 28]`.
- Simmilarly the next operation loads the previous output and divides it by `w0, [sp, 24]`=3
- The program then subtracts our input from the output if the answer is 0 we win 
Therefore:
(81>>0)/3 - input=0
27 - input = 0
`input=27`
- As per the instruction in the challenge changed it to hex to get `0000001b` employed that in the format to obtain the flag.

<img width="1280" height="832" alt="Screenshot 2025-10-24 at 6 26 27 PM" src="https://github.com/user-attachments/assets/763d625c-c336-4628-8d26-b51a5d65e4eb" />

## Flag:

```
picoCTF{0000001b}
```

## Concepts learnt:

- Understanding ARM64 assembly instruction .
- `w0` in assembly are temporary registers and they can be reused multiple times
-  Common Operations:
Left shift (lsl): Bitwise operation (multiply by powers of 2)
Division (sdiv): Signed integer division
Subtraction (sub): Basic arithmetic
Comparison (cmp): Setting flags for branching
- Understanding the `.S` file format. 
## Notes:

- Followed the vid given in refference.

## Resources:

- https://www.youtube.com/watch?v=1d-6Hv1c39c- simmilar to this challenge
- Took help of claude AI to understand the assembly instruction.


***

# 3. VaultDoor 3 

> This vault uses for-loops and byte arrays. The source code for this vault is here: VaultDoor3.java

## Solution:

- Opened the java file on VS code 
<img width="1280" height="832" alt="Screenshot 2025-10-24 at 7 16 58 PM" src="https://github.com/user-attachments/assets/8017023f-2984-4f2b-b153-ca6e1b4a1b07" />

- The code first deletes picoCTF{} from the input .
- Then it checks if the input is 32 characters then scrambles it according to the `for` loops.

```
for (i=0; i<8; i++) {
buffer[i] = password.charAt(i);
}
for (; i<16; i++) {
buffer[i] = password.charAt(23-i);
}
for (; i<32; i+=2) {
buffer[i] = password.charAt(46-i);
}
for (i=31; i>=17; i-=2) {
buffer[i] = password.charAt(i);
 }
```
- Made an Excel sheet corresponding to the `i` values given in the code.

<img width="1280" height="832" alt="Screenshot 2025-10-24 at 7 09 19 PM" src="https://github.com/user-attachments/assets/0c0179a5-b518-4e3c-8c60-70585ec01c76" />

- Manually unscrambled the password by replacing the characters according to the table.


## Flag:

```
picoCTF{jU5t_a_s1mpl3_an4gr4m_4_u_1fb380}
```

## Concepts learnt:

- Static Analysis - Reading the code without running it 
- working backwords to achieve the desired condition.

## Notes:

- Did not account that `i` started from `0` not `1` so messed up the counting a little getting the wrong flag.

## Resources:

NONE


***











