---
title:  "ASIS CTF 2022"
mathjax: true
layout: post
---

Writeup [ASIS CTF Quals 2022](https://ctftime.org/event/1574/) for two challenges:

- \[Crypto\] Desired Curve 
- \[Android\] figole




# Desired Curve

```python
#!/usr/bin/env sage

import sys
from Crypto.Util.number import *
from flag import flag

def die(*args):
	pr(*args)
	quit()

def pr(*args):
	s = " ".join(map(str, args))
	sys.stdout.write(s + "\n")
	sys.stdout.flush()

def sc():
	return sys.stdin.buffer.readline()

def main():
	border = "|"
	pr(border*72)
	pr(border, "Hi all, now it's time to solve a relatively simple challenge about  ", border)
	pr(border, "relatively elliptic curves! We will generate an elliptic curve with ", border)
	pr(border, "your desired parameters, are you ready!?                            ", border)
	pr(border*72)

	nbit = 256
	q = getPrime(nbit)
	F = GF(q)

	while True:
		pr(border, "Send the `y' element of two points in your desired elliptic curve:  ")
		ans = sc()
		try:
			y1, y2 = [int(_) % q for _ in ans.split(b',')]
		except:
			die(border, "Your parameters are not valid! Bye!!")
		A = (y1**2 - y2**2 - 1337**3 + 31337**3) * inverse(-30000, q) % q
		B = (y1**2 - 1337**3 - A * 1337) % q
		E = EllipticCurve(GF(q), [A, B])
		G = E.random_point()

		m = bytes_to_long(flag)
		assert m < q
		C = m * G
		pr(border, f'The parameters and encrypted flag are:')
		pr(border, f'q = {q}')
		pr(border, f'G = ({G.xy()[0]}, {G.xy()[1]})')
		pr(border, f'm * G = ({C.xy()[0]}, {C.xy()[1]})')

		pr(border, f'Now find the flag :P')

if __name__ == '__main__':
	main()
```

Một bài Elliptic Curve khá thú vị, đầu tiên ta `server` generate số nguyên tố $$q-256$$ bits, sau đó ta được query vô hạn lần, mỗi lần thực hiện các thao tác sau:
- Đầu tiên gửi cho `server` hai số nguyên $$\left(y_{1}, y_{2}\right)$$.
- Tiếp theo `server` sẽ tính $$2$$ tham số $$A, B$$ của Curve $$E\left(\mathbb{F}_{q}\right): y^{2} = x^{3} + Ax + B$$ thỏa mãn $$E$$ đi qua hai điểm $$\left(31337, y_{1}\right), \left(1337, y_{2}\right)$$:

$$
\begin{cases}
  A = \dfrac{y_{2}^{2} + 1337^{3} - y_{1}^{2} - 31337^{3}}{31337 - 1337} \\
  B = y_{1}^{2} - 1337^{3} - 1337A
\end{cases} \pmod{q}
$$

- Cuối cùng `server` generate một điểm random $$G$$ và trả cho ta kể quả $$\left(G, m \times G\right)$$ với $$m$$ là `flag`.

Rõ ràng ta phải giải quyết bài toán tính DLP trên Curve, vì Curve ở đây được chọn ngẫu nhiên (tùy theo giá trị $$y_{i}$$ mình chọn) nên sẽ có những trường hợp order của Curve chứa các ước nguyên tố nhỏ (ta tận dụng những trường hợp ngẫu nhiên này vì khá khó để chọn $$y_{i}$$ sao cho order của Curve hoàn toàn smooth). Vậy ý tưởng giải quyết tương tự với thuật toán [Pohlig-Hellman](https://en.wikipedia.org/wiki/Pohlig%E2%80%93Hellman_algorithm) như sau:

Lần query thứ $$i$$ ta thực hiện:

- Chọn ngẫu nhiên hai số $$\left(y_{1}^{(i)}, y_{2}^{(i)}\right)$$, gửi cho `server`, nhận lại được hai điểm $$\left(G_{i}, H_{i}\right)$$ và tính được $$A_{i}, B_{i}$$.
- Tiếp theo tính order $$\lambda_{i}$$ của Curve, sau đó factorize và tìm số $$O_{i} \| \lambda_{i}$$ thỏa mãn mọi ước của $$O_{i}$$ đều bé hơn $$2^{k}$$ (có thể chọn threshold $$k=30$$).
- Cuối cùng tính DLP cho bộ điểm $$\left(\dfrac{\lambda_{i}}{O_{i}} \times G_{i}, \dfrac{\lambda_{i}}{O_{i}} \times H_{i}\right)$$, lúc này cả hai điểm đều có order $$=O_{i}$$ smooth. Tính xong thay vì ra thẳng $$m$$ thì ta sẽ tìm được $$m_{i}$$ thỏa mãn

$$
m \equiv m_{i} \pmod{O_{i}}
$$

Sau $$n$$ lần query ta có hệ:

$$
\begin{cases}
  m &\equiv m_{1} \pmod{O_{1}} \\
  m &\equiv m_{2} \pmod{O_{2}} \\
    &\vdots                    \\
  m &\equiv m_{n} \pmod{O_{n}}
\end{cases}
$$

Khi $$n$$ đủ lớn thì ta sẽ tìm được $$m$$ bằng [CRT](https://en.wikipedia.org/wiki/Chinese_remainder_theorem) 😇

```python
from sage.all import *
from Crypto.Util.number import *
from tqdm import tqdm
from factordb.factordb import FactorDB
from pwn import *

class Challenge:
  def __init__(self, local=True) -> None:
    if local:
      self.io = process(["python3", "chall.py"])
    else:
      self.io = remote("65.21.255.31", 10101)


  def get(self):
    y1, y2 = [getrandbits(32) for _ in '01']
    self.io.sendlineafter(b"curve:  \n", f"{y1},{y2}".encode())


    self.io.recvuntil(b"q = "); q = int(self.io.recvline())
    self.io.recvuntil(b"G = "); G = eval(self.io.recvline())
    self.io.recvuntil(b"G = "); H = eval(self.io.recvline())
    
    A = (y1**2 - y2**2 - 1337**3 + 31337**3) * inverse(-30000, q) % q
    B = (y1**2 - 1337**3 - A * 1337) % q
    E = EllipticCurve(GF(q), [A, B])

    return E, E(G), E(H)

def small_factor(N):
  res = 1
  for _ in range(10):
    f = FactorDB(N)
    f.connect()
    for p in f.get_factor_list():
      if p.bit_length() < 30:
        res *= p
        N //= p
  return res

if __name__ == "__main__":
  io = Challenge()

  Xs, Ms = [], []
  for _ in tqdm(range(10)):
    E, G, H = io.get()
    N = E.order()
    O = small_factor(N)

    Ms.append(O)
    Xs.append(
      ZZ(discrete_log(H*(N//O), G*(N//O), ord=O, operation="+"))
    )
  
  m = crt(Xs, Ms)
  print(long_to_bytes(int(m)))
```

> **<gg>Flag: ASIS{(e$l6LH_JfsJ:~<}1v&}</gg>**

# figole

Đề cho duy nhất file `.apk` cài đặt và chạy thử thì có vẻ giống như một bài `flag checker`

<center>
  <img src="/images/2022-asis-ctf-quals/1.png" witdh="100%">
</center>

Decompile code bằng `jADX-gui`, đầu tiên ta sẽ phân tích một số hàm chính:

- <b>Hàm `gb`</b>: Hàm này đơn giản chỉ là decode base64

```java
  public static String gb(String s) {
        byte[] data = Base64.decode(s, 0);
        return new String(data, StandardCharsets.UTF_8);
    }
```

- <b>Hàm `getInst`</b>: Hàm này đọc bytecode lưu trong file có đường dẫn là `s` sau đó convert bytecode sang class instance.

```java
    public ClassLoader getInst(String s) {
        try {
            FileInputStream fis = new FileInputStream(s);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            byte[] buffer = new byte[4096];
            while (true) {
                int bytesRead = fis.read(buffer, 0, buffer.length);
                if (bytesRead != -1) {
                    baos.write(buffer, 0, bytesRead);
                } else {
                    baos.flush();
                    byte[] dex = baos.toByteArray();
                    ByteBuffer bb = ByteBuffer.allocate(dex.length);
                    bb.put(dex);
                    bb.position(0);
                    ClassLoader loader = new InMemoryDexClassLoader(bb, (ClassLoader) null);
                    bb.clear();
                    return loader;
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
```

- <b>class `Util`</b>: Định nghĩa trước các chuỗi sẽ được dùng phía sau.

```java
public class Util {
    public static final String an = "timesnewroman.ttf";
    public static final String pn1 = "Y29tLmV4YW1wbGUuc2hjdGZkZXguVVQ=";
    public static final String pn2 = "Y29tLmV4YW1wbGUuc2hjdGZkZXguRFg=";
}
```

- <b>Hàm `l1`</b>: Ta thấy `gb(Util.pn1)` = <b>com.example.shctfdex.UT</b>, đại khái hàm nãy sẽ đọc bytescode trong file `folder/timesnewroman.ttf` với folder là nơi chứa private data của app, sau đó gọi hàm `getInst` để convert bytecode sang `ClassLoader dLoader`. Cuối cùng tạo một class instance `com.example.shctfdex.UT` bên trong `dLoader` và invoke method `gf`.

```java
    public String l1(Context context) {
        String cn = gb(Util.pn1);
        String s = context.getExternalFilesDir(null).getAbsolutePath() + "/" + Util.an;
        try {
            ClassLoader dLoader = getInst(s);
            Class<?> loadedClass = dLoader.loadClass(cn);
            Object obj = loadedClass.newInstance();
            Method m = loadedClass.getMethod(gb("Z2Y="), Context.class);
            String re = (String) m.invoke(obj, context);
            return re;
        } catch (Exception e) {
            e.printStackTrace();
            return e.getMessage();
        }
    }
```

- <b>Hàm `l2` và `l3`</b>: Tương tự `l1`, chỉ thay đội method nên ta sẽ bàn sau

- <b>Hàm `cdx`</b>: Hàm này copy file `timesnewroman.ttf` trong thư mục `assets` của app rồi bỏ vào folder data nếu file đó chưa tồn tại.

```java
    private void cdx(Context context) throws IOException {
        File file = new File(context.getExternalFilesDir(null).getAbsolutePath(), Util.an);
        if (!file.exists()) {
            InputStream mInput = context.getAssets().open(Util.an);
            OutputStream mOutput = new FileOutputStream(file);
            byte[] mBuffer = new byte[4096];
            while (true) {
                int mLength = mInput.read(mBuffer);
                if (mLength > 0) {
                    mOutput.write(mBuffer, 0, mLength);
                } else {
                    mOutput.flush();
                    mOutput.close();
                    mInput.close();
                    return;
                }
            }
        }
    }
```

- <b>Hàm `cdec`</b>: Mã hóa gì đó khá đơn giản, kệ đi!

```java
 public String cdec(String s) {
        StringBuilder htd = new StringBuilder();
        for (int i = 0; i < s.length() - 1; i += 2) {
            try {
                String output = s.substring(i, i + 2);
                int decimal = Integer.parseInt(output, 16);
                htd.append((char) decimal);
            } catch (Exception e) {
                e.printStackTrace();
                return "null";
            }
        }
        StringBuilder r = new StringBuilder();
        int keyItr = 0;
        for (int i2 = 0; i2 < htd.length(); i2++) {
            int temp = htd.charAt(i2) ^ "key".charAt(keyItr);
            r.append((char) temp);
            keyItr++;
            if (keyItr >= "key".length()) {
                keyItr = 0;
            }
        }
        return r.toString();
    }
```

- Flow chương trình:

```java
     String eText = editText.getText().toString();
        String rr = l3(this, eText, cdec(q)).trim();
        boolean b = false;
        if (y.trim().equals(rr.trim())) {
            Toast.makeText(this, "congratulations!", 1).show();
            b = true;
            button.showDoneButton();
        } else {
            Toast.makeText(this, "Ooops :)", 1).show();
            button.showErrorButton();
        }
        boolean finalB = b;
        new Timer().schedule(new AnonymousClass1(finalB, button), 2000L);
```

Để hiểu rõ `l1,l2,l3` làm gì ta cần decompile file `assets/timesnewroman.ttf`, nếu bạn dùng `jADX` thì nó đã làm sẵn cho ta luôn 😃

- <b>class `DH`:</b> Chức năng dùng để quản lý Database với các thao tác cơ bản như tạo, mở, `gd`, `gdd` lần lượt lấy ô đầu tiên của dòng thứ nhất và thứ 2.

```java
/* loaded from: assets/timesnewroman.ttf */
public class DH {
    private final DBH dbh;
    private SQLiteDatabase mDb;
    private final String mt = "calibri";

    public DH(Context context) {
        this.dbh = new DBH(context);
    }

    public void cd() {
        try {
            this.dbh.createDataBase();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void open() {
        try {
            this.mDb = this.dbh.getReadableDatabase();
        } catch (SQLException mSQLException) {
            mSQLException.printStackTrace();
        }
    }

    public void close() {
        this.dbh.close();
    }

    public String gd() {
        String r = "null";
        try {
            Cursor mCur = this.mDb.rawQuery("SELECT * FROM calibri", null);
            if (mCur == null) {
                return "null";
            }
            mCur.moveToPosition(0);
            r = mCur.getString(0);
            mCur.close();
            return r;
        } catch (SQLException mSQLException) {
            mSQLException.printStackTrace();
            return r;
        }
    }

    public String gdd() {
        String r = "null";
        try {
            Cursor mCur = this.mDb.rawQuery("SELECT * FROM calibri", null);
            if (mCur == null) {
                return "null";
            }
            mCur.moveToPosition(1);
            r = mCur.getString(0);
            mCur.close();
            return r;
        } catch (SQLException mSQLException) {
            mSQLException.printStackTrace();
            return r;
        }
    }
}
```

- <b>class `UT`:</b> Hàm `l1` và `l2` ở trên thực chất sẽ gọi tới method `gf` và `gff` của class này. Mục đích chính là trả về ô đầu tiên của hai hàng đầu trong database.

```java
public class UT {
    public static String gb(String s) {
        byte[] data = Base64.decode(s, 0);
        return new String(data, StandardCharsets.UTF_8);
    }

    public String gf(Context context) {
        DH dh = new DH(context);
        dh.cd();
        dh.open();
        String r = dh.gd();
        dh.close();
        return r;
    }

    public String gff(Context context) {
        DH dh = new DH(context);
        dh.cd();
        dh.open();
        String r = dh.gdd();
        dh.close();
        return r;
    }

    public byte[] g(String hs) {
        if (hs.length() % 2 == 1) {
            throw new IllegalArgumentException("Invalid");
        }
        byte[] bytes = new byte[hs.length() / 2];
        for (int i = 0; i < hs.length(); i += 2) {
            bytes[i / 2] = hb(hs.substring(i, i + 2));
        }
        return bytes;
    }

    public byte hb(String hs) {
        int firstDigit = td(hs.charAt(0));
        int secondDigit = td(hs.charAt(1));
        return (byte) ((firstDigit << 4) + secondDigit);
    }

    private int td(char hc) {
        int digit = Character.digit(hc, 16);
        if (digit == -1) {
            throw new IllegalArgumentException("Invalid :" + hc);
        }
        return digit;
    }
}
```

- <b>class `DX`:</b> Hàm `l3` sẽ gọi method `ech` trong class này, mục đính chính là tạo encrypt AES-CBC với `key` và `iv` hoàn toàn tự tính được.

```java
/* loaded from: assets/timesnewroman.ttf */
public class DX {
    private static byte[] getK(int t, String s) {
        byte[] total = new UT().g(s);
        byte[] c = new byte[total.length / 2];
        if (t == 0) {
            for (int i = 0; i < total.length; i++) {
                if (i % 2 == 0) {
                    c[i / 2] = total[i];
                }
            }
        } else {
            for (int i2 = 0; i2 < total.length; i2++) {
                if (i2 % 2 == 1) {
                    c[i2 / 2] = total[i2];
                }
            }
        }
        return c;
    }

    public String ech(String message, String s) {
        try {
            byte[] srcBuff = message.getBytes(StandardCharsets.UTF_8);
            SecretKeySpec keySpec = new SecretKeySpec(getK(0, s + UT.gb("Mzg2OTM3NjEzNDc0MzYzMTM1MzUzMjM2MzMzMjMxMzA=")), "AES");
            IvParameterSpec ivSpec = new IvParameterSpec(getK(1, s + UT.gb("Mzg2OTM3NjEzNDc0MzYzMTM1MzUzMjM2MzMzMjMxMzA=")));
            Cipher ecipher = Cipher.getInstance("AES/CBC/PKCS7Padding");
            ecipher.init(1, keySpec, ivSpec);
            return Base64.encodeToString(ecipher.doFinal(srcBuff), 0);
        } catch (Exception ex) {
            ex.printStackTrace();
            return "encrypt :" + ex.getMessage();
        }
    }
}
```

Tóm lại flow của bài này là:

- Đầu tiên tạo database bằng cách copy từ file `assets/calibri.ttf`.
- `y, q` lần lượt gọi các hàm `l1, l2` để lấy từ databases về mặt ý nghĩa là nó `ciphertext` và `int_key`

<center>
  <img src="/images/2022-asis-ctf-quals/sqlite.png" width="100%">
</center>

- Cuối cùng là check `input` của ta bằng cách kiểm tra: `ciphertext` == AES-CBC(`input`, `init_key`)

Để đơn giản thì mình sẽ dùng [Frida](https://frida.re/) để hook vào hai hàm `SecretKeySpec` và `IvParameterSpec` để in ra `key` và `iv`:

```javascript
function __hook_SecretKeySpec() {
  const SecretKeySpec = Java.use("javax.crypto.spec.SecretKeySpec");

  SecretKeySpec.$init.overload("[B", "java.lang.String").implementation = function(key, algo) {
    console.log(`${algo} Key:`, key);
    
    return this.$init(key, algo);
  }
}

function __hook_IvParameterSpec() {
  const IvParameterSpec = Java.use("javax.crypto.spec.IvParameterSpec");

  IvParameterSpec.$init.overload("[B").implementation = function(iv) {
    console.log("IV:", iv);

    return this.$init(iv);
  }
}

Java.perform(function() {
  __hook_SecretKeySpec();
  __hook_IvParameterSpec();
})
```

<center>
    <img src="/images/2022-asis-ctf-quals/frida.png">
</center>

Cuối cùng là viết python script để decrypt `flag`:

```python
from Crypto.Cipher import AES
from base64 import *

key = bytes([53,54,50,49,52,53,57,54,56,55,52,54,53,50,51,49])
iv  = bytes([115,101,99,114,101,116,97,114,105,97,116,49,53,54,50,48])

ct  = b64decode("7mePfqpM6Wd1El2sj4dlUboU6PieF7La8IJ1e76cfp4=")
cp  = AES.new(key, AES.MODE_CBC, iv)
print(cp.decrypt(ct))
```

> **<gg>Flag: ASIS{D3x_iZ_n0t_fOn7!}</gg>**