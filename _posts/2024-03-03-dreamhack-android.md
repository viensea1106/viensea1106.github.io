---
title:  "Dreamhack - Android Challenges"
mathjax: true
layout: post
---

Writeup of some Android Challenges in [Dreamhack Wargames](https://dreamhack.io/)

<img src="/images/dreamhack-android/dreamhack.png" width="100%">



# \[CodeEngn\] MobileApp L01

Đề cho ta duy nhất một file `apk`, tiến hành cài đặt thử thì thấy lỗi:

<img src="/images/dreamhack-android/0fail_install.png" width="100%">

Loay hoay mãi không biết làm gì hết, mình thử extract file `apk` ra để kiểm tra thử:

<img src="/images/dreamhack-android/0zipinfo.png" width="100%">

Về bản chất `apk` là một file `zip` chứa tất cả các tài nguyên và mã code cần thiết để cài đặt và chạy một ứng dụng trên Android. Dưới đây là cấu trúc cơ bản của một file `apk`:

- <b>AndroidManifest.xml</b>: Đây là tệp tin `xml` chứa thông tin cơ bản về ứng dụng như `package name`, `permissions`, `activities`,...
- <b>classes.dex</b>: Đây là các tệp tin chứa bytecode (Dalvik Executable) được biên dịch từ mã nguồn Java của ứng dụng.
- <b>resources.arsc</b>: Đây là tệp tin chứa tài nguyên đã được biên dịch của ứng dụng như `strings`, `images`, `layout`,...
- <b>lib</b>: Thư mục này chứa các thư viện native (thường là các file shared-object `.so`), mỗi thư viện trong một thư mục con tương ứng với kiến trúc của thiết bị chạy Android. Ví dụ: `armeabi`, `armeabi-v7a`, `x86`, `x86_64`,...
- <b>assets</b>: Thư mục này chứa các tài nguyên không được biên dịch như tệp tin âm thanh, video,...
- <b>META-INF</b>: Thư mục này chứa các tệp tin metadata như `CERT.RSA`, `CERT.SF`, `MANIFEST.MF`. Các tệp tin này được sử dụng cho mục đích xác thực và kiểm tra chữ ký của ứng dụng.

Check lại output sau khi chạy lệnh `zipinfo` ta thấy như sau:
- Các files `AndroidManlfests.xml`, `class.dex`, `resource.arsc` đặt tên sai với cấu trúc chuẩn nên sẽ không thể cài đặt được.
- Trong folder `lib` chỉ có duy nhất folder `/armeabi` $$\Rightarrow$$ chỉ hỗ trợ các thiết bị kiến trúc `ARM`.

Vậy nguyên nhân cài không được là do cấu trúc file `apk` không đúng (sai tên) và do thư viện native không hỗ trọ kiến trúc của emulator/device bạn đang dùng. Cách giải quyết là:
- Sử lại tên $$\Rightarrow$$ zip và ký lại file `apk` mới.
- Cài đặt emulator hoặc dùng device thật có hỗ trợ `ARM`. 

Ở đây máy host mình đang là `x86` nên muốn emulate `ARM` thì phải cài `ARM translator`, sau một hồi tìm tòi thì mình tìm được:

- [Repo github](https://github.com/m9rco/Genymotion_ARM_Translation) `ARM translator`.
- [Link](https://docs.genymotion.com/desktop/041_Deploying_an_app/) hướng dẫn cài `ARM translator` trên `Genymotion x86 emulator`.

Làm theo hướng dẫn là được 😇 Ok sau khi mọi việc xong xuôi, cài đặt được gói và ta có ngay `flag`

<img src="/images/dreamhack-android/0done.png" width="100%">

> **<gg>Flag: H3ll0 C0de3ngn</gg>**


# \[CodeEngn\] MobileApp L02

Vẫn những thao tác cũ, check xem trong `apk` có những gì bằng lệnh `zipinfo`:

<img src="/images/dreamhack-android/1zipinfo.png" width="100%">

Ok...lần này tên file đúng còn kiến trúc thì vẫn là `ARM`, cài thử xem sao

<img src="/images/dreamhack-android/1error.png" width="100%">

Lỗi nữa hả 😅 đọc sơ qua thì có vẻ file chưa được ký nên không cài được, giờ thì ký lại và reinstall thử xem...

<img src="/images/dreamhack-android/1resign.png" width="100%">

OK được rồi, nhưng không có gì thú vị cả, decompile rồi reverse thôi...

```java
/* loaded from: classes.dex */
public class MainActivity extends Activity {
    TextView aView;

    @Override // android.app.Activity
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        this.aView = (TextView) findViewById(R.id.textView1);
        if (makeDate() == "2013-11-02-12:35:03" && Volume() == 53) {
            this.aView.setText(keyString());
        }
    }

    @Override // android.app.Activity
    public boolean onCreateOptionsMenu(Menu menu) {
        getMenuInflater().inflate(R.menu.main, menu);
        return true;
    }

    public int Volume() {
        return 33;
    }

    public String makeDate() {
        Date date = new Date();
        String tmpFormat = new SimpleDateFormat("yyyy-MM-dd-hh:mm:ss").format(date);
        return tmpFormat;
    }

    public String keyString() {
        String plainKey = Security.DecryptStr("-1c776f3a2fa678cae27879e87a74553c61009eefb037bbe2abdbd5e4314407e60000000000000000");
        return plainKey;
    }
}
```

Ta thấy điều kiện để hiển thị `key` là `if (makeDate() == "2013-11-02-12:35:03" && Volume() == 53)` trong đó:

- Hàm `public String makeDate()` trả về thời gian hiện tại.
- Hàm `public int Volume()` luôn trả về giá trị 33.

Tới đầy mình thấy 3 hướng giải quyết sau:

- <b>Cách 1:</b> Reverse trực tiếp hàm `Security.DecryptStr` để tính `key`, tuy nhiên cách này hơi chuối vì phải reverse lại thư viện native.
- <b>Cách 2:</b> Hooking và gọi thẳng method `public String keyString()` để in `key` cách này nhìn chung cũng không quá phức tạp nhưng mình sẽ đề dành nói về kỹ thuật Hooking trong các bài viết sau 😂
- <b>Cách 3:</b> Patching lại binary (mình sẽ làm cách này).

Ta thấy ngay hai hàm cần patch lại để bypass key check là:

- Hàm `public int Volume()` cho nó return về giá trị 53:

```java
// smali/com/namdaehyeon/findkey2/MainActivity.smali
[...]
.method public Volume()I
    .locals 1

    .prologue
    .line 44
    const/16 v0, 0x35

    return v0
.end method
[...]
```

- Hàm `public String makeDate()` cho nó return về chuỗi `2013-11-02-12:35:03`:

```java
// smali/com/namdaehyeon/findkey2/MainActivity.smali
[...]
.method public makeDate()Ljava/lang/String;
    [...]
    .line 50
    const-string v1, "2013-11-02-12:35:03"
    return-object v1
.end method
[...]
```

Patch xong build lại và ký lại bằng `jarsigner`, sau khi cài xong thì vô app ta có ngay `flag` 😃

<img src="/images/dreamhack-android/1done.png" width="100%">

> **<gg>Flag: November Rain</gg>**

# \[CodeEngn\] MobileApp L03

Câu này cũng dính lỗi y chang bài 2, ta phải ký lại file `apk` mới cài đặt được

<img src="/images/dreamhack-android/2installed.png" width="100%">

Tiếng Hàn không hiểu gì hết 😅 decompile rồi đọc code thử xem:

```java
[...]
    public int randomRange(int a1) {
        int random = ((int) (Math.random() * 10000.0d)) * (a1 << 2);
        return random;
    }

    public String myString() {
        return "c0de3ngn.com";
    }

    public void addListenerOnButton() {
        this.button = (ImageButton) findViewById(R.id.imageButton1);
        this.aView = (TextView) findViewById(R.id.textView4);
        this.bView = (TextView) findViewById(R.id.textView5);
        this.cView = (TextView) findViewById(R.id.textView6);
        this.aView.setText(String.format("%d", Integer.valueOf(randomRange(44444))));
        this.bView.setText("0");
        this.cView.setText(myString());
        this.button.setOnClickListener(new View.OnClickListener() { // from class: com.namdaehyeon.findkey3.MainActivity.1
            Integer myStairs;
            Integer stairs;

            {
                this.stairs = Integer.valueOf(Integer.parseInt(new StringBuilder().append((Object) MainActivity.this.aView.getText()).toString()));
                this.myStairs = Integer.valueOf(Integer.parseInt(new StringBuilder().append((Object) MainActivity.this.bView.getText()).toString()));
            }

            @Override // android.view.View.OnClickListener
            public void onClick(View arg0) {
                if (this.stairs.intValue() - 1 != this.myStairs.intValue()) {
                    this.myStairs = Integer.valueOf(this.myStairs.intValue() + 1);
                    MainActivity.this.bView.setText(this.myStairs.toString());
                    return;
                }
                MainActivity.this.aView.setText(Security.DecryptStr("2736f6055dbad2d42f6d5b0135395cb29e0d086b67e1fa266a0a0d277f151e5b00000000000000000000"));
                MainActivity.this.bView.setText("0");
            }
        });
    }
[...]
```

Logic bài này đại khái như sau:

- Biến `this.stairs` lưu số lần bấm click với giá trị khởi tạo là 0, biến `this.myStairs` được gán sẵn với giá trị `randomRange(44444)`
- Nhiệm vụ của ta là click đủ số lần để hai biến `this.stairs` và `this.myStairs` bằng nhau, khi đó `flag` sẽ được hiển thị ở `TextView aView`
- Một điều khá thú vị là ta không nhìn thấy số nào được hiển thị ở `TextView aView` mặc dù trước đó đã gọi lệnh `this.aView.setText(String.format("%d", Integer.valueOf(randomRange(44444))))` Nên mình đoán có lẽ là layout nó có vấn đề, hoặc đây là intended của tác giả để `flag` không thể hiển thị được trên `aView`.

Vậy hướng làm đơn giản vẫn là patching như sau:

- Patch lại hàm `randomRange` cho nó luôn return về giá trị 1. Khi đó chỉ việc click 1 lần là xong!

```java
// smali/com/namdaehyeon/findkey3/MainActivity.smali
[...]
.method public randomRange(I)I
    [...]
    .line 40
    const/4 v0, 0x1
    return v0
.end method
[...]
```

- Patch lại để `flag` hiển thị trên `bView`:

```java
// smali/com/namdaehyeon/findkey3/MainActivity$1.smali
[...]
    .line 78
    :goto_0
    return-void

    .line 75
    :cond_0
    iget-object v0, p0, Lcom/namdaehyeon/findkey3/MainActivity$1;->this$0:Lcom/namdaehyeon/findkey3/MainActivity;

    iget-object v0, v0, Lcom/namdaehyeon/findkey3/MainActivity;->bView:Landroid/widget/TextView;

    const-string v1, "2736f6055dbad2d42f6d5b0135395cb29e0d086b67e1fa266a0a0d277f151e5b00000000000000000000"

    invoke-static {v1}, Lcom/namdaehyeon/findkey3/Security;->DecryptStr(Ljava/lang/String;)Ljava/lang/String;

    move-result-object v1

    invoke-virtual {v0, v1}, Landroid/widget/TextView;->setText(Ljava/lang/CharSequence;)V

    goto :goto_0
.end method
```

Build lại rồi mở app lên click một cái là ra `flag` luôn


<img src="/images/dreamhack-android/3done.png" width="100%">

> **<gg>Flag: CodeEngn_4_Ever</gg>**
