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

