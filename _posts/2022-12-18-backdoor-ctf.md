---
title:  "BackdoorCTF 2022"
mathjax: true
layout: post
---

[BackdoorCTF 2022](https://ctftime.org/event/1796) writeup some Crypto challenges:
- Fishy




# Fishy

```python
from random import getrandbits as grb
from Crypto.Util.number import bytes_to_long as bl

modulus = pow(2, 32)
s_boxes = [[grb(32) for i in range(256)] for j in range(4)]
f = open("s_boxes.txt", "w")
f.write(str(s_boxes))
f.close()
initial_sub_keys = [
  "243f6a88",
  "85a308d3",
  "13198a2e",
  "03707344",
  "a4093822",
  "299f31d0",
  "082efa98",
  "ec4e6c89",
  "452821e6",
  "38d01377",
  "be5466cf",
  "34e90c6c",
  "c0ac29b7",
  "c97c50dd",
  "3f84d5b5",
  "b5470917",
  "9216d5d9",
  "8979fb1b",
]
key = "".join([hex(grb(32))[2:].zfill(8) for i in range(18)])
f = open("key.txt", "w")
f.write(str(key))
f.close()
processed_sub_keys = [
    hex(int(initial_sub_keys[i], 16) ^ int(key[8 * i : 8 * (i + 1)], 16))[2:].zfill(8)
    for i in range(len(initial_sub_keys))
]
f = open("processed_keys.txt", "w")
f.write(str(processed_sub_keys))
f.close()
pt = bin(bl(b"flag{th3_f4k3_fl4g}"))[2:]
while len(pt) % 64 != 0:
    pt = "0" + pt
pt = hex(int(pt, 2))[2:].zfill(len(pt) // 16)
ct = ""
for i in range(len(pt) // 16):
    xl = pt[16 * i : 16 * i + 8]
    xr = pt[16 * i + 8 : 16 * i + 16]
    # rounds
    for j in range(16):
        tmp = xl
        xl = bin(int(xl, 16) ^ int(processed_sub_keys[j], 16))[2:].zfill(32)
        xa = int(xl[:8], 2)
        xb = int(xl[8:16], 2)
        xc = int(xl[16:24], 2)
        xd = int(xl[24:32], 2)
        xa = (s_boxes[0][xa] + s_boxes[1][xb]) % modulus
        xc = s_boxes[2][xc] ^ xa
        f_out = (xc + s_boxes[3][xd]) % modulus
        xl = hex(int(xr, 16) ^ f_out)[2:].zfill(8)
        xr = tmp
    xrt = xr
    xr = hex(int(xl, 16) ^ int(processed_sub_keys[16], 16))[2:].zfill(8)
    xl = hex(int(xrt, 16) ^ int(processed_sub_keys[17], 16))[2:].zfill(8)
    ct += xl + xr
    f.write(str(xl) + str(xr) + "\n")
print(ct)
```

Nhìn code hơi loằng ngoằng nhỉ, bản chất bài này gồm có 2 phần chính như sau:

<b>Key Generation:</b> khóa là mảng các số nguyên $$32$$ bits được sinh "ngẫu nhiên" bằng thư viện [random](https://docs.python.org/3/library/random.html) của python.

- `s_boxes`: bao gồm 4 $$\times$$ 256 = 1024 số $$32$$ bits và ta được cho trước các số này.
- `processed_sub_keys`: cũng được sinh ra từ hàm `random.getrandbits(32)`.

Lưu ý: thư viện `random` của python sử dụng thuật toán [Mersenne Twister - mt19937](https://en.wikipedia.org/wiki/Mersenne_Twister), đây là [PRNG](https://en.wikipedia.org/wiki/Pseudorandom_number_generator) và có thể crack được nếu như biết đủ nhiều output (ở đây cần biết đủ $$624$$ output $$32$$ bits liên tiếp).

Vậy từ các giá trị `s_boxes` ta có thể crack được hàm random (tức là có thể predict được các giá trị tiếp theo), từ đó biết được giá trị của `processed_sub_keys`. Ở đây mình có dùng tool [RandCrack](https://github.com/tna0y/Python-random-module-cracker). 

<b>Block Cipher:</b> Như vậy ta có thể khôi phục lại `key` dùng cho quá trình mã hóa, vấn đề còn lại là hiểu thuật toán mã hóa và giải mã nó. Sau một hồi ngâm cứu thì mình phát hiện nó dựa theo cấu trúc của [Feistel Cipher](https://en.wikipedia.org/wiki/Feistel_cipher)😀

<p algin="center">
  <img src="/images/2022-backdoor-ctf/feistel-cipher.png">
</p>

Nhìn vô sơ đồ thì "dễ thấy" cách giải mã rồi ha😅

```python
from Crypto.Util.number import *
from randcrack import RandCrack # pip install randcrack

modulus = pow(2, 32)
initial_sub_keys = [
  "243f6a88",
  "85a308d3",
  "13198a2e",
  "03707344",
  "a4093822",
  "299f31d0",
  "082efa98",
  "ec4e6c89",
  "452821e6",
  "38d01377",
  "be5466cf",
  "34e90c6c",
  "c0ac29b7",
  "c97c50dd",
  "3f84d5b5",
  "b5470917",
  "9216d5d9",
  "8979fb1b",
]

def crack_grb(s_boxes):
  Ys = sum(s_boxes, [])
  rc = RandCrack()

  for _ in range(624):
    rc.submit(Ys[_])

  for i in range(624, len(Ys)):
    assert rc.predict_getrandbits(32) == Ys[i]
  
  return rc

def F(x, key):
  x = bin(int(x, 16) ^ int(key, 16))[2:].zfill(32)
  xa = int(x[:8]   , 2)
  xb = int(x[8:16] , 2)
  xc = int(x[16:24], 2)
  xd = int(x[24:32], 2)
  xa = (s_boxes[0][xa] + s_boxes[1][xb]) % modulus
  xc = s_boxes[2][xc] ^ xa
  return (xc + s_boxes[3][xd]) % modulus

def decrypt(ct):
  pt = ""
  for i in range(len(ct) // 16):
    xl = hex(int(ct[16*i + 8 : 16*i + 16], 16) ^ int(processed_sub_keys[16], 16))[2:].zfill(8)
    xr = hex(int(ct[16*i : 16*i + 8], 16)      ^ int(processed_sub_keys[17], 16))[2:].zfill(8)
    # rounds
    for j in range(15, -1, -1):
      tmp = xr
      xr = hex(int(xl, 16) ^ F(xr, processed_sub_keys[j]))[2:].zfill(8)
      xl = tmp
    pt += xl + xr

  return pt

if __name__ == "__main__":
  with open("./s_boxes.txt", "rb") as f:
    s_boxes = eval(f.read())

  grb = crack_grb(s_boxes)
  key = "".join([hex(grb.predict_getrandbits(32))[2:].zfill(8) for i in range(18)])
  processed_sub_keys = [
    hex(int(initial_sub_keys[i], 16) ^ int(key[8 * i : 8 * (i + 1)], 16))[2:].zfill(8)
    for i in range(len(initial_sub_keys))
  ]
  
  ct = [...]
  flag = decrypt(ct)
  print(bytes.fromhex(flag))
```

> **<gg>Flag: flag{d0n't_u53_th3_m3r53nn3_r4nd0m_g3n3r4t0r_w1th0ut_c4ut10n_<3}</gg>**

# RandomNonsense