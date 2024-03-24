---
title:  "picoCTF - Android Challenges"
mathjax: true
layout: post
---

Writeup of some Android Challenges in [picoCTF Wargames](https://picoctf.org/)

<img src="/images/pico-android/pico.png" width="100%">





# droids0

Đề bài cung cấp duy nhất file `zero.apk`, ta sẽ cài đặt file và tiến hành kiểm thử. Chúng ta có thể cài đặt trực tiếp trên device thật (điện thoại di động), tất nhiên ít ai lại làm thế mà thông thường sẽ cài trên các thiết bị giả lập (emulator):

- [Genymotion](https://www.genymotion.com/)
- [Android Studio](https://developer.android.com/studio)

Genymotion được ưa chuộng hơn vì nó nhẹ, dễ cài đặt và cấu hình giả lập được hầu hết các thiết bị. Còn Android Studio ngoài việc cung cấp tính năng giả lập, nó còn hỗ trợ các công cụ dev cho lập trình android nên sẽ nặng hơn nhiều, tuy nhiên vẫn có một số tiện ích hay ho nên máy bạn nào khỏe thì cứ cài thêm 😂

Tiến hành cài đặt bằng cách kéo thả file APK vào trong máy giả lập hoặc có thể sử dụng công cụ command-line [ADB](https://developer.android.com/tools/adb)

<img src="/images/pico-android/0adb.png" width="100%">

Bật app lên, nhập thử vài lần nhưng không thấy phản hồi gì hết...

<center>
  <img src="/images/pico-android/0installed.png">
</center>

Tiến hành decompile file APK để hiểu logic chương trình, ở đây mình dùng tool [jADX-gui](https://github.com/skylot/jadx)

```java
/* loaded from: classes.dex */
public class MainActivity extends AppCompatActivity {
    Button button;
    Context ctx;
    TextView text_bottom;
    EditText text_input;
    TextView text_top;

    /* JADX INFO: Access modifiers changed from: protected */
    @Override // androidx.appcompat.app.AppCompatActivity, androidx.fragment.app.FragmentActivity, androidx.core.app.ComponentActivity, android.app.Activity
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        this.text_top = (TextView) findViewById(R.id.text_top);
        this.text_bottom = (TextView) findViewById(R.id.text_bottom);
        this.text_input = (EditText) findViewById(R.id.text_input);
        this.ctx = getApplicationContext();
        System.loadLibrary("hellojni");
        this.text_top.setText(R.string.hint);
    }

    public void buttonClick(View view) {
        String content = this.text_input.getText().toString();
        this.text_bottom.setText(FlagstaffHill.getFlag(content, this.ctx));
    }
}
```

Khi ta nhập thì input sẽ được lưu vào biến `content`, khi click vào button sẽ gọi phương thức `getFlag(content, this.ctx)` của class `FlagstaffHill`, giá trị trả về dạng chuỗi và được gán cho `TextView text_bottom`. Tiến hành đọc code `class FlagstaffHill`

```java
/* loaded from: classes.dex */
public class FlagstaffHill {
    public static native String paprika(String str);

    public static String getFlag(String input, Context ctx) {
        Log.i("PICO", paprika(input));
        return "Not Today...";
    }
}
```

Hàm `String getFlag` nhận input của ta nhưng không xử lý gì hết mà luôn trả về chuỗi <yy>Not Today...</yy> Tuy nhiên input của ta lại được xử lý trong hàm `public static native String paprika(String str)` được load từ thư viện native `System.loadLibrary("hellojni")`, kết quả sau khi xử lý không hiển thị cho người dùng mà được đẩy vào log hệ thống bằng cách gọi hàm `Log.i(TAG, MSG)`. 

Để xem log hệ thống của thiết bị Android ta có thể sử dụng công cụ [Logcat](https://developer.android.com/tools/logcat) (có sẵn trong ADB). Lưu ý log được gắn tag `PICO` nên mình sẽ filter cho dễ nhìn, chứ không nó sẽ ra một đống log luôn 😅

<center>
  <img src="/images/pico-android/0logcat.png">
</center>

Ohhh có `flag` luôn nè, như vậy là nhập gì nó cũng nhả `flag` chứ không liên quan gì tới input 😁

> **<gg>picoCTF{a.moose.once.bit.my.sister}</gg>**

# droids1

Bài này về cơ bản vẫn giống lần trước, điểm khác biệt năm ở hàm `getFlag`, sẽ check input khi ta click vào button

```java
/* loaded from: classes.dex */
public class FlagstaffHill {
    public static native String fenugreek(String str);

    public static String getFlag(String input, Context ctx) {
        String password = ctx.getString(R.string.password);
        return input.equals(password) ? fenugreek(input) : "NOPE";
    }
}
```

Chuối ta nhập sẽ được so với biến `password` được lấy ra bằng cách gọi hàm `ctx.getString`. Bản chất APK là một file zip, bên trong lưu mã code (compiled), resources (lưu fonts/images/icons/defined-colors,strings/...) Khi gọi hàm `ctx.getString(R.string.password)` sẽ lấy chuỗi tên là `password` trong file resource `res/values/strings.xml`. Ta có thể check nó bằng JADX

<center>
  <img src="/images/pico-android/1strings.png">
</center>

Vậy password cần nhập là <yy>opossum</yy>, vào app nhập nó và lấy `flag` thôi 😄

<center>
  <img src="/images/pico-android/1done.png">
</center>

> **<gg>picoCTF{pining.for.the.fjords}</gg>**

# droids2

```java
/* loaded from: classes.dex */
public class FlagstaffHill {
    public static native String sesame(String str);

    public static String getFlag(String input, Context ctx) {
        String[] witches = {"weatherwax", "ogg", "garlick", "nitt", "aching", "dismass"};
        int second = 3 - 3;
        int third = (3 / 3) + second;
        int fourth = (third + third) - second;
        int fifth = 3 + fourth;
        int sixth = (fifth + second) - third;
        String password = "".concat(witches[fifth]).concat(".").concat(witches[third]).concat(".").concat(witches[second]).concat(".").concat(witches[sixth]).concat(".").concat(witches[3]).concat(".").concat(witches[fourth]);
        return input.equals(password) ? sesame(input) : "NOPE";
    }
}
```

Lần này `password` không lưu trực tiếp trong resources nữa mà là được tạo thành thông qua vài bước tính toán. Do đó ta chỉ việc copy lại đoạn code java và chạy nó để in ra `password` sau đó nhập vô app để lấy `flag`

```java
import java.io.*;

public class Solve {
  public static void main(String[] args) {
    String[] witches = {"weatherwax", "ogg", "garlick", "nitt", "aching", "dismass"};
    int second = 3 - 3;
    int third = (3 / 3) + second;
    int fourth = (third + third) - second;
    int fifth = 3 + fourth;
    int sixth = (fifth + second) - third;
    String password = "".concat(witches[fifth]).concat(".").concat(witches[third]).concat(".").concat(witches[second]).concat(".").concat(witches[sixth]).concat(".").concat(witches[3]).concat(".").concat(witches[fourth]);
    
    System.out.println("password: " + password);
    // password: dismass.ogg.weatherwax.aching.nitt.garlick
  }
}
```

<center>
  <img src="/images/pico-android/2done.png">
</center>

> **<gg>picoCTF{what.is.your.favourite.colour}</gg>**

# droids3

```java
/* loaded from: classes.dex */
public class FlagstaffHill {
    public static native String cilantro(String str);

    public static String nope(String input) {
        return "don't wanna";
    }

    public static String yep(String input) {
        return cilantro(input);
    }

    public static String getFlag(String input, Context ctx) {
        String flag = nope(input);
        return flag;
    }
}
```

Hmm... lần này hàm `getFlag` sẽ trả về kết quả `nope(input)` (luôn return chuỗi <yy>don't wanna</yy>) chứ không phải gọi hàm `yeb(input)` (hàm sẽ trả về `flag` cho mình). Vậy để đọc được `flag` chúng ta cần thay đổi luồng thực khi của chương trình, theo mình tìm hiểu thì có hai cách chính:

- Static: Patching APK.
- Dynamic: Hooking/Modifying Instruction at Runtime.

Ở đây mình sẽ sử dụng kỹ thuật patching, ý tưởng là sửa lại bytecode thay vì gọi `nope` sẽ gọi `yeb`. Các bước cơ bản như sau: 

- Dùng tool [Apktool](https://apktool.org/) để disassemble (`.dex`) ra assembly code (`.smali`).

```bash
$ apktool d three.apk
I: Using Apktool 2.9.3 on three.apk
I: Loading resource table...
I: Decoding file-resources...
I: Loading resource table from file: C:\Users\viens\AppData\Local\apktool\framework\1.apk
I: Decoding values */* XMLs...
I: Decoding AndroidManifest.xml with resources...
I: Regular manifest package...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
Press any key to continue . . .

$ ls -la
drwxrwxrwx    - vnc1106  2 Mar 23:23 three
.rw-r--r-- 1.8M vnc1106 27 Oct  2020 three.apk
```

- Đọc assembly và chỉnh sửa (patching) logic trong các file `.smali`.

```java
[...] // smali/com/hellocmu/picoctf/FlagstaffHill.smali
.method public static getFlag(Ljava/lang/String;Landroid/content/Context;)Ljava/lang/String;
    .locals 1
    .param p0, "input"    # Ljava/lang/String;
    .param p1, "ctx"    # Landroid/content/Context;

    .line 19
    invoke-static {p0}, Lcom/hellocmu/picoctf/FlagstaffHill;->nope(Ljava/lang/String;)Ljava/lang/String;

    move-result-object v0

    .line 20
    .local v0, "flag":Ljava/lang/String;
    return-object v0
.end method
[...]
```
$$\Rightarrow$$ Ta thấy vị trí chính xác câu lệnh gọi hàm `nope`. Vì hai hàm `nope` và `yeb` cấu trúc "tương tự" nhau (đều nhận 1 tham số kiểu `string` và giá trị trả về cũng là kiểu `string`) nên ta chỉ việc sửa `nope` thành `yeb` là tự động chương trình sẽ nhảy tới `yeb` và lấy `flag` cho mình 😅

- Dùng tool [Apktool](https://apktool.org/) để assemble (`.smali` -> `.dex`) và build lại thành file `.apk`

```bash
$ apktool b ./three -o ./three_patched.apk
I: Using Apktool 2.9.3
I: Checking whether sources has changed...
I: Smaling smali folder into classes.dex...
I: Checking whether resources has changed...
I: Building resources...
I: Copying libs... (/lib)
I: Building apk file...
I: Copying unknown files/dir...
I: Built apk into: three_patched
Press any key to continue . . .

$ ls -la
.rwxrwxrwx 1.5M vnc1106  2 Mar 23:35 three_patched
drwxrwxrwx    - vnc1106  2 Mar 23:35 three
.rw-r--r-- 1.8M vnc1106 27 Oct  2020 three.apk
```

- Lúc này ta đã thay đổi nội dung trong file `.apk` nên cần ký lại file (vì khi cài đặt tệp `.apk`, Android sẽ có cơ chế kiểm tra app có toàn vẹn bằng các check các chữ ký số lưu trong folder `META-INF`).

```bash
# generate keypair using `keytool`
$ keytool -genkeypair -v -keystore ctf.jks -keyalg RSA -keysize 2048 -validity 10000 -alias ctf

# zip aligment using `zipalign`
$ zipalign.exe -p -f -v 4 .\three_patched.apk .\patched.apk

# resign apk using `apksigner`
$ apksigner sign --ks .\ctf.jks .\patched.apk

# you can found `keytool`, `apksigner`, `zipalign` by installing Android Studio SDK
```

- Cuối cùng cài lại file `patched.apk` và kiểm tra

<center>
  <img src="/images/pico-android/3done.png">
</center>

> **<gg>Flag: picoCTF{tis.but.a.scratch}</gg>**

# droids4

```java
/* loaded from: classes.dex */
public class FlagstaffHill {
    public static native String cardamom(String str);

    public static String getFlag(String input, Context ctx) {
        StringBuilder ace = new StringBuilder("aaa");
        StringBuilder jack = new StringBuilder("aaa");
        StringBuilder queen = new StringBuilder("aaa");
        StringBuilder king = new StringBuilder("aaa");
        ace.setCharAt(0, (char) (ace.charAt(0) + 4));
        ace.setCharAt(1, (char) (ace.charAt(1) + 19));
        ace.setCharAt(2, (char) (ace.charAt(2) + 18));
        jack.setCharAt(0, (char) (jack.charAt(0) + 7));
        jack.setCharAt(1, (char) (jack.charAt(1) + 0));
        jack.setCharAt(2, (char) (jack.charAt(2) + 1));
        queen.setCharAt(0, (char) (queen.charAt(0) + 0));
        queen.setCharAt(1, (char) (queen.charAt(1) + 11));
        queen.setCharAt(2, (char) (queen.charAt(2) + 15));
        king.setCharAt(0, (char) (king.charAt(0) + 14));
        king.setCharAt(1, (char) (king.charAt(1) + 20));
        king.setCharAt(2, (char) (king.charAt(2) + 15));
        String password = "".concat(queen.toString()).concat(jack.toString()).concat(ace.toString()).concat(king.toString());
        return input.equals(password) ? "call it" : "NOPE";
    }
}
```

Đọc sơ qua thì ta đoán `flag` được load từ thư viện native `cardamom(String str)`, nhiệm vụ của ta là viết smali code gọi hàm này với tham số truyền vào là `password`. Disassembly và đọc smali code:

```java
.method public static getFlag(Ljava/lang/String;Landroid/content/Context;)Ljava/lang/String;
[...]
    .line 36
    .local v4, "password":Ljava/lang/String;
    invoke-virtual {p0, v4}, Ljava/lang/String;->equals(Ljava/lang/Object;)Z

    move-result v5

    if-eqz v5, :cond_0

    const-string v5, "call it"

    return-object v5

    .line 37
    :cond_0
    const-string v5, "NOPE"

    return-object v5
.end method
```

ở đây ta cần sửa lại code chỗ <yy>call it</yy>, về cú pháp gọi hàm, truyền tham số và lấy giá trị trả về khá đơn giản như sau:

```java
[...]
    if-eqz v5, :cond_0

    invoke-static {v4}, Lcom/hellocmu/picoctf/FlagstaffHill;->cardamom(Ljava/lang/String;)Ljava/lang/String;

    move-result-object v5

    return-object v5
[...]
```

`v4` là chuỗi `password` sau khi tính toán, `com/hellocmu/picoctf/FlagstaffHill;->cardamom` để gọi method của class cụ thể, `move-result-object v5` để lấy giá trị trả về (lưu ý giá trị trả về là `string` nên phải thêm keyword `object`)

Patch xong rồi thì build lại và chạy thử coi có crash không 😅

<center>
  <img src="/images/pico-android/4done.png">
</center>

ngon lành luôn 😉. À còn `password` thì chỉ việc code lại đoạn java và chạy là có ngay (`alphabetsoup`)

> **<gg>Flag: picoCTF{not.particularly.silly}</gg>**

# Tổng kết

Như vậy ta đã hoàn thành 5 challenge Android của picoCTF. Qua 5 bài trên ta đã làm quen với các kỹ thuật/công cụ cơ bản khi làm việc với Android như: `adb`, `adb shell`, `logcat`, `apktool`, `jadx-gui`, `disassemble`, `patching smali`, `re-assemble`, `rebuild-apk`, `resign-apk`.

Theo đánh giá của mình của 5 bài đều ở mức độ dễ và phù hợp với những ai mới bắt đầu vọc Android (như mình 😉)