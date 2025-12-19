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

# 2. Challenge name

> Description

.
.
.
