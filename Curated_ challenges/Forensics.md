# 1. Hide n Seek 

> Description:

Sakamoto’s at it again with a game of hide and seek, but this time, it’s not with Shin or his daughter. An old friend hid some secret data in this image. Can you find it before the others do?

Hint:
Even in retirement, Sakamoto never loses at hide and seek. Maybe stegseek can help you keep up.

## Solution:

- First opened an Ubuntu docker and set up stegseek.
<img width="1280" height="832" alt="Screenshot 2025-12-07 at 10 35 08 AM" src="https://github.com/user-attachments/assets/785ce6fc-c085-4366-966c-91c5adb3e29b" />
- Simply used stegseek with a wordlist to extractfrom the given image 
```
root@17ce5ffabde1:/ctf# stegseek sakamoto.jpg rockyou.txt
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "iloveyou1"
[i] Original filename: "flag.txt".
[i] Extracting to "sakamoto.jpg.out".

root@17ce5ffabde1:/ctf# cat sakamoto.jpg.out
nite{h1d3_4nd_s33k_but_w1th_st3g_sdfu9s8}
root@17ce5ffabde1:/ctf# 

```
## Flag:

```
nite{h1d3_4nd_s33k_but_w1th_st3g_sdfu9s8}
```

## Concepts learnt:

- Using Stegseek

## Notes:

- On my system i was facing difficulty loading stegseek directly so tried steghide but had no luck.

## Resources:

- Include the resources you've referred to with links. [example hyperlink](https://google.com)


***

# 2.Nutrela Chunks

> One of my favorite foods is soya chunks. But as I was enjoying some Nutrela today, I noticed a few chunks weren’t quite right. Seems like something’s off with their structure. Could you help me fix these broken chunks so I can enjoy my meal again?

## Solution:

- This challenge had a broken PNG file I tried fixing it manually on hexedit, but that was no help

<img width="1280" height="832" alt="Screenshot 2025-12-07 at 11 05 28 AM" src="https://github.com/user-attachments/assets/b39e0897-6c7e-4623-a479-a840a9fe3fae" />

 - The root problem were that all the terms were lowercase .PNG is case sensitive.
 - Used a tool pngcheck to understand the problem.
<img width="1280" height="832" alt="Screenshot 2025-12-07 at 11 08 20 AM" src="https://github.com/user-attachments/assets/243bd2bf-df4f-4743-b063-2a31dbe2814b" />

- Then used a python program to fix each position, it checks if the next 4 bytes spell 'idat' 

>If yes: it prints where it found it
>Replaces those 4 bytes with 'IDAT' (uppercase)
>Increments the counter

>It does the same check for 'iend' and 'ihdr'.

- The loaded the file to hexedit again fixed the header and obtained the flag.


## Flag:

<img width="1000" height="1000" alt="fixed_nutrela-2" src="https://github.com/user-attachments/assets/a1d77d65-c942-461b-9867-202b35acf299" />


## Concepts learnt:

- File structures and Magic numbers
- Hex editing
- PNG files are organized into chunks (IHDR, IDAT, IEND) with Structure: [Length][Chunk Type][Data][CRC].


## Notes:

- Include any alternate tangents you went on while solving the challenge, including mistakes & other solutions you found.
- 

## Resources:

- Include the resources you've referred to with links. [example hyperlink](https://google.com)


***


