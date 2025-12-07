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
# 3. RAR of the abyss

> Description:

Two philosophers peer into the networked abyss and swap a secret. Use the secret to decrypt the Abyss’ RAwR and pull your flag from the void.

## Solution:

- First, we opened abyss.pcap in Wireshark to analyze the network traffic.

<img width="1280" height="832" alt="Screenshot 2025-12-07 at 11 48 30 AM" src="https://github.com/user-attachments/assets/42ba6667-c152-4470-874a-96a138753fb7" />

- Looked for the protocol hierarchy to find
TCP traffic (the main communication)
ICMP (ping packets)
UDP (DNS, SSDP)
- Followed the TCP stream to find hex-encoded data, we converted them to reveal
  stream 0 : "Camus: One must imagine Sisyphus happy but are we happy ?"
  stream 1 : "Nietzsche: You will be happy after reading my latest work"
  stream 2 : This starts with "Rar!" - indicating a RAR archive file
  stream 3 : "Camus: whats the password ?"
  stream 4 : "Nietzsche: b3y0ndG00dand3vil"
  stream 5 : "Camus: thanks"
- Stream 2 had the RAR file and stream 4 had the passphrase, we extracted the RAR file using the tool unrar.
<img width="1280" height="832" alt="Screenshot 2025-12-07 at 11 52 34 AM" src="https://github.com/user-attachments/assets/a9afa27f-9947-41b1-a212-1cd65f4f9229" />

- extracted into a file RAR.txt


## Flag:

```
nite{thus_sp0k3_th3_n3tw0rk_f0r3ns1cs_4n4lyst}
```

## Concepts learnt:

- Network Traffic analysis : Using Wireshark to inspect captured network packets, following TCP streams to reconstruct conversations.
- Hex Encoding/Decoding
- RAR files start with 52 61 72 21 ("Rar!")
- The "two philosophers" conversation contained the password in plaintextStream 4 revealed: b3y0ndG00dand3vil
- Tools Used

Wireshark - GUI packet analyzer for visualizing network traffic
xxd - Hex dump utility for converting between hex and binary
unrar - Tool for extracting RAR archives
echo - Used to output hex strings for conversion
## Notes:

- Include any alternate tangents you went on while solving the challenge, including mistakes & other solutions you found.
- 

## Resources:

- Include the resources you've referred to with links. [example hyperlink](https://google.com)


***
# 4. ninetails

> Description:

Looks like I got a little too clever and hid the flag as a password in Firefox, tucked away like one of NineTails’ many tails. Recover the "logins" and the "key4" and let it guide you to the flag.

Hint:
I named my Ninetails "j4gjesg4", quite a peculiar name isn't it?

## Solution:

- Extrancting the rar file gave an `.ad1` file loaded that into a ftk imager.
![WhatsApp Image 2025-12-07 at 16 03 58](https://github.com/user-attachments/assets/154ccfe1-ee2a-454e-950a-d08330334d3f)

- We were looking for two files `login.json` and `keys.db` found 
![WhatsApp Image 2025-12-07 at 16 03 58 (1)](https://github.com/user-attachments/assets/6ec49bac-5bd0-4328-9b09-0b7b99fa7676)

- We used firepwd in the same directory as the two extracted files, a Python tool specifically designed to decrypt Firefox passwords.
```
git clone https://github.com/lclevy/firepwd.git
cd firepwd
```
- Creted a python virtual environment and downloaded the neccessary dependencies
<img width="1280" height="832" alt="Screenshot 2025-12-07 at 4 08 42 PM" src="https://github.com/user-attachments/assets/6cc7b968-fc47-4b7c-b1d4-61b2530bb62a" />
- Once firepwd successfully decrypted the files, it revealed several stored login credentials:
```
https://www.rehack.xyz: username='warlocksmurf', password='GCTF{m0zarella'
https://ctftime.org: username='ilovecheese', password='CHEEEEEEEEEEEEEEEEEEEEEEEEEESE'
https://www.reddit.com: username='bluelobster', password='_f1ref0x_'
https://www.facebook.com: username='flag', password='SIKE'
https://warlocksmurf.github.io: username='Man I Love Forensics', password='p4ssw0rd}'
```

## Flag:

```
GCTF{m0zarella_f1ref0x_p4ssw0rd}
```

## Concepts learnt:

- Firefox Password Storage: Firefox uses a two-file system (logins.json + key4.db) to store encrypted passwords
- Master Password: The key4.db encryption can be protected by a master password, which was hidden in the challenge hint
- Using firepwd
- Tools Used

FTK Imager (or similar) - For extracting files from the forensic image
firepwd - Python tool for decrypting Firefox passwords
Python libraries: pyasn1, cryptography, pycryptodome

## Notes:

- Include any alternate tangents you went on while solving the challenge, including mistakes & other solutions you found.
- 

## Resources:

- Include the resources you've referred to with links. [example hyperlink](https://google.com)


***




