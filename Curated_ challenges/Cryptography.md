# 1. All Signs Allign


## Solution:

- The challenge used the concept of quadratic residue to make an encryption.
- The core challenge used a function get_x to obtain the value of x as a quadratic residue and a function get_y which gae us quadratic non residue.
```
def get_x():
    while True:
        x = randint(2,p-1)
        if pow(x,(p-1)//2,p) == 1:
            break
    return x
```
-Each flag bit is encoded as:
0 → a · x (mod p)
1 → a · y (mod p)
- To solve we use the properties of the Legendre Symbol to check each bit of the `out.txt` for quadratic residue
  ```
  for val in out:
    check = pow(val,(p-1)//2,p)
    if check == 1:
        bits.append('1')
    else :
        bits.append('0')

  ```
- this give an output in bytes format we convert it to flag using 
```
bit_string=''.join(bits)
flag_int = int(bit_string,2)
flag= long_to_bytes(flag_int) 
```
python script is as follows:
```
def bytes_to_long(s):
    return int.from_bytes(s, 'big')

def long_to_bytes(n):
    return n.to_bytes((n.bit_length() + 7)//8, 'big')

p = 9129026491768303016811207218323770273047638648509577266210613478726929333106121387323539916009107476349319902011390210650434835260358014251332047605739279

with open('out-2.txt', 'r') as f:
     out = eval(f.read())

bits = []
for val in out:
    check = pow(val,(p-1)//2,p)
    if check == 1:
        bits.append('1')
    else :
        bits.append('0')

bit_string=''.join(bits)
flag_int = int(bit_string,2)
flag= long_to_bytes(flag_int)       

print(flag)
```

## Flag:

```
nite{r3s1du35_f4ll1ng_1nt0_pl4c3}
```

## Concepts learnt:

- Quadratic residue : A number x is a quadratic residue mod p if it's a perfect square mod p.
- legendre symbol is a method to check if a number is quadratic residue or not `x^((p-1)/2) mod p`.
- The property of legendre symbol that helped in solving this question was the symbol treats each factor independently.

## Notes:

- In my first attempt i had wrong values of 0,1 which gave me garbage value.
```
legendre = pow(val, (p-1)//2, p)
if legendre == 1:  # It's a QR
    bit = '0'
else:  # It's a NQR
    bit = '1'
```

## Resources:

- Chat GPT
- Cluade
- https://www.youtube.com/watch?v=aBn7BaRxu2g

# 2. Residue Refinery 


## Solution:

- The encryption is done in arithmetic modulo `x^2 - 3= 0` and all coefficients were module `257` this is a vulnerability coz the calculations are finite.
- The main encryption takes two bits from the flag and multiplies it with random integer keys.
- We are given the cyphertext and the first two bits of the flag, with this information we can find the random key and then reverse the whole encryption.
- we wound need to the find the modular inverse of the flag bits as `ct=ks*flag` so we define this operation 
```
def inverse(self):
        a, b = self.n
        det = (a*a - 3*b*b) % self.p
        
        if det == 0:
            raise ValueError("No inverse exists")
        
        # Find modular inverse of det
        det_inv = pow(det, -1, self.p)  
        
        c = (a * det_inv) % self.p
        d = (-b * det_inv) % self.p
        
        return Num([c, d])
```
- The logic behind it being For (a + bx)^(-1) where x^2 = 3 We need (a + bx)(c + dx) = 1, This means: ac + 3bd = 1 and ad + bc = 0.
- We first find the keys using 
```
ks = ct_num * flag_inv
```
- Simmilarly the decryption would follow 
```
for i in range(0, len(ct_bytes), 2):
    
    ct_block = Num([ct_bytes[i+1], ct_bytes[i]])
    
    flag_block = ks_inv * ct_block
    
    flag.extend(flag_block.to_bytes())
```
The python script is as follows:
```

class Num:
    def __init__(self, n):
        self.n = list(n)
        self.p = 257
        self.f = [1, 0, -3]  # reduction polynomial
    
    def __add__(self, other):
        return (self.n + other.n) % self.p

    def __mul__(self, other):
        prod = [0] * 5
        for i in range(2):
            for j in range(2):
                prod[i + j] += self.n[i] * other.n[j]
        return Num([(prod[0] + 3*prod[2]) % self.p, prod[1] % self.p])

    def to_bytes(self):
        return bytes(self.n)[::-1]
    
    def inverse(self):
        a, b = self.n
        det = (a*a - 3*b*b) % self.p
        
        if det == 0:
            raise ValueError("No inverse exists")
        
        # Find modular inverse of det
        det_inv = pow(det, -1, self.p)  
        
        c = (a * det_inv) % self.p
        d = (-b * det_inv) % self.p
        
        return Num([c, d])
    
    def __repr__(self):
        return f"Num({self.n})"


flag_known = [49, 109]  # First 2 bytes: '316d'
ct_hex = '9813d3838178abd17836f3e2e752a99d5cd3fba291205f90c1d0a78b6eca'
ct_bytes = bytes.fromhex(ct_hex)


ct_first = [ct_bytes[1], ct_bytes[0]] 
print(f"First CT block: {ct_first}")



flag_num = Num(flag_known)
ct_num = Num(ct_first)

flag_inv = flag_num.inverse()
print(f"Flag inverse: {flag_inv}")

ks = ct_num * flag_inv
print(f"Recovered key: {ks}")

ks_inv = ks.inverse()
print(f"Key inverse: {ks_inv}")

flag = bytearray()

for i in range(0, len(ct_bytes), 2):
    
    ct_block = Num([ct_bytes[i+1], ct_bytes[i]])
    
    flag_block = ks_inv * ct_block
    
    flag.extend(flag_block.to_bytes())

print(f"\nDecrypted flag: nite{{{flag.decode()}}}")

```

