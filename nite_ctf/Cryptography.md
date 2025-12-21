# 1. Hash Vegas 

> I think the casino is broken. I bought a ticket, the system said I won, and then my bank account instantly hit zero.
Connection: ncat --ssl vegas.chals.nitectf25.live 1337

## Solution:

- The casino gives us a voucher for winnign our goal is to redeem the voucher and forge a voucher that would give us 1 billion dollars which gives us the flag.
- After taking a look at `lottery.py` the server generates a secret key
```
self.secret = os.urandom(16).hex()
```
`os.urandom(16)` creates 16 raw bytes, but `.hex()` converts them into a hexadecimal string

- When a voucher is created the server signs it using this format:
```
ticket_hash = hash_func((self.secret + ticket_data).encode()).digest()[:20]
```
The code .digest()[:20] truncates the hash to only 20 bytes for SHA256 that means that 12 bytes are deleted whereas for SHA1 nothing changes which is the vulnerability we are looking for.

- we create a brute force script that would connect to the server epeatedly until we randoly hit the desired SHA1 encryption i.e. 1 in 2048 chances.
- The attack first used python's `Socket` library to connect to the server because the server uses SSL we had to wrap our socket.
- The second step of the attack is to repeatedly closing and opening the server till we win a voucher.
- We used the hashpumpy tool to forge a new signature. We told it to take the original signature and append |1000000000 to the data.
```
new_sig, new_data = hashpumpy.hashpump(
    original_sig,       # The valid signature we got
    original_data,      # The original text
    b"|1000000000",     # The malicious payload
    32                  # The key length (Crucial: 32 bytes!)
)
```
- After creating the forged voucher we send it back to the server if the server is using SHA256 or SHA3 the attack fails but if the server is using SHA1 server accepts the voucher, credited us $1,000,000,000, and we bought the flag.
- 

## Flag:

```
nite{9ty%_0f_g4mbler5_qu17_b3f0re_th3y_mak3_1t_big}
```

## Concepts learnt:

- Hash length extension attack: hash functions (like MD5, SHA-1, SHA-256) operate in blocks. If an application calculates Hash(Secret + Data), an attacker who knows the Hash and the length of the Secret can mathematically compute Hash(Secret + Data + Padding + MaliciousData) without ever knowing the Secret.
- Using `ThreadPoolExecutor` to launch concurrent network connections. This turned a sequential process (waiting for one attempt to finish before starting the next) into a parallel process, increasing speed by ~50-100x.
- Using python's `Socket`, `hashpumpy` libraries.
## Notes:

- Found out that the server creates new tokens everytime one connects to a server and according to my original assumption the token was unique so changed that.
- The probability of getting the desired SHA-1 encryption were so low (0.05%) the first script took 1 hour to check for 2000 iterations so incorporated multithreading.

## Resources:

- ClaudeAI
- GeminiAI


***

# 2. smol fan

> I was just doing my homework in the library, but there was this huge server with noisy FANs distracting me. To show them who's boss I recorded the FAN noises and I'm almost there, just need to get the signing key.


## Solution:

- This specific challenge had an ECDSA encyption and due to the constraints of the challenge the value of `k` is very small which makes the challenge solvable.
- If the value of `k` is biased we can find the value of the private key `d`.
- `k≡s ^−1z+s ^−1r⋅d(modn)` this looks like a simple algebric expression with some modular wrapping where
   We know u and t (from the signature).
- Since we have one equation with two unknowns we cant solve it directly, so we create many signatures making a system of equations.
```
samples = []
for i in range(12):
    print(f"[*] Collecting signature {i+1}...")
    r, s, z = recover_r_s_z(io, i)
    samples.append((r, s, z))
```
- We want to find a single value of `d` that makes all the value of ki 'small' simultaneously.
- Crucially, we recenter u. Instead of looking for a nonce between 0 and 2^200, we shift the values so the target is centered around 0.
```
CENTER = 2**(BITS-1)

# ... inside the loop ...
s_inv = inverse_mod(s, n)
t = (s_inv * r) % n
u = (s_inv * z) % n

# KEY FIX: Recenter the 'u' value
u_centered = (u - CENTER) % n

ts.append(t)
us.append(u_centered)
```
- We construct a matrix M that represents our system of linear equations.
```
M = Matrix(ZZ, num_samples + 2, num_samples + 1)

# Rows 0..m-1: Modulus n
for i in range(num_samples):
    M[i, i] = n

# Row m: The 't' coefficients
for i in range(num_samples):
    M[num_samples, i] = ts[i]

# Row m+1: The 'u' constants (offsets)
for i in range(num_samples):
    M[num_samples + 1, i] = us[i]
M[num_samples + 1, num_samples] = B
```
- We run the LLL algorithm on our matrix to find th shortest vector because we recentered our data, the vector containing our biased nonces is mathematically guaranteed to be one of the shortest in the lattice.
```
# --- LLL Reduction ---
print("[*] Running LLL reduction (this may take a moment)...")
L = M.LLL()
```
- After finding the shortest vector using `LLL` we then turn it into a secret key.
```
# --- Solution Extraction ---
    for row in L:
        # 1. FIND THE ROW: Look for the anchor B
        if abs(row[num_samples]) == B:
            
            # 2. EXTRACT NONCE: Get the first centered nonce from the row
            potential_k_centered = row[0]
            
            # 3. FIX SIGN: Flip the sign if the anchor is negative
            if row[num_samples] == -B:
                potential_k_centered = -potential_k_centered
                
            # 4. SOLVE FOR d: Use the algebra equation
            # k = t*d + u  =>  d = (k - u) * t^-1
            
            t0 = ts[0]
            u0 = us[0] # Note: us[0] is the centered u
            
            d_cand = ((potential_k_centered - u0) * inverse_mod(t0, n)) % n
            
            # 5. VERIFY: Check if d generates the correct Public Key
            try:
                Q_cand = d_cand * G
                if Q_cand.xy()[0] == Qx:
                    return d_cand
            except:
                continue
```
- We do this by finding the correct anchor value by design we have set up an anchor value at the end of every row.
- We have the value of `k` and we can derive the value of `d` by simple algebric.
- Now that we have the correct private key `d` we can forge the signature to send to the server that would give us the flag.

