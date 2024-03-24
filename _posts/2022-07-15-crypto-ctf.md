---
title:  "Crypto CTF 2022"
mathjax: true
layout: post
---

[CryptoCTF 2022](https://ctftime.org/event/1573/) writeup some challenges...






# polyRSA

> **Description**\
> *Hello RSA, my old friend!*
>
> **Attachments**\
> **[source](https://github.com/sajjadium/ctf-archives/tree/main/ctfs/Crypto/2022/polyRSA)**

Một bài RSA khá đơn giản để warmup😄. Các tham số $$(p, q)$$ được xây dựng như sau:

$$
\begin{align*}
    p &= k^{6} + 7k^{4} - 40k^{3} + 12k^{2} - 114k + 31377 \\
    q &= k^{5} - 8k^{4} + 19k^{3} - 313k^{2} - 14k + 14011 \\
\end{align*}, \quad k \sim 2^{64}
$$

Quy về giải phương trình (đa thức) một ẩn $$k$$: $$f(k) = p(k)q(k) - n$$. Có $$k$$ thì thay vô tìm được $$p, q$$ tới đây coi như xong!

```python
from sage.all import *
from Crypto.Util.number import long_to_bytes, bytes_to_long

n = [...]
enc = [...]

PR = PolynomialRing(ZZ, "k"); k = PR.gen()
pk = k**6 + 7*k**4 - 40*k**3 + 12*k**2 - 114*k + 31377
qk = k**5 - 8*k**4 + 19*k**3 - 313*k**2 - 14*k + 14011
fk = pk * qk - n
k0 = fk.roots()[0][0]

p, q = pk(k=k0), qk(k=k0)
d = pow(31337, -1, (p - 1)*(q - 1))
m = pow(enc, int(d), n)
print(long_to_bytes(m))
```

> **<gg>Flag: CCTF{F4C70r!N9_tRIcK5_aR3_fUN_iN_RSA?!!!}</gg>**

# Baphomet

> **Description**\
> *N/A*
>
> **Attachments**\
> **[source](https://github.com/sajjadium/ctf-archives/tree/main/ctfs/Crypto/2022/baphomet)**

Ngó sơ qua source:

```python
#!/usr/bin/env python3

from base64 import b64encode
from flag import flag

def encrypt(msg):
	ba = b64encode(msg.encode('utf-8'))
	baph, key = '', ''

	for b in ba.decode('utf-8'):
		if b.islower():
			baph += b.upper()
			key += '0'
		else:
			baph += b.lower()
			key += '1'

	baph = baph.encode('utf-8')
	key = int(key, 2).to_bytes(len(key) // 8, 'big')

	enc = b''
	for i in range(len(baph)):
		enc += (baph[i] ^ key[i % len(key)]).to_bytes(1, 'big')

	return enc

enc = encrypt(flag)
f = open('flag.enc', 'wb')
f.write(enc)
f.close()
```

Hmm... ta thấy một điều thú vị về tỷ lệ chiều dài (tính theo bytes) là:

- `len(baph)`:`len(enc)` = 1:1
- `len(baph)`:`len(key)` = 8:1

vậy `len(key)` = `len(enc)`:8 = 48:8 = 6. Để ý ta đã biết phần đầu của `flag` là `CCTF{` nên ta có thể tính được phần đầu của `baph`, nếu biết đủ 6 bytes thì hoàn toàn có thể tìm lại được giá trị `key`!

```python
from base64 import b64encode, b64decode
from pwn import xor

with open("./flag.enc", "rb") as f:
  enc = f.read()

# recover key
ba = b64encode(b"CCTF{")
baph = b""

for b in ba.decode():
  if b.islower(): baph += b.upper().encode()
  else: baph += b.lower().encode()

key = xor(enc[:6], baph[:6])

# recover flag
baph = b''
for i in range(len(enc)):
	baph += (enc[i] ^ key[i % len(key)]).to_bytes(1, 'big')

ba = ""
for b in baph.decode():
  if b.islower(): ba += b.upper()
  else: ba += b.lower()

print(b64decode(ba))
```

> **<gg>Flag: CCTF{UpP3r_0R_lOwER_17Z_tH3_Pr0bL3M}</gg>**

# aniely

> **Description**\
> *There is stream cipher, try to keep it smooth!*
>
> **Attachments**\
> **[source](https://github.com/sajjadium/ctf-archives/tree/main/ctfs/Crypto/2022/aniely)**

```python
#!/usr/bin/env python3

from struct import *
from os import *
from secret import passphrase, flag

def aniely_stream(passphrase):
	def mixer(u, v):
		return ((u << v) & 0xffffffff) | u >> (32 - v)

	def forge(w, a, b, c, d):
		for i in range(2):
			w[a] = (w[a] + w[b]) & 0xffffffff
			w[d] = mixer(w[a] ^ w[d], 16 // (i + 1))
			w[c] = (w[c] + w[d]) & 0xffffffff
			w[b] = mixer(w[b] ^ w[c], (12 + 2*i) // (i + 1))

	bring = [0] * 16
	bring[:4] = [0x61707865, 0x3320646e, 0x79622d32, 0x6b206574]
	bring[4:12] = unpack('<8L', passphrase)
	bring[12] = bring[13] = 0x0
	bring[14:] = [0] * 2

	while True:
		w = list(bring)
		for _ in range(10):
			forge(w, 0x0, 0x4, 0x8, 0xc)
			forge(w, 0x1, 0x5, 0x9, 0xd)
			forge(w, 0x2, 0x6, 0xa, 0xe)
			forge(w, 0x3, 0x7, 0xb, 0xf)
			forge(w, 0x0, 0x5, 0xa, 0xf)
			forge(w, 0x1, 0x6, 0xb, 0xc)
			forge(w, 0x2, 0x7, 0x8, 0xd)
			forge(w, 0x3, 0x4, 0x9, 0xe)
		for c in pack('<16L', *((w[_] + bring[_]) & 0xffffffff for _ in range(16))):
			yield c
		bring[12] = (bring[12] + 1) & 0xffffffff
		if bring[12] == 0:
			bring[13] = (bring[13] + 1) & 0xffffffff

def aniely_encrypt(msg, passphrase):
	if len(passphrase) < 32:
		passphrase = (passphrase * (32 // len(passphrase) + 1))[:32]
	rand = urandom(2) * 16
	return bytes(a ^ b ^ c for a, b, c in zip(msg, aniely_stream(passphrase), rand))

key = bytes(a ^ b for a, b in zip(passphrase, flag))
enc = aniely_encrypt(passphrase, key)
print(f'key = {key.hex()}')
print(f'enc = {enc.hex()}')
```

Tóm tắt:

- `key` = `passphrase` $$\oplus$$ `flag`
- `enc` = `passphrase` $$\oplus$$ `aniely_stream(key)` $$\oplus$$ `rand`

các giá trị `key` và `enc` ta đã biết do đó `aniely_stream(key)` tự tính được. `rand` là 2 bytes ngẫu nhiên ta cũng có thể bruteforce được. Tới đây tìm `flag` bằng cách:

$$
flag = key \oplus enc \oplus aniely\_steam(key) \oplus brute\_rand
$$

```python
from struct import *
from pwn import xor
from tqdm import tqdm

def aniely_stream(passphrase):
	def mixer(u, v):
		return ((u << v) & 0xffffffff) | u >> (32 - v)

	def forge(w, a, b, c, d):
		for i in range(2):
			w[a] = (w[a] + w[b]) & 0xffffffff
			w[d] = mixer(w[a] ^ w[d], 16 // (i + 1))
			w[c] = (w[c] + w[d]) & 0xffffffff
			w[b] = mixer(w[b] ^ w[c], (12 + 2*i) // (i + 1))

	bring = [0] * 16
	bring[:4] = [0x61707865, 0x3320646e, 0x79622d32, 0x6b206574]
	bring[4:12] = unpack('<8L', passphrase)
	bring[12] = bring[13] = 0x0
	bring[14:] = [0] * 2

	while True:
		w = list(bring)
		for _ in range(10):
			forge(w, 0x0, 0x4, 0x8, 0xc)
			forge(w, 0x1, 0x5, 0x9, 0xd)
			forge(w, 0x2, 0x6, 0xa, 0xe)
			forge(w, 0x3, 0x7, 0xb, 0xf)
			forge(w, 0x0, 0x5, 0xa, 0xf)
			forge(w, 0x1, 0x6, 0xb, 0xc)
			forge(w, 0x2, 0x7, 0x8, 0xd)
			forge(w, 0x3, 0x4, 0x9, 0xe)
		for c in pack('<16L', *((w[_] + bring[_]) & 0xffffffff for _ in range(16))):
			yield c
		bring[12] = (bring[12] + 1) & 0xffffffff
		if bring[12] == 0:
			bring[13] = (bring[13] + 1) & 0xffffffff

key = bytes.fromhex("4dcceb8802ae3c45fe80ccb364c8de19f2d39aa8ebbfb0621623e67aba8ed5bc")
enc = bytes.fromhex("e67a67efee3a80b66af0c33260f96b38e4142cd5d9426f6f156839f2e2a8efe8")

stream = aniely_stream(key)
stream = bytes([next(stream) for _ in range(32)])

for b1 in range(256):
	for b2 in range(256):
		rand = bytes([b1, b2]) * 16
		passphrase =  bytes(a ^ b ^ c for a, b, c in zip(enc, stream, rand))
		flag = bytes(a ^ b for a, b in zip(passphrase, key))

		if flag.startswith(b'CCTF{') and flag.endswith(b'}'):
			print("Flag:", flag.decode())
			exit()
```
> **<gg>Flag: CCTF{7rY_t0_D3cRyPT_z3_ChaCha20}</gg>**

# cantilever

> **Description**\
> *What if you can find the message? if you can, that means you are genius, because we harden our crypto system with a very modern tool!*
>
> **Attachments**\
> **[source](https://github.com/sajjadium/ctf-archives/tree/main/ctfs/Crypto/2022/cantilever)**

Thêm một bài nữa về RSA😅Để ý hàm `gen_primes` dùng để sinh private key

```python
def gen_primes(nbit, imbalance):
	p = 2
	FACTORS = [p]
	while p.bit_length() < nbit - 2 * imbalance:
		factor = getPrime(imbalance)
		FACTORS.append(factor)
		p *= factor
	rbit = (nbit - p.bit_length()) // 2

	while True:
		r, s = [getPrime(rbit) for _ in '01']
		_p = p * r * s
		if _p.bit_length() < nbit: rbit += 1
		if _p.bit_length() > nbit: rbit -= 1
		if isPrime(_p + 1):
			FACTORS.extend((r, s))
			p = _p + 1
			break
```

hàm này trả về số nguyên $$p$$ thỏa mãn:

$$
\max\{p_{k} \quad\text{that}\quad p_{k}|p - 1 \text{  and  } p_{k} \in \mathbb{P}\} \sim 2^{\epsilon}
$$

với $$\epsilon$$ là giá trị tham số `imbalance`. Khi giá trị $$\epsilon$$ nhỏ (ở bài này người ta chọn $$\epsilon \le 20$$) thì những số như vậy được gọi là $$\epsilon-$$[smooth number](https://en.wikipedia.org/wiki/Smooth_number). Với cách sinh private key như vậy, tồn tại thuật toán có thể phá được là [Pollard's p - 1](https://en.wikipedia.org/wiki/Pollard%27s_p_%E2%88%92_1_algorithm).

Như vậy dễ dàng phân tích được $n$ ra thừa số bằng cách:

```python
from sage.all import *
n = [...]

# Pollard p - 1 factorization
p = gcd(n, pow(2, factorial(2**19), n) - 1)
q = n//p
```

Sau khi có private key ta cần giải quyết 2 bài toán sau:

- **Part 1 - Solving RSA:** giải quyết dễ dàng khi đã có $$p$$ và $$q$$.
- **Part 2 - Solving DLP:** cần tìm $$m_{2}$$ thỏa mãn

$$
c_{2} = e^{m_{2}} \pmod{n}
$$

Dùng [CRT](https://en.wikipedia.org/wiki/Chinese_remainder_theorem) ta có thể chia nhỏ thành 2 bài toán:

$$
\begin{cases}
  c_{2} = e^{m_{2}} \pmod{p} \\
  c_{2} = e^{m_{2}} \pmod{q}
\end{cases}
$$

cả hai bài trên đều giải quyết dàng bằng thuật toán [Pohlig Hellman](https://en.wikipedia.org/wiki/Pohlig%E2%80%93Hellman_algorithm) (do $$p-1$$ và $$q-1$$ đều smooth!)

```python
from sage.all import *
from Crypto.Util.number import long_to_bytes, bytes_to_long

n = [...]
c_1 = [...]
c_2 = [...]
e = 0x10001

# Pollard p - 1 factorization
p = gcd(n, pow(2, factorial(2**19), n) - 1)
q = n//p

# part 1
d  = pow(e, -1, (p - 1)*(q - 1))
m1 = pow(c_1, d, n)

# part 2
Fp, Fq = GF(p), GF(q)
mp2 = discrete_log(Fp(c_2), Fp(e), operation='*')
mq2 = discrete_log(Fq(c_2), Fq(e), operation='*')
m2  = crt([mp2, mq2], [p - 1, q - 1])

# final flag
flag = long_to_bytes(m1) + long_to_bytes(m2)
print("Flag:", flag.decode())
```

> **<gg>Flag: CCTF{5L3Ek_4s__s1lK__Ri9H7?!}</gg>**

# larisa & lagima

## Symmetric Group

Vì cả hai bài này đều liên quan tới [Symmetric Group](https://en.wikipedia.org/wiki/Symmetric_group) nên mình sẽ gộp chung lại. Đầu tiên nhắc lại định nghĩa: 

"Gọi $$\Omega$$ là tập bất kì khác rỗng (thường xét tập $$\Omega = \{1, 2, \cdots, n\}$$) và $$S_{\Omega}$$ là tập tất cả các song ánh $$f: \Omega \rightarrow \Omega$$ (hay $$f$$ là một hàm hoán vị $$\Omega$$). Khi đó $$\left(S_{\Omega}, \circ\right)$$ (với $$\circ$$ là [Function Composition](https://en.wikipedia.org/wiki/Function_composition)) lập thành nhóm gọi là **Symmetric Group**, ký hiệu là $$\mathbb{S_{n}}$$"

Một số lưu ý:
- Số lượng phần tử: $$\|\mathbb{S_{n}}\| = n!$$
- [Disjoint Cycle Notation](https://en.wikipedia.org/wiki/Permutation#Cycle_notation): 

$$
G = \left(a_{1}, a_{2}, \cdots, a_{m_{1}}\right)
\left(a_{m_{1}+1}, a_{m_{1}+2}, \cdots, a_{m_{2}}\right)\cdots
\left(a_{m_{k-1}+1}, a_{m_{k-2}+2}, \cdots, a_{m_{k}}\right)
$$

- Order of an Element: $$\|G\| = \operatorname{lcm}\left(m_{1}, m_{2}, \cdots, m_{k}\right)$$

## larisa

Ở bài này chúng ta cần tìm $$M = \left(m_{1}, m_{2}, \cdots, m_{n}\right)$$, khi biết

$$
C = \left(c_{1}, c_{2}, \cdots, c_{n}\right) = \left(m_{1}^{e}, m_{2}^{e}, \cdots, m_{n}^{e}\right)
$$

với $$(n, e) = (128, 65537)$$ và $$m_{i} \in \mathbb{S_{n}}$$. Thoạt nhìn thấy hơi giống RSA... đúng vậy, ý tưởng hoàn toàn tương tự, ta sẽ tính $$d = e^{-1} \pmod{\|\mathbb{S_{n}}\| = n!}$$, khi đó $$m_{i}$$ được tính bằng công thức:

$$
m_{i} = c_{i}^{d}
$$

```python
from sage.all import *

l, e = 128, 65537
S = SymmetricGroup(l)
d = pow(e, -1, S.order())

if __name__ == "__main__":
  # recover `M`
  C = eval(open("./enc.txt", "r").read())  
  M = [
    list((S(c)**d).tuple()) for c in C
  ]

  # recover `flag``
  decrypt = lambda M, r, s: "".join([chr(m[(_*r + s)%l]) for _, m in enumerate(M)])
  for r in range(2, l):
    for s in range(2, l):
      flag = decrypt(M, r, s)
      if "CCTF{" in flag:
        print(flag, r, s)
```

> **<gg>Flag: CCTF{pUbliC_k3y_crypt0graphY_u5in9_rOw-l4t!N_5quAr3S!}</gg>**

## lagima

Bài này chúng ta cần tìm $$x$$ khi biết:

$$
\begin{cases}
	G = \left(g_{1}, g_{2}, \cdots, g_{n}\right) \\
	C = \left(h_{1}, h_{2}, \cdots, h_{n}\right) = \left(m_{1}^{x}, m_{2}^{x}, \cdots, m_{n}^{x}\right)
\end{cases}
$$

với $$n = 313$$ và $$g_{i}, h_{i} \in \mathbb{S_{n}}$$. 

Hồi nãy giống RSA giờ giống DLP... ha😅, khác với việc tính DLP trên $$\mathbb{F_{p}}$$ thì trên $$\mathbb{S_{n}}$$ có sẵn công thức tìm order của phần tử trong nhóm (xuất phát từ thuật toán biểu diễn Disjoint Cycle) nên việc tính DLP khá nhanh! Sau khi giải DLP xong ta sẽ có hệ

$$
\begin{cases}
	x &= \log_{g_{1}}\left(h_{1}\right) &\pmod{|g_{1}|} \\
	x &= \log_{g_{2}}\left(h_{2}\right) &\pmod{|g_{2}|} \\
		&\vdots \\
	x &= \log_{g_{n}}\left(h_{n}\right) &\pmod{|g_{n}|}
\end{cases}
$$

giải hệ này bằng [CRT](https://en.wikipedia.org/wiki/Chinese_remainder_theorem) có ngay $$x$$ cũng chính là `flag` cần tìm😄

```python
from sage.all import *
from Crypto.Util.number import *

n = 313
S = SymmetricGroup(n)

if __name__ == "__main__":
  with open("./output.txt", "r") as f:
    G = eval(f.readline()[4:])
    H = eval(f.readline()[4:])
  
  x = crt(
    [discrete_log(S(h), S(g)) for h, g in zip(H, G)],
    [S(g).order() for g in G]
  )
  print(long_to_bytes(x))
```

> **<gg>Flag: CCTF{3lGam4L_eNcR!p710n_4nD_L4T!n_5QuarS3!}</gg>**