## Flag:

```
nite{m10p7rm_d0lu?31_4__Mh7_30mud3l}
```

## Concepts learnt:

- Polynomial Rings : representing the numbers as polynomials a + bx
- Finding Multiplicative Inverse

## Notes:


## Resources:

- Claude.ai


***
# 3. 3. Quixorte

> Put in the challenge's description here

## Solution:

 - The main encryption rotates the bits of an image and takes a randomly generated key and uses an XOR encryption on it and then the bits are multiplied.
 - To reverse the encryption we first need to retrieve the key, XOR is self reversing but th XOR is sliding so we need to solve it cummulatively 
 ```
key = [0]*8
key[0] = v[0]
xor_sum = key[0]

for i in range(1, 8):
    key[i] = v[i] ^ xor_sum
    xor_sum ^= key[i]
 ```
- Since we know that this is a PNG file we already now the first 8 bytes of the file and can use that to compare.
- Once we have obtained the key we can just use the derypt function which unrotates by left rotation and uses XOR which is self reversing to find the original file.
```
def rotate(b, i):
    r = i % 8
    return ((b >> r) | (b << (8 - r))) & 0xFF

def unrotate(b, i):
    r = i % 8
    return ((b << r) | (b >> (8 - r))) & 0xFF

def decrypt(enc, key):
    dec = bytearray(enc)

    # Undo sliding XOR in REVERSE
    for i in range(len(dec) - len(key), -1, -1):
        for j in range(len(key)):
            dec[i + j] ^= key[j]

    # Undo rotation
    return bytearray(unrotate(dec[i], i) for i in range(len(dec)))

# PNG header
png_header = b"\x89PNG\r\n\x1a\n"

enc = bytearray(open("quote.png.enc","rb").read())

rot = [rotate(png_header[i], i) for i in range(8)]
v = [enc[i] ^ rot[i] for i in range(8)]

key = [0]*8
key[0] = v[0]
xor_sum = key[0]

for i in range(1, 8):
    key[i] = v[i] ^ xor_sum
    xor_sum ^= key[i]

key = bytearray(key)
print("Recovered key:", key.hex())


dec = decrypt(enc, key)

with open("decrypted.png","wb") as f:
    f.write(dec)

print("Done. Check decrypted.png")

```
- Used an online tool hexedit to check for mistakes
<img width="1280" height="832" alt="Screenshot 2025-12-07 at 12 41 33 AM" src="https://github.com/user-attachments/assets/ec761012-ffad-42cc-b905-3a0a270fa0a4" />


## Flag:

<img width="250" height="250" alt="decrypted" src="https://github.com/user-attachments/assets/ee6bd862-49ed-4f3b-8e67-6667092f641d" />


## Concepts learnt:

- All PNG file have the same headers.
- XOR is self reversing.
- Making use of a triangle XOR system.
- Since we already know the header of the PNG file i had to compare encrypted bytes with known plaintext.

## Notes:

- The first time i solved this i did not applied a cummulative way of reversing the XOR, which led to having a damaged PNG.

## Resources:

- CHAT GPT


***



