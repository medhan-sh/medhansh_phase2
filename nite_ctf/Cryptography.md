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


# 4. Symetric starter 

# Solution 

- The critical vulnerability is that `shifts` are leaked in the output file.
- The second line of `out.txt` contains all the MSB values extracted during the encryption this creates a system of constraints.
- For each block `i` `shifts[i]` = MSB of `nonce[i]`
- We make of `Z3` to find a key that satisfies all the constraints
```
key_bv = BitVec('key', 128)  # Unknown 128-bit key
solver = Solver()

nonce = key_bv
for i in range(constraint_blocks):
    expected_msb = int(shifts_bin[i])  # Known from leak
    actual_msb = Extract(127, 127, nonce)  
    solver.add(actual_msb == expected_msb)  
    
    shifts_value = int(shifts_bin[:i+1], 2)
    nonce = (nonce + shifts_value) % (2**128)
    nonce = RotateLeft(nonce, N)
```
- Once `Z3` finds a key tisfying all the constraints reconstruct the keystream form the reconvered KEY
- We run the AES-ECB on the nonce to recover the first 16 bytes of the keystream .
- The cipher decryption is just XOR which is self reversing
- we then run a search for the flag looking for the prefix `nite{` in the plaintext

# Script

```
from Crypto.Cipher import AES
from Crypto.Util.number import long_to_bytes, bytes_to_long
from pwn import xor

print("="*70)
print("CTF Challenge: Symmetric Starter - Solution")
print("="*70)

# Hardcoded data from out.txt (from the document provided)
encrypted_hex = "97dd3c17644f43689603fd3ae505743ea5d8a116337267cda591c51fb89aecb9aecab0715a8f763de910c7230af1a7570f935d0f52e4e7506613624321ecd66f1eebd7f7d481ab1cfc85d5bcc42246b09f502c062e501824df89bd0aef5080108fd8370775a88c004c134ca151dcf64ab9691ffdf4bfd5e06b1f6e167620add68435041a0700b859a460d1e138d66c1a8c4789df20bced9985c5bec9078b4c2185b15d2a4c9fd7e26ede487fd4bba4fa4ff8709b8e8d9a00fcd088edc6187ded39862b8af0c55a66d0bd4553f7a75f495bee5f70b0a8dbdbdcd7966aaa3e5390c9de9575576eb9571bab74790039983c90a64afc2dc31e16b8b1a7a41fa734a8fa5bd625c86d206eb7562dae01a5d2be9421eefdde8f3b7c33f31ae09455e656e6ae0090b060d82868a2f72e2a8b53814435b7ffbafeb2182d213a677b0918f043b81190ec5572ee3903720574bb227613c60813dc9a9391e22e7cae1cc41322e8be00a1ffaec43861264d81e1aabec9b9ad92a06dacfd6ffdba2710d1439fb0945f1ab6b1d599a654bba3f7e8a0b0cf671160c26629d8dcd629ebfe5117214f29224ae5ee119ff50157d97d84f5ddd44b5c3ac6edfb726bfc721dd407f37314c6567ec5aa0486203dba822d98380cc2cd9ed833b3fc26148831010e3b1f75a3a2d4215440f462b591d26e466c4dc40b840d568f2bd8a38f05bc0663e788c60ea5af71b15a55edfa008a8bd1829131afa1294e51217f35ab60a09e2e311e02d704c6d1491ba3343b22ac3ab9d4eeff7f07229e27ab170c223aa406222990e694a40bb61993dd0d7af59463463cecc15bec8175f9b9b5829004cadee88580c4e201fb0fef9079a3b85034acf9645afdf41a9d389eeb2458ecc563c08f2ab97a4d1cf9e4b29046c937dff5321e0e4ba914f635e8cb0bb93a58fbad6eec4dff029c614655488914b7086c79834c14c894d129df24d6323f5f6307cbc59efe11553b98182396e1b965107ed632eb44350c0b4eeec69e184a9ae5160c94928793b59f31f98c224890acf60833ffa0efe4f17f3fb6a479ff038689bd8beaa0cf93c2eef7a2a25723b46119d342b6158287cf9a748bd06c24d59e24156e2fc5405d2c7dcea0dfbaf1d6ba90e7c3cf1e60e9a2f8ea01c2c4d19193a025f2d51c2b533a9423a3e7b7a047562a1cf944815811f200ab5bc4c19cabb52e34a94e09459e591f27dec7ed1885ecf626d0db22a2717279cc7a6fd87e0cbc72134aa07750fc5c884952327dc1d6b9067a119d037693fbd0fa31013b38755d87ed711af9b3da5a246fe5873a41bb6dfab309c5938905b2d13bd299da2474ccaddb310b490970bbab447623e1bb5784940f29b1d3bf489507257e2e778995c27b998010d24735a537e8f0ac68a9f67cd1a017952c97f2de79a594cd6690257bea84989913ca83d487585335d9e00773753d7bd5573b92857d9cf96a4900b741d1700f475e9ffe236087653243c74a4dddd3490305838c1a85aea2cf32e39acb2441f248c6519c668798e5dba0686ec366801fe3c7dddb5729e021b34e388c1c0e4eace2dd2d77c33e71d1771f5408c319d370aa1aaaa515de479b138f5f6df6c33e22c376c7d277caab46b921e0faa7b642f20c5b1b24018b759f1f123b8753b7f57fda014ce0cd67d6cb7b419e8656e581808e02ea6511e8cb52e72810d106f21ef133fafcf193357a90be44482acb407147b97c568b43c8a4cd5ce22b0b2bc56ee740c08bdfca6edd947e9632859249ad4522a63e31961d81255bf80ad27a197ddbaed048500e34808abdc6d856a6ece7524df94a8493826acd0c88ab53d89a4e98709775112190f84f8de86eaa9a6643efbe271b2fde4a9fd3a15fe049dbc8b0d899f407a5188c8d66744490505c376c00018020acd6f03a53b160094deb984abbd9b2c7d7d65d5885417b942749fe3d03f5196326f664bdfc3ebeb51f4f6b53157af79b0ecf1ba67e04b6a41a0ca63f453e670bba4b00eb5ffb5e34e58edc45361410ca5a030fadbc339efb47c58b880c026f8d86bd15f745f1ab0d4a173a2f3a9c9d9213d3365640844dd3859da96cdf1dc120add4a6bacc4c9a637fd9370ab8004d79554920255008b263aa6b757812cf2367a6ad57eb69ba7b88c21a29697aabe184cde6ab8581ad0df7424b062ebefa9d9eeb9e543273e96a89747edc384f15fa9e349634c93e9ee5b75dbebfabd83504d95807f7c86bf1e89f1d312493a88a1188c01ef248eadb4744f9a5feab750f1878c461f6e94960b073f03c6bcbaf41ab2bac8e6385cbc54095cd5ac7b17a8b11d515614d28e25ee49ca97ffe40900650eb4a8ec2818154286f0ea9dda54f232e8e5aceec0d63c26663c7df892c3151860ff6c531a4c21f0e0e13349fe7641a59fc96ef62d9699972dcea298873f2f99b4f7ff00b69aa19717d378d6354c33911ae0fb42cc0127cae7fd2346ee6068b6c27e047aca46436e976456c14734490d0b04b1de5697fe6bf115c4c020fc939cb054d1b5c8cccdbea3dfba3e69a141656a98e8a3669ee18a4f889e66d84fbd2c45d859ad6088b7286cd8d815c3307ea1e19a8d47da200dded31f4e41af57f2a174605cb9f9379823105c494823a5a6e5e53edd5ab602f05fdf403dd26325a6acdea223b263e965640fed07fb62f8bda482c7d14797cfabd0c006d0566e10b899f08226e4e2bd1a28622af4f552503f41bc7ddadb3973ddc00969517ffca18d837ae830a3fbbd93a07904be5904db24ccb839bf9b70e629b35c676f6f9108f4d4749344e7ddf85262719caf7381f8069e3bf5af3acb0424323a6cf85c6a9d1b18a313963e29b32d308034507c5eac0ed600f12f74a9c7bfff3de9d547fd396914e0b3e56d0"

shifts_hex = "1b7011ba4c40233fd54815751e3e0c81"

print("Using hardcoded data from out.txt")

# Parse the hex strings
ciphertext = bytes.fromhex(encrypted_hex)
shifts_bytes = bytes.fromhex(shifts_hex)

# Convert shifts to binary  
shifts_int = bytes_to_long(shifts_bytes)
shifts_bin = bin(shifts_int)[2:]  # Don't pad - use actual bits

num_blocks = len(ciphertext) // 16

print(f"Ciphertext: {len(ciphertext)} bytes ({num_blocks} blocks)")
print(f"Shifts: {len(shifts_bytes)} bytes = {len(shifts_bin)} bits")
print(f"Note: We have {len(shifts_bin)} shift bits for {num_blocks} blocks")
print(f"First 80 shift bits: {shifts_bin[:80]}")

# Constants
MOD = 1 << 128
N = 3

def rol(x, n):
    """Rotate left by n bits"""
    return ((x << n) | (x >> (128 - n))) & (MOD-1)

def decrypt_with_key(key_bytes, ciphertext, shifts_str):
    """Decrypt ciphertext using the given key"""
    cipher = AES.new(key=key_bytes, mode=AES.MODE_ECB)
    nonce = bytes_to_long(key_bytes)
    
    keystream_blocks = []
    current_shifts = ""
    num_blocks = len(ciphertext) // 16
    
    # We need to handle the case where shifts might be shorter
    actual_blocks = min(num_blocks, len(shifts_str))
    
    for i in range(actual_blocks):
        # Append current shift bit to accumulator
        current_shifts += shifts_str[i]
        
        # Update nonce before encryption
        nonce = (nonce + int(current_shifts, 2)) & (MOD-1)
        
        # Encrypt nonce to get keystream block
        ks_block = cipher.encrypt(nonce.to_bytes(16, 'big'))
        keystream_blocks.append(ks_block)
        
        # Rotate for next iteration
        nonce = rol(nonce, N)
    
    keystream = b"".join(keystream_blocks)
    plaintext = xor(ciphertext[:len(keystream)], keystream)
    return plaintext

print("\n" + "="*70)
print("SOLVING WITH Z3 CONSTRAINT SOLVER")
print("="*70)

try:
    from z3 import *
    
    # Use first 20 blocks to constrain the key
    constraint_blocks = min(20, num_blocks, len(shifts_bin))
    
    print(f"\nSetting up {constraint_blocks} constraints...")
    
    key_bv = BitVec('key', 128)
    solver = Solver()
    
    # Simulate nonce evolution
    nonce = key_bv
    
    for i in range(constraint_blocks):
        # Extract MSB and add constraint
        expected_msb = int(shifts_bin[i])
        actual_msb = Extract(127, 127, nonce)
        solver.add(actual_msb == expected_msb)
        
        # Update nonce
        shifts_so_far = shifts_bin[:i+1]
        shifts_value = int(shifts_so_far, 2)
        nonce = (nonce + shifts_value) % (2**128)
        nonce = RotateLeft(nonce, N)
    
    print("Solving (this may take 30-60 seconds)...")
    
    result = solver.check()
    
    if result == sat:
        model = solver.model()
        key_val = model[key_bv].as_long()
        key_bytes = key_val.to_bytes(16, 'big')
        
        print(f"\n✅ SUCCESS! Found key:")
        print(f"   {key_bytes.hex()}")
        
        # Decrypt
        print("\nDecrypting...")
        plaintext = decrypt_with_key(key_bytes, ciphertext, shifts_bin)
        
        # Search for flag
        if b"nite{" in plaintext:
            start = plaintext.index(b"nite{")
            end = plaintext.find(b"}", start)
            
            if end != -1:
                flag = plaintext[start:end+1]
                print("\n" + "="*70)
                print("🎉 FLAG FOUND!")
                print("="*70)
                print(f"\n{flag.decode()}\n")
                print("="*70)
            else:
                print(f"\n⚠️  Found 'nite{{' but no '}}'")
        else:
            print("\n⚠️  No flag found")
            print(f"Plaintext sample: {plaintext[:200]}")
    else:
        print(f"\n❌ Result: {result}")

except ImportError:
    print("\n❌ Z3 not installed")
    print("Install with: pip install z3-solver")
except Exception as e:
    print(f"\n❌ Error: {e}")
    import traceback
    traceback.print_exc()


```
