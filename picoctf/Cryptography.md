# 1. rsa_oracle

> Can you abuse the oracle?
An attacker was able to intercept communications between a bank and a fintech company. They managed to get the message (ciphertext) and the password that was used to encrypt the message.
After some intensive reconassainance they found out that the bank has an oracle that was used to encrypt the password and can be found here nc titan.picoctf.net 64803. Decrypt the password and use it to decrypt the message. The oracle can decrypt anything except the password.


## Solution:

- After launching the oracle used it to decrypt the value of `2` 

<img width="1280" height="832" alt="Screenshot 2025-10-27 at 12 01 09 AM" src="https://github.com/user-attachments/assets/6c564974-318d-480c-8479-04eccbe2a197" />

- The oracle would decrypt anything but the password so i multiply the password to a number then divide by the same number after decrypting to obtain the key.
```
2336150584734702647514724021470643922433811330098144930425575029773908475892259185520495303353109615046654428965662643241365308392679139063000973730368839root@cc71042c3bbd:/shared# cat secret.enc
Salted__jX???9V)?pT3Ͼq?$?qEs?5?n\K5=?%c??	?Q?9s?1!quM?0??4??
```
- So i ran a python program to multiply the two big integers.
```
vishalsharan@Vishals-MacBook-Air ~/Downloads % python3 
Python 3.9.6 (default, Aug  8 2025, 19:06:38) 
[Clang 17.0.0 (clang-1700.3.19.1)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> num1=4707619883686427763240856106433203231481313994680729548861877810439954027216515481620077982254465432294427487895036699854948548980054737181231034760249505
>>> num2=2336150584734702647514724021470643922433811330098144930425575029773908475892259185520495303353109615046654428965662643241365308392679139063000973730368839
>>> result=num1*num2
>>> print(result)
10997708943982761084006315359417483254965299487204584192712335192036789472336196626179282134890223733758401125471056267054908321079024432384222437910457194483711112753102678178170094968585207806212096960492328042941752878907452001886104974213833155189826877814877017136978779880432127774578986380439317174695
```
- I decrypted this number using the oracle the output needed to be turned from hex to decimal then divided by `50` which is the ascii value of `2` so used a python program to do that and obtain the key .

```
>>> import binascii
>>> hex_result = "a9573f66360"
>>> num_result = int(hex_result, 16)
>>> password_num = num_result // 50
>>> password_hex = hex(password_num)[2:]
>>> password = binascii.unhexlify(password_hex)
>>> print(password)
b'60f50'

```
- As instructed in the challenege used `openssl enc -aes-256-cbc -d -in secret.enc` to decrypt the text and obtained the flag 

<img width="1280" height="832" alt="Screenshot 2025-10-27 at 12 08 38 AM" src="https://github.com/user-attachments/assets/feacd756-f1e4-45a6-80e1-424db1771ae9" />


## Flag:

```
picoCTF{su((3ss_(r@ck1ng_r3@_60f50766}
```

## Concepts learnt:

- Include the new topics you've come across and explain them in brief
- 

## Notes:

- RSA encryption has a special mathemathical property the encryption of (Message 1 × Message 2) is equal to (Encryption of Message 1 × Encryption of Message 2).
- Basic application of python
- Understanding different data encoding the output the decrypter gave was in hex which needed to be cahnged to decimal using python and also when I provided the input `2` its read it as teh ascii character `50`.

## Resources:

- Gemini AI helped me with understanding Python
- https://www.youtube.com/watch?v=hm8s6FAc4pg&t=163s- used this video to understand what rsa is.


***

# 2. miniRSA

> Let's decrypt this: ciphertext? Something seems a bit small.

## Solution:

- Include as many steps as you can with your thought process
- You **must** include images such as screenshots wherever relevant.

```
put codes & terminal outputs here using triple backticks

you may also use ```python for python codes for example
```

## Flag:

```
picoCTF{}
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

