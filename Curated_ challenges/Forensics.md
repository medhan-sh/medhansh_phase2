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

# 2. Challenge name

> Description

.
.
.
