# 1. Trivial Flag transfer Protocol 

> Figure out how they moved the flag.

## Solution:

- Loaded the file on wireshark and extracted all the files from `TFTP` 

<img width="1280" height="832" alt="Screenshot 2025-10-29 at 9 12 01 PM" src="https://github.com/user-attachments/assets/46bd73d1-c804-47c2-8f9e-faa41cf1ff92" />

- There were 2 encrypted .txt files instruction and plan used an online ROT13 decoder to decode the files

<img width="1280" height="832" alt="Screenshot 2025-10-29 at 9 49 51 PM" src="https://github.com/user-attachments/assets/316b198e-5ea5-4cc0-a720-1965a7e0cc8e" />

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
# 2. tunn3l v1s10n

> We found this file. Recover the flag.

## Solution:

- First used the command `file` to examine the program gave the output `data` 

```
vishalsharan@Vishals-MacBook-Air ~/Downloads % file /Users/vishalsharan/Downloads/tunn3l_v1s10n 
/Users/vishalsharan/Downloads/tunn3l_v1s10n: data
```

<img width="1280" height="832" alt="Screenshot 2025-10-31 at 10 20 52 PM" src="https://github.com/user-attachments/assets/0d30c089-9b04-4c5a-9971-71f4bc9ebb5b" />

- Opened the file on an online hex editor and found out the file started with 42 4d which is a `.bmp` file.
<img width="1280" height="832" alt="Screenshot 2025-10-31 at 10 25 55 PM" src="https://github.com/user-attachments/assets/589802ea-709d-4817-bedb-35b4f484d07f" />

- parts of the file's header, the metadata at the beginning of the file that describes the image, were incorrect.
- Manually fixed the file and exported it to get an image which contained the flag 
<img width="568" height="415" alt="Screenshot 2025-10-31 at 10 32 34 PM" src="https://github.com/user-attachments/assets/ad9819cf-9fa2-4a03-bcef-c7bd12d619ad" />


## Flag:

```
picoCTF{qu1t3_a_v13w_2020 }
```

## Concepts learnt:

- Introduction to a Hex editor
- File header magic numbers
- File repair 

## Notes:

- NONE

## Resources:

- https://www.youtube.com/watch?v=5Vh_YT-nexM


***
# 3. m00nwalk

> Decode this message from the moon.

## Solution:

- The hint in this challenge was really helpful a few google searches gave me the solution first opened the file in davinci to observe the audio file but that was no help so started looking on how did we recieve images from the moon landing gave me teh hint for using a SStv decoder 

<img width="1280" height="832" alt="Screenshot 2025-10-30 at 10 25 11 PM" src="https://github.com/user-attachments/assets/a7ccd21f-1adb-40db-9a20-567a91103adb" />

- Used an online SStv decoder to extract the images from the `.wav` file

<img width="1280" height="832" alt="Screenshot 2025-10-30 at 10 27 39 PM" src="https://github.com/user-attachments/assets/532f60a3-834b-48cb-a100-0e7ba9c6ae35" />

- Copied the flag from there.

## Flag:

```
picoCTF{beep_boop_im_in_Space}
```

## Concepts learnt:

- Steganography tools like SSTV decoder
- Audio analysis

## Notes:

- NONE

## Resources:

- Google
<img width="1280" height="832" alt="Screenshot 2025-10-30 at 10 34 44 PM" src="https://github.com/user-attachments/assets/7003da12-9c32-4d87-b3cd-46facb0092f4" />


***







