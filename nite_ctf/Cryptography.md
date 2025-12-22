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
# 3. Stronk Rabin

> rabin weak. I make it stronk. oh no. it broke.

## Solution:

- The vulnerability lies within the server's `DEC` function, the function is returning mutiples of the secret primes.
```
    test_values = list(range(2, 60)) 
    dec_results = {}
    
    for val in test_values:
        enc_val = query(io, "ENC", [val])
        
        # THIS IS THE TRIGGER. The server returns a "broken" result here.
        dec_val = query(io, "DEC", [enc_val])
        
        dec_results[val] = dec_val
```
- For the exploit we use the concept of GCD the idea is if we find the GCD of two multiples of the same prime it would give us the value of the prime
```
factor_set = set()
    dec_list = list(dec_results.items())
    
    # We compare every result against every other result
    for i in range(len(dec_list)):
        for j in range(i+1, len(dec_list)):
            
            # THE ATTACK: Calculate GCD of two different decrypted values
            g = gmpy2.gcd(dec_list[i][1], dec_list[j][1])
            
            # Filter: If the result is a 200-300 bit number, it's likely one of the primes
            if 200 < g.bit_length() < 300:
                factor_set.add(g)
```
- `gmpy2.gcd` gives us the raw primes and the `if` statement ensures that we dont include `1`
- How this works is the script sends simple numbers to the sever and asks it to `ENC` them then `DEC` them retrieve the output and `GCD` it against all the other outputs this will eventually lead us to four prime numbers.
- Once we have the primes we have to recover the private key, to do that we would have to find the squareroots modulo of each prime number individually
```
moduli = [p, q, r, s]
    roots_per_prime = []
    
    for m in moduli:
        # THE FORMULA: Calculate square root for this specific prime
        # pow(base, exponent, modulus)
        root = pow(C, (m + 1) // 4, m)
        
        # Save both the positive (+root) and negative (-root) versions
        roots_per_prime.append((root, (-root) % m))
```
- Once we have the four square modulo we have to reassemble them back together into one big number that satisifies neccesary conditions for that we use Chinese remainder Theorem.
```
plaintexts = []
    # Loop 0 to 15 (16 combinations)
    for i in range(16):
        selected_roots = []
        
        # Pick either the positive or negative root for each prime based on bits
        for j, (pos_root, neg_root) in enumerate(roots_per_prime):
            bit = (i >> j) & 1
            selected_roots.append(neg_root if bit else pos_root)
        
        # THE ASSEMBLY: Use CRT to combine the 4 small roots into one big plaintext
        plaintext = crt(moduli, selected_roots)[0]
        plaintexts.append(plaintext)
```
- We would have 16 candidate messages we convert them to text and check which has the prefix `nite{`
```
for i, candidate in enumerate(candidates):
        try:
            # Convert the big number to bytes (text)
            flag_bytes = long_to_bytes(candidate)
            
            # Check if the known flag format is present
            if b'nite{' in flag_bytes:
                print(f"[+] Flag: {actual_flag}")
                return actual_flag
```

## Flag:

```
nite{rabin_stronk?_no_r4bin_brok3n}
```

# Script:

