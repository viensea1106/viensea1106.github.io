---
title:  "HITCON CTF 2022"
mathjax: true
layout: post
---

Writeup some Crypto challenges in [HITCON CTF 2022](https://ctftime.org/event/1772/):

- babySSS
- secret




# babySSS

```python
from random import SystemRandom
from Crypto.Cipher import AES
from hashlib import sha256
from secret import flag

rand = SystemRandom()


def polyeval(poly, x):
    return sum([a * x**i for i, a in enumerate(poly)])


DEGREE = 128
SHARES_FOR_YOU = 8  # I am really stingy :)

poly = [rand.getrandbits(64) for _ in range(DEGREE + 1)]
shares = []
for _ in range(SHARES_FOR_YOU):
    x = rand.getrandbits(16)
    y = polyeval(poly, x)
    shares.append((x, y))
print(shares)

secret = polyeval(poly, 0x48763)
key = sha256(str(secret).encode()).digest()[:16]
cipher = AES.new(key, AES.MODE_CTR)
print(cipher.encrypt(flag))
print(cipher.nonce)
```

Một câu về [Secret Sharing Scheme](https://en.wikipedia.org/wiki/Secret_sharing#:~:text=A%20secret%2Dsharing%20scheme%20can,the%20shares%20among%20the%20participants), đầu tiên đề generate ra một đa thức $$f(x)$$ bậc $$n = 128$$ và $$8$$ bộ số $$\left(x_{i}, y_{i}=f(x_{i})\right)$$ thỏa mãn:

$$
f\left(x\right) = a_{0} + a_{1}x + a_{2}x^{2} + \cdots + a_{n}x^{n}
$$

với $$f \in \mathbb{Z}[x]$$ và $$a_{i} \sim 2^{64} \quad\forall i \in \overline{0, n}$$. Nhiệm vụ là khôi phục lại đa thức $$f$$ và tìm `secret_key` = $$f(0x48763)$$ để giải mã AES. Để ý rằng ta đang làm việc trên $$\mathbb{Z}$$ nên ta có tính chất thú vị sau:

$$
\begin{cases}
  y_{1} &= f(x_{1}) = a_{0} + a_{1}x_{1} + a_{2}x^{2}_{1} + \cdots + a_{n}x^{n}_{1} = a_{0} &\pmod{x_{1}}\\
  y_{2} &= f(x_{2}) = a_{0} + a_{1}x_{2} + a_{2}x^{2}_{2} + \cdots + a_{n}x^{n}_{2} = a_{0} &\pmod{x_{2}}\\
        &\vdots  \\                                
  y_{8} &= f(x_{8}) = a_{0} + a_{1}x_{8} + a_{2}x^{2}_{8} + \cdots + a_{n}x^{n}_{8} = a_{0} &\pmod{x_{8}}
\end{cases}
$$

Giải hệ trên bằng CRT ta tìm được $$a_{0} \pmod{\lambda}$$ với $$\lambda = lcm(x_{1}, x_{2}, \cdots, x_{8})$$. Tính toán trực tiếp ta thấy $$\lambda ~ 2^{102}$$ nên ta tìm được chính xác giá trị $$a_{0}$$ sau bước tính CRT. Tới đây ta xây dựng thuật toán recover toàn bộ đa thức $$f$$ như sau:

- Giả sử hiện tại ta đã tìm được các giá trị $$(a_{0}, a_{1}, \cdots, a_{i})$$.
- Đặt $$f_{i}(x) = a_{0} + a_{1}x + a_{2}x^{2} + \cdots a_{i}x^{i}$$, khi đó ta có:

$$
\begin{cases}
  \dfrac{y_{1} - f_{i}(x_{1})}{x_{1}^{i}} &= a_{i+1} + a_{i+2}x_{1} + \cdots a_{n}x^{n - i - 1}_{1} = a_{i+1} &\pmod{x_{1}}\\
  \dfrac{y_{2} - f_{i}(x_{2})}{x_{2}^{i}} &= a_{i+1} + a_{i+2}x_{2} + \cdots a_{n}x^{n - i - 1}_{2} = a_{i+1} &\pmod{x_{2}}\\
        &\vdots  \\      
  \dfrac{y_{8} - f_{i}(x_{8})}{x_{8}^{i}} &= a_{i+1} + a_{i+2}x_{8} + \cdots a_{n}x^{n - i - 1}_{8} = a_{i+1} &\pmod{x_{8}}\\
\end{cases}
$$

- Tương tự sử dụng CRT ta cũng tìm được $$a_{i+1}$$

```python
from sage.all import *
from hashlib import sha256
from Crypto.Cipher import AES

def polyeval(poly, x):
    return sum([a * x**i for i, a in enumerate(poly)])

shares = [(41458, 3015894889650529600470920314593280408459518223054415623846810748413393737686521849609926975694824777687791824408686652245102687392987299828716863372946074882798754477101786150262288970710451710086966378817944448615584285684364802621112755627795146504720812935041851556318832824799502759754100408717888912062197676588256634343721633045179136302533777168978134770315363985448879229514802330846792965525004570768212871252658334277172395338054448791891165981203069346039654617938169527772805687564575525262812469960675835101499054296722994451502140787064163668418661661374437567033971648550576296023422536253955229), (3389, 188433716494377932944071544153838579057591833387651830021721770473524507947811754295899393634645349682360212761145039355690817927625249659010181081209481357850193656763556243022791637306094953982811471415645267589939465925098159204147714779617946431727015863707468081949286110249296858079354949234074465541940264775783884708819566758872542606519408358277173683256608326688673226933790117016596834640875497643330432185114931410656582728964222203181026468387428893233826461), (20016, 100434774699078525844435127144579870564983915777345068724291926367405061427748836490810414860997895358378538088786283372231649911113841061354335739776409724471256377867811133591349442950556374825868587940833009529662869081130218551306459690738900795035660420986807973542512081415453215211908130387754214098414826747340962722685373241806099462750595976574593799013733614097923338311883793416643213898201680852118540438376386415411317989072583126108177482838299109479175882214603698768498421016054035672774286507312986602290254323930575001551875601243671354491241420409219), (50683, 444545881882748849210617532697661279371689521082184772844723908765173319859389018743414369945234307906596253496624659734919646710483514374218993496994560985318096082923429834553341897367168830049334302307406087637232329348570485341223211629167329394484624055745054495405880099706580380696671879365741197827080224977821589102425678989782880274304484630899425664722718972847034030888019348402685383311095030884356731112886316823960378572796288532824588478234949384868912708000223119984161992105752059185137674711077940232530298853451166664700609238496874366152042676602089571801873748042888046623717879084695143810047335029), (6445, 101461065764578261241074518788237888467081270902741849861528201922043223477790661159690684156056890167304291810116447916457265705130707166062372766839626095333813681671546097679623755546322833727082145873422243641505450049118758544298328784536759107951763715458884889255549767465897671061295486677353893450789955616926292534325337544782386120469581214993770910137353221116457111551538222138388416162630076391624447865248920466274175229034129561913505977209131490066291917549232913771218316393849495621818397), (1359, 301175604076484656987097022479686300460199620068959954988990822483114048418823291831080744590394713639405681060973359346474547015206086229256524657214311815578895906855833813636970640902962286472992468394831014254279137613828904924898823470285520515090889491445149243620044782726415898188702226878029241518020146726699446397961112596830223444821094650508662477147134721631935528182772284099429814417490160457082241680661), (45286, 244867719210730952183489456726726432791149629831242968845409984537752132549250274779516590253042559196452609852176114909791657154092483479876795482861784431886143414585698773882088948703730268947925790809436449512089696895048994874003651088538416399435467483409931121063976149037130454114161175715871108284419975118570732022104749321213013756795645219060997019373915339235627535694458093194617642834806820772479160496966470147893963746139947337914575231526069667124822677688977724313174612816604463495630041075005651663546036363128325535621487658461744362098985183050127661470315454320073092665472364666768205258769), (5649, 4766101906865350375503575239791521167258753430948472304582908507542293595346756303331383584550516424087839316050412570112796817549423179461056531056102741963677007097061600281918678364910813585444151640384802648969082273001142879806475184857246441212406056540028447374033197873299250076862108042582790928405869475508762352345569281589853917902601519294573327847401601789315980414998055948162169170771240383220643819333682845459742335249254576151835966500230706707674854493184181354958093926469960861)]
ct = b'G$\xf5\x9e\xa9\xb1e\xb5\x86w\xdfz\xbeP\xecJ\xb8wT<<\x84\xc5v\xb4\x02Z\xa4\xed\x8fB\x00[\xc0\x02\xf9\xc0x\x16\xf9\xa4\x02\xb8\xbb'
nonce = b'\x8f\xa5z\xb4mZ\x97\xe9'
DEGREE = 128

if __name__ == "__main__":
  poly = []
  for i in range(129):
    Xs, Ms = [], []
    for x, y in shares:
      Ms.append(x)
      Xs.append((y - polyeval(poly, x))//x**i)
    poly.append(ZZ(crt(Xs, Ms)))

  secret = polyeval(poly, 0x48763)
  key = sha256(str(secret).encode()).digest()[:16]
  cipher = AES.new(key, AES.MODE_CTR, nonce=nonce)
  print(cipher.decrypt(ct))
```

> **<gg>Flag: hitcon{doing_SSS_in_integers_is_not_good_:(}</gg>**

# secret

```python
import random, os
from Crypto.Util.number import getPrime, bytes_to_long

p = getPrime(1024)
q = getPrime(1024)
n = p * q

flag = open('flag','rb').read()
pad_length = 256 - len(flag)
m = bytes_to_long(os.urandom(pad_length) + flag)
assert(m < n)
es = [random.randint(1, 2**512) for _ in range(64)]
cs = [pow(m, p + e, n) for e in es]
print(es)
print(cs)
```

Source khá ngắn nhưng cũng không dễ xơi chút nào 😁 Đầu tiên ta được cấp $$64$$ cặp $$\left(e_{i}, c_{i}\right)$$ thỏa mãn:

$$
c_{i} = m^{p + e_{i}} \pmod{n}, \quad\forall i \in \overline{1, 64}
$$

trong đó $$m$$ là `flag` cần tìm, $$n = p \times q$$ chưa biết. Vấn đề đặt ra trước tiên là nghĩ cách khôi phục $$n$$. Ta thử nghĩ cách kết hợp các dữ kiện lại với nhau, đầu tiền gọi $$\left(x_{1}, x_{2}, \cdots, x_{64}\right) \in \mathbb{Z}^{64}$$ là một tổ hợp nào đó, khi đó kết hợp các dữ kiện lại ta được:

$$
\prod_{i=1}^{64}c_{i}^{x_{i}} = \prod_{i=1}^{64}m^{x_{i}(p + e_{i})} \pmod{n} \tag{1}
$$

Hmm... nếu như ta chọn bộ $$\left(x_{1}, x_{2}, \cdots, x_{64}\right)$$ thỏa mãn:

$$
\begin{cases}
  x_{1} + x_{2} + \cdots + x_{n} &= 0 \\
  e_{1}x_{1} + e_{2}x_{2} + \cdots + e_{n}x_{n} &= 0 \\
\end{cases} \tag{2}
$$

thì $$(1)$$ suy ra

$$
\prod_{i=1}^{64}c_{i}^{x_{i}} = 1 \pmod{n}
$$

Tới đây ta lần lượt giải quyết các vấn đề sau:

- <b>Tìm bộ số $$(x_{i})_{i=1}^{64}$$:</b> bản chất đây tất cả bộ số này thỏa mãn

$$
A
\begin{bmatrix}
  x_{1} \\ x_{2} \\ \vdots \\ x_{64}
\end{bmatrix} = 
\begin{bmatrix}
  0 & 0
\end{bmatrix}, \qquad \text{where } A =
\begin{bmatrix}
  1     & 1     & \cdots & 1     \\
  e_{1} & e_{2} & \cdots & e_{64}
\end{bmatrix}
$$

và tạo thành không gian vector (Null Space hoặc Kernel): $$N(A) = \ker(A) = \{\mathbf{x} \in \mathbf{Z}^{64} \text{ }\vert\text{ } A\mathbf{x} = 0\}$$ Đã có thuật toán tìm ma trận cơ sở của Kernel (trong sagemath có hàm `right_kernel_matrix`). Vậy không những tìm ra một bộ mà ta ta còn tìm được cơ sở sinh ra không gian các vector $$(x_{i})_{i=1}^{64}$$ thỏa mãn như vậy, giả sử ma trận cơ sở đó là:

$$
B = 
\begin{bmatrix}
  b_{1,1} & b_{1,2} & \cdots & b_{1,64}     \\
  b_{2,1} & b_{2,2} & \cdots & b_{2,64}     \\
  \cdots  & \cdots  & \cdots & \cdots       \\
  b_{m,1} & b_{m,2} & \cdots & b_{m,64}     \\
\end{bmatrix}
$$

- <b>Xử lý số lớn:</b> trường hợp $$x_{i}$$ quá lớn thì ta không thể tính trực tiếp $$c_{i}^{x_{i}}$$ được, cần nghĩ cách "thu gọn" bộ này lại... nói tới đây chắc mọi người đều nghĩ tới Lattice rồi ha 😂. Do bộ số $$x_{i}$$ được sinh từ cơ sở $$B$$ nên nếu ta áp dụng LLL lên B thì có thể rút gọn các số $$x_{i}$$ nhỏ lại mà vẫn đảm bảo thỏa điều kiện $$(2)$$

- <b>Khôi phục $$n$$:</b> gọi $$B'$$ là ma trận sau khi rút gọn LLL của $$B$$, lúc này với mỗi $$i, j$$ bất kì ta có:

$$
n \| \gcd\left(\prod_{b_{i, k} < 0}c_{k}^{-b_{i, k}} - \prod_{b_{i, k} > 0}c_{k}^{b_{i, k}}, \prod_{b_{j, l} < 0}c_{l}^{-b_{j, l}} - \prod_{b_{j, l} > 0}c_{l}^{b_{j, l}}\right)
$$

sau khi tính $$\gcd$$ ta loại bỏ các ước nhỏ chừng nào phần còn lại $$\le 2048$$ bits là tìm được chính xác $$n$$.

- <b>Tìm $$m$$:</b> có $$n$$ thì quay lại dạng quen thuộc Common Modulus Attack 😁 Khi đó ta chỉ việc tìm bộ 3 $$(i, j, k)$$ thỏa mãn $$\gcd\left(e_{i} - e_{j}, e_{i} - e_{k}\right) = 1$$ thì lúc này dùng thuật toán Euclid mở rộng tìm được $$u, v$$ sao cho:

$$
u(e_{i} - e_{j}) + v(e_{i} - e_{k}) = 1
$$

vậy $$m$$ lúc này sẽ là

$$
\left(\dfrac{c_{i}}{c_{j}}\right)^{u} \times \left(\dfrac{c_{i}}{c_{k}}\right)^{v} = \left(m^{e_{i} - e_{j}}\right)^{u} \times \left(m^{e_{i} - e_{k}}\right)^{v} = m \pmod{n}
$$

```python
from sage.all import *
from Crypto.Util.number import *
from tqdm import tqdm

def gen_kN(cs, row):
  lhs, rhs = 1, 1
  for r, c in zip(row, cs):
    if r < 0: lhs *= c**(-r)
    else: rhs *= c**r

  return lhs - rhs

if __name__ == "__main__":
  with open("./output.txt", "r") as f:
    es = eval(f.readline())
    cs = eval(f.readline())
    n  = 64

    A   = matrix(ZZ, es).stack(vector(ZZ, [1]*n))
    ker = A.right_kernel_matrix()

    N = 0
    for row in tqdm(ker.LLL()):
      N = gcd(N, gen_kN(cs, row))
      N = factor(N, limit=2**20)[-1][0]
      if N.nbits() <= 2048:
        break

    # print(N)
    Zn = Zmod(N)
    for i in range(1, n):
      for j in range(i+1, n):
        if gcd(es[0] - es[i], es[0] - es[j]) == 1:
          _, u, v = xgcd(es[0] - es[i], es[0] - es[j])
          m = (Zn(cs[0])/ Zn(cs[i]))**u * (Zn(cs[0])/ Zn(cs[j]))**v
          print(long_to_bytes(int(m)))
          exit(0)
```

> **<gg>Flag: hitcon{K33p_ev3rythIn9_1nd3p3ndent!}</gg>**