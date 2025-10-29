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

- RSA encryption has a special mathemathical property the encryption of (Message 1 × Message 2) is equal to (Encryption of Message 1 × Encryption of Message 2).
- Basic application of python
- Understanding different data encoding the output the decrypter gave was in hex which needed to be cahnged to decimal using python and also when I provided the input `2` its read it as teh ascii character `50`.

## Notes:

- NONE

## Resources:

- Gemini AI helped me with understanding Python
- https://www.youtube.com/watch?v=hm8s6FAc4pg&t=163s- used this video to understand what rsa is.


***

# 2. miniRSA

> Let's decrypt this: ciphertext? Something seems a bit small.

## Solution:

- First made the file executable but executing the file did nothing so used `cat` to read the file
```
vishalsharan@Vishals-MacBook-Air ~/Downloads % chmod +x ciphertext 
vishalsharan@Vishals-MacBook-Air ~/Downloads % ./ciphertext 
./ciphertext: line 2: N:: command not found
./ciphertext: line 3: e:: command not found
./ciphertext: line 5: syntax error near unexpected token `c'
./ciphertext: line 5: `ciphertext (c): 2205316413931134031074603746928247799030155221252519872649649212867614751848436763801274360463406171277838056821437115883619169702963504606017565783537203207707757768473109845162808575425972525116337319108047893250549462147185741761825125 '
vishalsharan@Vishals-MacBook-Air ~/Downloads % cat ciphertext 

N: 29331922499794985782735976045591164936683059380558950386560160105740343201513369939006307531165922708949619162698623675349030430859547825708994708321803705309459438099340427770580064400911431856656901982789948285309956111848686906152664473350940486507451771223435835260168971210087470894448460745593956840586530527915802541450092946574694809584880896601317519794442862977471129319781313161842056501715040555964011899589002863730868679527184420789010551475067862907739054966183120621407246398518098981106431219207697870293412176440482900183550467375190239898455201170831410460483829448603477361305838743852756938687673
e: 3

ciphertext (c): 2205316413931134031074603746928247799030155221252519872649649212867614751848436763801274360463406171277838056821437115883619169702963504606017565783537203207707757768473109845162808575425972525116337319108047893250549462147185741761825125
```
- The value of `e` is very small in this so if we take the assumption that `m` is significantly smaller, if `m^3` is significantly smaller than `N` then according to Rsa algorithm `c=m^e mod(N)`, `c` would become `m^3`
- Made a python program that would find an integer cube root of `c` turns it int hex, unhexlify it then decode it.

<img width="1280" height="832" alt="Screenshot 2025-10-28 at 2 34 06 PM" src="https://github.com/user-attachments/assets/4f47d5d9-4112-4c28-b63b-964000563e52" />

- Ran the script to obtain the flag 

<img width="1280" height="832" alt="Screenshot 2025-10-28 at 2 34 40 PM" src="https://github.com/user-attachments/assets/2b9cd90f-1b17-4b34-8759-b0ad90399b25" />


## Flag:

```
picoCTF{n33d_a_lArg3r_e_606ce004}
```

## Concepts learnt:

- Vulnerabilty in having a very small `e` values in RSA encryption without any padding.
- To find integer value cube roots the best way is to employ binary search because mathematical functions would give float point value which are not ideal for this case.

## Notes:

- While creating the python program I did run into some indentation errors.

## Resources:

- Gemini helped with the python program


***
# 3. Custom Encryption 

> Can you get sense of this code file and write the function that will decode the given encrypted file content.
Find the encrypted file here flag_info and code file might be good to analyze and get the flag.


## Solution:

- Understanding the encryption was a huge task took claude's help to understand the encryption the encryption works in three steps
  1. Reversing the string
  2. XOR encrypt
  3. Multiplication with a special number and key