```
#!/usr/bin/env python3
from pwn import *
import json
from Crypto.Util.number import long_to_bytes
import gmpy2
from sympy.ntheory.modular import crt

def query(io, func, args):
    """Send a query to the server"""
    request = json.dumps({"func": func, "args": args})
    io.sendline(request.encode())
    response = json.loads(io.recvline().decode())
    return response.get("retn")

def is_prime(n):
    return gmpy2.is_prime(n)

def solve():
    print("[*] Connecting to server with SSL...")
    io = remote('stronk.chals.nitectf25.live', 1337, ssl=True)
    
    # Receive C
    msg = io.recvline().decode().strip()
    c_line = io.recvline().decode().strip()
    data = json.loads(c_line)
    C = data['C']
    
    print(f"[+] Received C: {C}")
    
    # Extract primes via GCD
    print("\n[*] Extracting prime factors...")
    
    test_values = list(range(2, 60))
    dec_results = {}
    
    for val in test_values:
        enc_val = query(io, "ENC", [val])
        dec_val = query(io, "DEC", [enc_val])
        dec_results[val] = dec_val
    
    factor_set = set()
    dec_list = list(dec_results.items())
    
    for i in range(len(dec_list)):
        for j in range(i+1, len(dec_list)):
            g = gmpy2.gcd(dec_list[i][1], dec_list[j][1])
            if 200 < g.bit_length() < 300:
                factor_set.add(g)
    
    primes = [f for f in factor_set if is_prime(f)]
    
    if len(primes) != 4:
        print(f"[-] Found {len(primes)} primes, expected 4")
        io.close()
        return
    
    p, q, r, s = sorted(primes)
    n = p * q * r * s
    
    print(f"[+] Recovered primes (all ≡ 3 mod 4):")
    print(f"    p = {p.bit_length()} bits")
    print(f"    q = {q.bit_length()} bits")
    print(f"    r = {r.bit_length()} bits")
    print(f"    s = {s.bit_length()} bits")
    print(f"[+] n = {n.bit_length()} bits")

    moduli = [p, q, r, s]
    
    roots_per_prime = []
    for m in moduli:
        root = pow(C, (m + 1) // 4, m)
        roots_per_prime.append((root, (-root) % m))
    
    print(f"[+] Computed square roots for each prime")
    
    # Generate all 16 possible combinations using CRT
    plaintexts = []
    for i in range(16):
        # Each bit of i determines which root to use
        selected_roots = []
        for j, (pos_root, neg_root) in enumerate(roots_per_prime):
            bit = (i >> j) & 1
            selected_roots.append(neg_root if bit else pos_root)
        
        # Use CRT to combine
        plaintext = crt(moduli, selected_roots)[0]
        plaintexts.append(plaintext)
    
    print(f"[+] Generated {len(plaintexts)} possible plaintexts via CRT")
    
    # Now we need to find which one is the flag
    # The constraints are: flag < n, 2*flag > n, flag^2 > n
    
    candidates = []
    for pt in plaintexts:
        # Check constraints
        if pt < n and 2*pt > n and pt*pt > n:
            candidates.append(pt)
            print(f"[*] Candidate satisfies constraints: {pt.bit_length()} bits")
    
    print(f"\n[+] Found {len(candidates)} candidates matching constraints")

    for i, candidate in enumerate(candidates):
        try:
            flag_bytes = long_to_bytes(candidate)
            if b'nite{' in flag_bytes:
                # Extract just the flag part (before any padding)
                flag_str = flag_bytes.decode('latin-1')
                # Find the flag
                start = flag_str.index('nite{')
                end = flag_str.index('}', start) + 1
                actual_flag = flag_str[start:end]
                
                print(f"\n[!] ========== FOUND FLAG! ==========")
                print(f"[+] Flag: {actual_flag}")
                print(f"[!] ===================================")
                io.close()
                return actual_flag
        except:
            pass

    print("\n[*] Direct candidates didn't work, using server's DEC oracle...")
    dec_c = query(io, "DEC", [C])
    print(f"[+] DEC(C) = {dec_c}")
    
    
    print("\n[*] Analyzing relationship between roots and DEC output...")
    
    print("\n[*] Testing all 16 CRT plaintexts...")
    for i, pt in enumerate(plaintexts):
        try:
            flag_bytes = long_to_bytes(pt)
            # Check if it looks like a valid flag (printable ASCII)
            if all(32 <= b < 127 or b in [10, 13] for b in flag_bytes[:20]):
                print(f"[*] Plaintext {i}: starts with {flag_bytes[:50]}")
                if b'nite{' in flag_bytes:
                    print(f"\n[!] FOUND FLAG!")
                    print(f"[+] Flag: {flag_bytes.decode()}")
                    io.close()
                    return flag_bytes.decode()
        except:
            pass
    
    print("\n[-] Could not find flag among CRT plaintexts")
    io.close()

if __name__ == "__main__":
    solve()
```

