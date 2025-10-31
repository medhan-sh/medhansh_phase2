# 1. IQ test 

> let your input x = 30478191278.

wrap your answer with nite{ } for the flag.

As an example, entering x = 34359738368 gives (y0, ..., y11), so the flag would be nite{010000000011}.

## Solution:

- Changed the decimal to binary first using an online tool.

<img width="1280" height="832" alt="Screenshot 2025-10-31 at 8 18 07 PM" src="https://github.com/user-attachments/assets/ba1933d4-4bb2-4fff-b6f1-1cdfb11e131b" />

- Took a printout the the png file to manually solve the problem 

![WhatsApp Image 2025-10-31 at 20 20 02](https://github.com/user-attachments/assets/54b56542-3c3c-4fda-95bb-3b87bf1144f3)

## Flag:

```
niteCTF{100010011000}
```

## Concepts learnt:

- Logic gates

## Notes:

- NONE

## Resources:

- NONE


***

# 2. I like logic 

> i like logic and i like files, apparently, they have something in common, what should my next step be.


## Solution:

- Opened the file on `Logic2`
<img width="1280" height="832" alt="Screenshot 2025-10-31 at 11 29 02 PM" src="https://github.com/user-attachments/assets/7821195b-9f36-41d6-bbbf-1796c0a1aa26" />
- Started using random analyser till i get an output used `SPI` and `Async Serial` 
<img width="1280" height="832" alt="Screenshot 2025-10-31 at 11 31 41 PM" src="https://github.com/user-attachments/assets/f881a640-73ce-4369-ba15-f0dec7a01b3e" />

- On the terminal there was the output from there retrieved the flag.

## Flag:

```
FCSC{b1dee4eeadf6c4e60aeb142b0b486344e64b12b40d1046de95c89ba5e23a9925}

```

## Concepts learnt:

- Using Logic2

## Notes:

- None

## Resources:

- None


***

# 3. Bare metal Alchemist

> my friend recommended me this anime but i think i've heard a wrong name.

## Solution:

- Started by reading or executing the file which was a big waste of time then loaded the file on `r2`

<img width="1280" height="832" alt="Screenshot 2025-10-31 at 11 55 57 PM" src="https://github.com/user-attachments/assets/320bb070-3850-4e88-9246-69162e4736b2" />

- First listed all the functions to look for some sort of `main` or `decrypt_flag` kinda function. 
```
[0x00000000]> afl
0x00000000    8     62 entry0
0x000000d4    1     14 sym.z1__
0x00000176   38    362 main
0x000000e2    4    148 sym.__vector_16
```
- Shifted to main using `s main`.|
- Ran a decompiler but it was kinda confused coz code was in assembly.

<img width="1280" height="832" alt="Screenshot 2025-10-31 at 11 58 19 PM" src="https://github.com/user-attachments/assets/93fe53c1-fa6c-4740-8e4e-ee465e21ed26" />

- The orphan loop is where the decryption was happening and `0x68` was the address were the decryption was being stored so tried printing that but failed.

<img width="1280" height="832" alt="Screenshot 2025-10-31 at 11 59 58 PM" src="https://github.com/user-attachments/assets/c969adcd-1834-4d44-8a20-e355cba03341" />

- Then used `psx 44 @ 0x68` which would print 44 bytes of the string
- Feeded the output to a python decrypter script to obtain the flag.

```
vishalsharan@Vishals-MacBook-Air Downloads % python3
Python 3.9.6 (default, Aug  8 2025, 19:06:38) 
[Clang 17.0.0 (clang-1700.3.19.1)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> encrypted_flag = b'\xf1\xe3\xe6\xe6\xf1\xe3\xdf\xf1\xcd\x94\xd6\xfa\x94\xd6\xfa\xd6\xca\xc8\x97\xfa\xd6\x94\xc8\xd5\xc9\x03\xfa\x91\xd7\xc1\xd0\x94\xcb\xca\xfa\xc3\x94\xd7\xc8\xd2\x91\xd7\xc0\xd8'
>>> key = 0xa5
>>> flag = ""
>>> for byte in encrypted_flag:
...   flag += chr(byte ^ key)
... 
>>> print(flag)
```


## Flag:

```
FCCTF{Th1s_1s_som3_s1mpl3_4rdu1no_f1rmw4re}

```

## Concepts learnt:

- Using the radare2 tool
- Xor encryption
- decompilation processes
- python scripting for decompiling 

## Notes:

- reading the files was a waste of time
- using ` ps @ 0x68` command failed because it's looking for a standard, null-terminated (text) string. Your encrypted flag is just raw bytes, and it probably contains a \x00 (null) byte, which makes ps stop immediately.

## Resources:

- Gemini


***

