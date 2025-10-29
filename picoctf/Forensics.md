# 1. Trivial Flag transfer Protocol 

> Figure out how they moved the flag.

## Solution:

- Loaded the file on wireshark and extracted all the files from `TFTP` 

<img width="1280" height="832" alt="Screenshot 2025-10-29 at 9 12 01 PM" src="https://github.com/user-attachments/assets/46bd73d1-c804-47c2-8f9e-faa41cf1ff92" />

- There were 2 encrypted .txt files instruction and plan used an online ROT13 decoder to decode the files

<img width="1280" height="832" alt="Screenshot 2025-10-29 at 9 13 40 PM" src="https://github.com/user-attachments/assets/e7ac394e-7381-445c-9c60-eebf9f18ba7e" />

- The challenge wanted me to install the program and run stegify with the password `DUEDILIGENCE` 

<img width="1280" height="832" alt="Screenshot 2025-10-29 at 9 15 20 PM" src="https://github.com/user-attachments/assets/bf6d60ff-7cdd-47c5-87bf-b63006bc7cc0" />

- Started using the program with the correct syntax which i got from the manual 
```
root@cc71042c3bbd:/shared# steghide extract -sf picture3.bmp -p DUEDILIGENCE
wrote extracted data to "flag.txt".
root@cc71042c3bbd:/shared# cat flag.txt
picoCTF{h1dd3n_1n_pLa1n_51GHT_18375919}
```
- The data was pasted in a file called `file.txt` read the file using `cat` to obtain the flag.

## Flag:

```
picoCTF{h1dd3n_1n_pLa1n_51GHT_18375919}
```

## Concepts learnt:

- Understanding basic operation of Wireshark
- Stegangraphy using stegihide used to hide textual data inside an image.


## Notes:

- NONE

## Resources:

- https://www.youtube.com/watch?v=qTaOZrDnMzQ&t=209s


***

