# Summary :
RSA is an [^1]asymmetric cryptographic system where:
Public key verifies signatures / encrypts data
Private key creates signatures / decrypts data
Security comes from difficulty of factoring a large number.

This note is my attempt to make that mental model of my understanding of RSA explicit.

# Components :
## Values

- **p, q** → large secret prime numbers
- **n = p × q** → called modulus (this can be public, but guessing p and q from just n would be hard for large prime numbers)
- **φ(n) = (p−1)(q−1)** → Euler’s totient
- **e** → public exponent (pick such a number it must be coprime with Euler’s totient i.e no common divisor/factor other than 1)
- **d** → private exponent (chosen such that e×d mod φ(n) = 1 )

## Private Key
- **Private key = (n, d)**
- d is the secret part which can not be shared.
- p,q,n,e are also part of private key in typical RSA implementation to allow construction of public key.

## Public Key
- **Public key = (n, e)**
- n, e can be shared publicly. With just n and e, it's practically impossible to "guess" d hence the private key is safe.

# Core Idea
## Public → Private
From public key we know the:
- n
- e
We don't know the:
- **p, q** (the primes)
- φ(n) = (p–1)(q–1)
- d

To get the private key, we need 'd'. d can only be calculated if we have p and q but we only have n and it's not practically possible to guess the p and q from n.

## Signing/Verification:
- **Signing with Private Key:** *(Message_hash)^d mod n* → This is why we needed n and d in private key. The result is signature
- **Verification with Public Key:** *(Signature)^e mod n* → The result of this would be the message hash
- Only someone with the private key could have produced a signature that verifies back to message hash. The public key lets anyone check, but not generate valid signatures.
- **JWT Context:**
	- A JWT has: Header, Payload, Signature
	- The issuer signs it using their **private key** on the payload to generate the signature
	- You can look up the issuer’s public key(n + e) in the [^2]JWKS, rebuild the RSA public key, and check whether *(Signature)^e mod n* is equal to the hash of payload of not

## Encryption/Decryption:
- **Encrypt with Public Key:** *(Data)^e mod n* → This gives us the cipher that can be shared, this will only be decrypted with the private key.
- **Decrypt with Private Key:** *(Cipher)^d mod n* → This would give us back our message

# In Practice
## Public Key (PEM format)
A public key PEM is short and simple. Example:
```
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAw...
...snip...
-----END PUBLIC KEY-----
```
If you decode the Base64, it contains only:
- n (modulus)
- e (public exponent)

That’s it. Enough to verify signatures of messages to verify **authenticity**, or to encrypt messages that can only be decrypted by the key owner to ensure **confidentiality**. This can be checked locally on any public key
```
openssl rsa -in public.pem -pubin -text -noout

Public-Key: (2048 bit)
Modulus: n = ...
Exponent: ...
```


## Private Key (PEM format, PKCS#1)
A private key PEM is much longer. Example:
```
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAw...
...snip...
-----END RSA PRIVATE KEY-----
```
Inside, it stores:
- n (modulus, same as public key)
- e (public exponent, same as public key)
- d (private exponent)
- p (first prime)
- q (second prime)
- d mod (p–1) (used to speed up decryption/signing)
- d mod (q–1)
- q⁻¹ mod p (Chinese Remainder Theorem optimization)

So the private key has all the ingredients to easily regenerate the public key. You can check this as well with openssl in your machine if you have a private key
```
openssl rsa -in private.pem -text -noout

RSA Private-Key: (2048 bit, 2 primes)
modulus: n = ...
publicExponent: e = 65537
privateExponent: d = ...
prime1: p = ...
prime2: q = ...
exponent1: d mod (p-1)
exponent2: d mod (q-1)
coefficient: q^-1 mod p
```


# Example

To understand better, lets have trivial examples
- `p = 3`, `q = 11` (trivially small primes chosen freely)
- `n = p * q = 33`
- φ(n) = (p–1)(q–1) = 2 * 10 = 20
- `e = 3` (chosen freely but coprime with 20)
- `d = 7` (since 3 * 7 mod 20 = 1)

- Public key = (n=33, e=3)
- Private key = (n=33, d=7)

## Signing/Verification:
Say our message hash = `m = 4`
Signature is computed as:
```
signature = m^d mod n
          = 4^7 mod 33
signature = 16
```
We verify by computing:
```
check = s^e mod n
      = 16^3 mod 33
check = 4
```
Since v is same as message hash, we can consider this message verified

## Encryption/Decryption:
Let’s say the message = `m = 4`.  
We encrypt with `e`:
```
cipher = m^e mod n
       = 4^3 mod 33
cipher = 31
```
Now we decrypt with `d`:
```
message = cipher^d mod n
        = 31^7 mod 33
message = 4
```
We get our original message back

# Padding:
There are two completely different paddings used in the context of RSA
1. **Base64 padding (=):** This is for the purpose of encoding correctness. This padding has no security implications. It is purely an encoding concern. Base64 encodes binary data using 4-character blocks.
	- 3 bytes (24 bits) → 4 Base64 characters
	- Each Base64 character represents 6 bits
	- Base64 strings must be length % 4 == 0.
	- If remainder is `0`, 4 or 0 base64 characters → No padding needed (input was 0 or 24 bits or multiple).
	- If remainder is `2`, 2 base64 characters → add `==` in padding (input was 12 bits or multiple).
	- If remainder is `3`, 3 base64 characters → add `=` in padding (input was 18 bits or multiple).
	- If remainder is `1`, 1 base64 characters → Input was 6 bits? Impossible, minimum input is 1 byte or 8 bits.

2. **Cryptographic Padding:** If we sign directly with signature = hash^d mod n, then same message → same signature. This makes it vulnerable to replay and forgery attacks so we never sign raw hashes. Instead, standardized padding schemes are used:
	- **RSA-PSS** for signatures
	- **RSA-OAEP** for encryption
These schemes introduce randomness and structure.


# Footnotes
[^1]: Symmetric cryptography uses **single shared key** for encryption and decryption, it is **fast** for large data but need secure key exchange since single key is used; asymmetric cryptography uses a **public key (shareable)** for encryption and a **private key (secret) for decryption**, slow for large data but more secure

[^2]: JWKS (JSON Web Key Set) exists because services like Google, AWS etc. rotate keys regularly. Instead of hardcoding keys, they expose a JWKS endpoint.