```
from random import randint
import sys


def generator(g, x, p):
    return pow(g, x) % p


def encrypt(plaintext, key):
    cipher = []
    for char in plaintext:
        cipher.append(((ord(char) * key*311)))
    return cipher


def is_prime(p):
    v = 0
    for i in range(2, p + 1):
        if p % i == 0:
            v = v + 1
    if v > 1:
        return False
    else:
        return True


def dynamic_xor_encrypt(plaintext, text_key):
    cipher_text = ""
    key_length = len(text_key)
    for i, char in enumerate(plaintext[::-1]):
        key_char = text_key[i % key_length]
        encrypted_char = chr(ord(char) ^ ord(key_char))
        cipher_text += encrypted_char
    return cipher_text


def test(plain_text, text_key):
    p = 97
    g = 31
    if not is_prime(p) and not is_prime(g):
        print("Enter prime numbers")
        return
    a = randint(p-10, p)
    b = randint(g-10, g)
    print(f"a = {a}")
    print(f"b = {b}")
    u = generator(g, a, p)
    v = generator(g, b, p)
    key = generator(v, a, p)
    b_key = generator(u, b, p)
    shared_key = None
    if key == b_key:
        shared_key = key
    else:
        print("Invalid key")
        return
    semi_cipher = dynamic_xor_encrypt(plain_text, text_key)
    cipher = encrypt(semi_cipher, shared_key)
    print(f'cipher is: {cipher}')


if __name__ == "__main__":
    message = sys.argv[1]
    test(message, "trudeau")

```
- To solve this I needed to decrypt the cipher in an descending way this was encrypted meaning i had to first reverse teh multiplication 

<img width="1280" height="832" alt="Screenshot 2025-10-29 at 1 22 46 PM" src="https://github.com/user-attachments/assets/c59ebed7-f082-4495-8198-68ca8d97b7f3" />

- Created a function `decrypt_multiplication` which reverses the multiplication encryption by dividing each cipher number by (key × 311) to recover ASCII values, then converts them to characters
```
def decrypt_multiplication(cipher,key) :
    plaintext=""
    for num in cipher:
        original_ord = num // (311 * key) 
        plaintext+=chr(original_ord)
    return plaintext 
```
- The output is then fed to a function `dynamic_XOR_decrypt` which reverses XOR encryption by XORing each character with the `text_key`, then reverses the string. Since XOR is self-inverse, applying it again with the same key decrypts the text

<img width="1280" height="832" alt="Screenshot 2025-10-29 at 1 59 15 PM" src="https://github.com/user-attachments/assets/b81fa778-586d-40ba-a9e7-fe170d71c405" />

- Called the function to recieve the output, this the final script
```
from random import randint
import sys    

def generator(g,x,p):
    return pow(g,x) % p

a = 94
b = 21
cipher = [131553, 993956, 964722, 1359381, 43851, 1169360, 950105, 321574, 1081658, 613914, 0, 1213211, 306957, 73085, 993956, 0, 321574, 1257062, 14617, 906254, 350808, 394659, 87702, 87702, 248489, 87702, 380042, 745467, 467744, 716233, 380042, 102319, 175404, 248489]

def decrypt_multiplication(cipher,key) :
    plaintext=""
    for num in cipher:
        original_ord = num // (311 * key) 
        plaintext+=chr(original_ord)
    return plaintext 

p=97
g=31
u = generator(g, a, p)
v = generator(g, b, p)
key = generator(v, a, p)

print(key)
decipher = decrypt_multiplication(cipher, key)
print(decipher)

def dynamic_xor_decrypt(cipher_text, text_key):
    plaintext = ""
    key_length = len(text_key)
    for i, char in enumerate(cipher_text):
        key_char = text_key[i % key_length]
        decrypted_char = chr(ord(char) ^ ord(key_char))
        plaintext += decrypted_char
    # Reverse the string since encryption reversed it
    return plaintext[::-1]
##text_key= trudeau

print("Flag :",dynamic_xor_decrypt(decipher,"trudeau"))

```
and the output 

<img width="1280" height="832" alt="Screenshot 2025-10-29 at 2 17 49 PM" src="https://github.com/user-attachments/assets/9418ef25-a3e9-4496-93a5-d02ddcff930c" />



## Flag:

```
picoCTF{custom_d2cr0pt6d_8b41f976}
```

## Concepts learnt:

- Lots of debugging practice
- XOR encryption takes characters converts them to ASCII and works at a bit level simmilar to XOR gate. Has a self-reversible nature.
- Understanding multi-layered encryption
- Reverse engineering the code to understand the encruyption
- A better undestanding of Python 
## Notes:

- append only works on lists not strings i tried adding value to `decipher` using append by that didn't work


## Resources:

- Claude AI 


***