## Flag:

```
nite{1'm_a_smol_f4n_of_LLL!}
```



# Script

```
#!/usr/bin/env sage
from pwn import *
from hashlib import sha256
import math

# --- CONFIGURATION ---
HOST = 'smol.chalz.nitectf25.live'
PORT = 1337
# SECP256k1 Constants
n = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141
p = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F
a = 0
b = 7
E = EllipticCurve(GF(p), [a, b])
G = E.point((0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798, 
             0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8))

def recover_r_s_z(io, msg_idx):
    msg_str = f"msg{msg_idx}"
    io.sendlineafter(b"> ", b"2")
    io.sendlineafter(b"hex: ", msg_str.encode().hex().encode())
    
    io.recvuntil(b"m = ")
    m = int(io.recvline().decode().strip())
    io.recvuntil(b"a = ")
    a = int(io.recvline().decode().strip())
    io.recvuntil(b"b = ")
    _ = io.recvline()

    # Recovery logic
    val = pow(10, 11)
    r = math.gcd(a - val, m)
    s = m // r
    z = int(sha256(msg_str.encode()).hexdigest(), 16)
    return r, s, z

def solve_hnp(samples, Qx):
    # HNP Parameters
    # We are looking for k < 2^200
    # To optimize LLL, we recenter the interval to [-2^199, 2^199]
    BITS = 200
    B = 2**BITS
    CENTER = 2**(BITS-1)
    
    num_samples = len(samples)
    print(f"[*] Building lattice for {num_samples} samples (Recentered)...")

    # Matrices for HNP usually follow the CVP->SVP embedding technique
    # We want to find vector v = (k1, k2, ... km, C)
    # k_i = t_i * d + u_i  (mod n)
    
    M = Matrix(ZZ, num_samples + 2, num_samples + 1)
    
    ts = []
    us = []
    
    for r, s, z in samples:
        s_inv = inverse_mod(s, n)
        t = (s_inv * r) % n
        u = (s_inv * z) % n
        u_centered = (u - CENTER) % n
        
        ts.append(t)
        us.append(u_centered)

    # --- Matrix Construction ---
    # Rows 0..m-1: Modulus n
    for i in range(num_samples):
        M[i, i] = n

    for i in range(num_samples):
        M[num_samples, i] = ts[i]
    M[num_samples, num_samples] = 0

    for i in range(num_samples):
        M[num_samples + 1, i] = us[i]
    M[num_samples + 1, num_samples] = B

    # --- LLL Reduction ---
    print("[*] Running LLL reduction (this may take a moment)...")
    L = M.LLL()

    # --- Solution Extraction ---
    for row in L:
        # We look for a row where the last element is ±B (indicating we used the last row once)
        if abs(row[num_samples]) == B:
            print("[*] Found potential vector!")
            
            # The row contains (k1', k2', ... km', ±B)
            # where ki' is the centered nonce (ki - CENTER)
            # We just need one to solve for d.
            
            potential_k_centered = row[0]
            
            # Determine sign based on the last element
            # If last element is -B, we need to flip the whole row to get +B (our anchor)
            if row[num_samples] == -B:
                potential_k_centered = -potential_k_centered
                
            # Un-center the k
            k_recovered = potential_k_centered + CENTER
            
            # Solve for d
            # k = t*d + u => d = (k - u) * t^-1
            t0 = ts[0]
            u0 = us[0] # Note: us[] list contains RECENTERED u
            # Using the original equation: k_real = t*d + u_original
            # Or using centered: k_centered = t*d + u_centered
            
            d_cand = ((potential_k_centered - u0) * inverse_mod(t0, n)) % n
            
            # Verify
            try:
                Q_cand = d_cand * G
                if Q_cand.xy()[0] == Qx:
                    return d_cand
            except:
                continue
    return None

def main():
    io = remote(HOST, PORT, ssl=True)
    io.recvuntil(b"Quit")

    io.sendline(b"1")
    io.recvuntil(b"Qx = ")
    Qx = int(io.recvline().decode().strip())
    print(f"[+] Target Public Key X: {Qx}")

    # Increased samples to 12 for better stability
    samples = []
    for i in range(12):
        print(f"[*] Collecting signature {i+1}...")
        r, s, z = recover_r_s_z(io, i)
        samples.append((r, s, z))
    
    d = solve_hnp(samples, Qx)
    
    if d:
        print(f"\n[+] SUCCESS! Private Key d: {d}")
        
        target_msg = b"gimme_flag"
        z_target = int(sha256(target_msg).hexdigest(), 16)
        
        k_eph = 13371337
        R = k_eph * G
        r_sig = Integer(R.xy()[0]) % n
        s_sig = (inverse_mod(k_eph, n) * (z_target + r_sig * d)) % n
        
        print(f"[*] Submitting signature...")
        io.sendline(b"3")
        io.sendlineafter(b"r: ", str(r_sig).encode())
        io.sendlineafter(b"s: ", str(s_sig).encode())
        
        print(io.recvall().decode())
    else:
        print("[-] Lattice attack failed. The bias might be tighter than expected.")

if __name__ == "__main__":
    main()

```

