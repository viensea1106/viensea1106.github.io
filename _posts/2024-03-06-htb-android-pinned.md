---
title:  "[HTB - Android] Pinned"
mathjax: true
layout: post
---

Writeup an intersting android challenge using <b>SSL pinning</b> technique.



> **<span><yy>Description</yy></span>**\
> This app has stored my credentials and I can only login automatically. I tried to intercept the login request and restore my password, but this seems to be a secure connection. Can you help bypass this security restriction and intercept the password in plaintext?

Tải resource challenge về thì bên trong có 1 file `apk` và 1 file `README` với nội dung:

> 1. Install this application in an API Level 29 or earlier (i.e. Android 10.0 (Google APIs)).

...ok vậy chúng ta cần cài emulator với API level $$\le 29$$. Cài xong, tải app vô và bật lên fuzz linh tinh thử:

<img src="/images/htb-android-pinned/pinned.png" width="100%">

Bật lên thì thấy app để sẵn `username` với `password` mặc định rồi nên mình nhấn login thử thì nó chỉ trả về thông báo `You are logged in` thôi, thay đổi linh tinh rồi login thì báo sai user or pass. Decompile xem thử logic bên trong như nào:

<b>OnClickListener:</b> Khi click Login thì gọi lần lượt hai method <b>w</b> và <b>x</b> (`r` là `TextView` để hiển thị dòng chữ `You are logged in`)

```java
    /* loaded from: classes.dex */
    public class a implements View.OnClickListener {
        public a() {
        }

        @Override // android.view.View.OnClickListener
        public void onClick(View view) {
            MainActivity.this.r.setText("");
            try {
                MainActivity.this.w();
                MainActivity.this.x();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
```

<b>w:</b> Đây là chỗ setup <b>Certificate Pinning</b>, tạm thời mình sẽ phân tích sau.

```java
    public void w() {
        this.p = CertificateFactory.getInstance("X.509").generateCertificate(getResources().openRawResource(R.raw.certificate));
        KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        keyStore.load(null, null);
        keyStore.setCertificateEntry("Self signed certificate", this.p);
        TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        trustManagerFactory.init(keyStore);
        SSLContext sSLContext = SSLContext.getInstance("TLS");
        this.q = sSLContext;
        sSLContext.init(null, trustManagerFactory.getTrustManagers(), null);
    }
```

<b>x:</b> Hàm này gồm 2 nội dung chính:

- <b>Khởi tạo HttpsURLConnection:</b> setup `medthod`, `header` và quan trọng nhất là phần <b>setSSLSocketFactory</b>
- <b>Tạo dữ liệu POST:</b> Khi Login thì app sẽ tạo post request có dạng:

<center>
<yy>https://pinned.com:443/pinned.php?uname=bnavarro&pass=</yy>
</center>

trong đó giá trị của trường `pass` được tính toán phức tạp (bao gồm của phần encrypt hay decrypt gì đó ở cuối cùng). Sau khi tạo xong request sẽ được lưu lại trong `String MainActivity.o`

```java
    public void x() {
        HttpsURLConnection httpsURLConnection;
        TextView textView;
        String str;
        MainActivity mainActivity = this;
        HttpsURLConnection httpsURLConnection2 = (HttpsURLConnection) new URL("https://pinned.com:443/pinned.php").openConnection();
        httpsURLConnection2.setRequestMethod("POST");
        httpsURLConnection2.setRequestProperty("Content-Type", "application/x-www-form-urlencoded");
        httpsURLConnection2.setRequestProperty("Accept", "application/x-www-form-urlencoded");
        httpsURLConnection2.setRequestProperty("charset", "utf-8");
        httpsURLConnection2.setDoOutput(true);
        httpsURLConnection2.setSSLSocketFactory(mainActivity.q.getSocketFactory());
        if (mainActivity.s.getText().toString().equals("bnavarro") && mainActivity.t.getText().toString().equals("1234567890987654")) {
            StringBuilder g = c.a.a.a.a.g("uname=bnavarro&pass=");
            StringBuilder sb = new StringBuilder();
            sb.append(d.a());
            sb.append(c.b.a.b.a());
            sb.append(h.a());
            sb.append(c.a());
            sb.append(i.a());
            ArrayList arrayList = new ArrayList();
            arrayList.add("9GDFt6");
            arrayList.add("83h736");
            arrayList.add("kdiJ78");
            arrayList.add("vcbGT6");
            arrayList.add("LPGt63");
            arrayList.add("kFgde4");
            arrayList.add("5drDr4");
            arrayList.add("Y6ttr5");
            arrayList.add("444w45");
            arrayList.add("hjKd56");
            sb.append((String) arrayList.get(4));
            sb.append(e.a());
            sb.append(f.a());
            ArrayList arrayList2 = new ArrayList();
            arrayList2.add("TG7ygj");
            arrayList2.add("U8uu8i");
            arrayList2.add("gGtT56");
            arrayList2.add("84hYDG");
            arrayList2.add("yRCYDm");
            arrayList2.add("7ytr4E");
            arrayList2.add("j5jU87");
            arrayList2.add("yRCYDm");
            arrayList2.add("jd9Idu");
            arrayList2.add("kd546G");
            sb.append((String) arrayList2.get(7));
            sb.append(c.b.a.a.a());
            String sb2 = sb.toString();
            char charAt = b.q.h.b().charAt(3);
            char charAt2 = b.q.h.b().charAt(0);
            char charAt3 = c.a().charAt(0);
            char charAt4 = c.b.a.a.a().charAt(8);
            char charAt5 = h.a().charAt(1);
            char charAt6 = b.q.h.b().charAt(0);
            char charAt7 = i.a().charAt(5);
            char charAt8 = c.b.a.a.a().charAt(7);
            char charAt9 = c.b.a.b.a().charAt(4);
            httpsURLConnection = httpsURLConnection2;
            char charAt10 = e.a().charAt(4);
            char charAt11 = e.a().charAt(4);
            char charAt12 = i.a().charAt(5);
            char charAt13 = d.a().charAt(3);
            char charAt14 = d.a().charAt(5);
            char charAt15 = h.a().charAt(1);
            char charAt16 = h.a().charAt(1);
            SecretKeySpec secretKeySpec = new SecretKeySpec((String.valueOf(charAt) + String.valueOf(charAt2) + String.valueOf(charAt3) + String.valueOf(charAt4).toUpperCase() + String.valueOf(charAt5) + String.valueOf(charAt6) + String.valueOf(charAt7).toUpperCase() + String.valueOf(charAt8) + String.valueOf(charAt9) + String.valueOf(charAt10) + String.valueOf(charAt11) + String.valueOf(charAt12).toUpperCase() + String.valueOf(charAt13) + String.valueOf(charAt14) + String.valueOf(charAt15) + String.valueOf(charAt16)).getBytes(), g.a());
            Cipher cipher = Cipher.getInstance(g.a());
            cipher.init(2, secretKeySpec);
            g.append(new String(cipher.doFinal(Base64.decode(sb2, 0)), "utf-8"));
            mainActivity = this;
            mainActivity.o = g.toString();
            textView = mainActivity.r;
            str = "You are logged in.";
        } else {
            httpsURLConnection = httpsURLConnection2;
            StringBuilder g2 = c.a.a.a.a.g("uname=");
            g2.append(mainActivity.s.getText().toString());
            g2.append("&pass=");
            g2.append(mainActivity.t.getText().toString());
            mainActivity.o = g2.toString();
            textView = mainActivity.r;
            str = "Wrong credentials!";
        }
        textView.setText(str);
        DataOutputStream dataOutputStream = new DataOutputStream(httpsURLConnection.getOutputStream());
        try {
            dataOutputStream.writeBytes(mainActivity.o);
            dataOutputStream.flush();
            dataOutputStream.close();
            new Thread(new b(httpsURLConnection)).start();
        } finally {
        }
    }
```

Tóm lại flow của bài này như sau:

- Khi click Login thì client sẽ tạo request gửi cho server: <yy>https://pinned.com:443/pinned.php?uname=bnavarro&pass=</yy>
- Đầu tiên check xem `username` với `password` ta điền có giống với `bnavarro` và `1234567890987654` hay không, nếu có thì tiếp tục tính toán trường `pass` để hoàn thành và gửi request đi.
- Trường `pass` được tính toán khá phức tạp và lưu vào biến `String MainActivity.o`.

$$\Rightarrow$$ Code chỉ có nhiêu đó nên mình đoán `flag` cần tìm nằm ở trường `pass` trong request. Để giải quyết thì có hai hướng:

- <b>Intercept Traffic</b> giữa client và server từ đó đọc được toàn bộ nội dung của request login.
- <b>Hooking</b> và lấy chuỗi post request lưu trong `String MainActivity.o`

Ban đầu mình chọn cách hooking vì lý do BẮT TRAFFIC BẰNG BURPSUITE KHÔNG ĐƯỢC 😀, cách làm dùng frida khá đơn giản bằng script sau:

```javascript
Java.performNow(function() {
  Java.choose("com.example.pinned.MainActivity", {
    onMatch: function(ins) {
      console.log(ins.o.value);
    },
    onComplete: function(){}
  })
})
```

Lúc sau mình tìm hiểu lý do sao burpsuite mình setup bắt traffic từ web-browser bên trong điện thoại ngon lành mà sao bắt traffic từ app lại không được... và cuối cùng mình đã biết tới kỹ thuật <b>SSL pinning</b> (tên challenge gợi ý luôn đó 😅)

SSL Pinning là cơ chế phòng ngừa Man-in-the-middle attack bằng cách lưu trước certificate của server bên trong app. Mỗi lần thực hiện kết nối tới server, phía client sẽ check xem cert được gửi về có trùng khớp với cert được lưu (pinned) trước đó hay không, nếu không sẽ lập tức hủy kết nối (phát hiện bên thứ 3 giả mạo server).

Đọc lại code ở hàm <b>w</b> ta thấy cert được lấy ra từ file <b>res/raw/certificate.cer</b>. Vậy để bắt được traffic bằng burpsuite thì ta sẽ nghĩ cách thay thế cert được lưu trong này bằng cert của burpsuite, cơ bản có 2 cách:

- Patch lại apk, thay đổi cert được lưu sẵn.
- Hooking: tạo `TrustManagerFactory` "giả" (có chưa `burp.cer`) và hook vào constructor của `class SSLContext`, modify cho nó luôn trust cert của chúng ta.

Ở đây mình sẽ dùng phương pháp hooking (sẵn luyện viết script Frida luôn 😀), ta chỉ việc implement tương tự các bước như trong hàm <b>w</b>, khác biệt duy nhất ở chỗ thay vì lấy cert trong <b>res</b>, ta sẽ load nó trực tiếp (từ file chúng ta đã push vô device, vd: <b>data/local/tmp/burp.der</b>):

```javascript
Java.perform(function() {
  try {
    // Step 1: Push burp.der to /data/local/tmp then load it
    console.log("[+] Loading our CERT")
    const FileInputStream = Java.use("java.io.FileInputStream");
    const BufferedInputStream = Java.use("java.io.BufferedInputStream");
    const CertificateFactory = Java.use("java.security.cert.CertificateFactory");
    const X509Certificate = Java.use("java.security.cert.X509Certificate");
    
    var cert_file = FileInputStream.$new("/data/local/tmp/burp.der");
    var cert_stream = BufferedInputStream.$new(cert_file);
    var cert_factory = CertificateFactory.getInstance("X.509").generateCertificate(cert_stream);
    cert_stream.close();
  
    var certInfo = Java.cast(cert_factory, X509Certificate);
    console.log("[o] Our CA Info: " + certInfo.getSubjectDN());

    // Step 2: Create a KeyStore containing our burp.der
    const KeyStore = Java.use("java.security.KeyStore");

    var ks = KeyStore.getInstance(KeyStore.getDefaultType());
    ks.load(null, null);
    ks.setCertificateEntry("Self signed certificate", cert_factory);

    // Step 3: Create a TrustManager that trusts burp.der in our KeyStore
    const TrustManagerFactory = Java.use("javax.net.ssl.TrustManagerFactory");
    
    var tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
    tmf.init(ks);

    // Step 4: Hooking SSLContext to force it using our TrustManagerFactory
    const SSLContext = Java.use("javax.net.ssl.SSLContext");

    SSLContext.init.overload("[Ljavax.net.ssl.KeyManager;", "[Ljavax.net.ssl.TrustManager;", "java.security.SecureRandom").implementation = function(a, b, c) {
      console.log("[+] Hooked javax.net.ssl.SSLContext.init to bypass SSL pinning");
      SSLContext.init.overload("[Ljavax.net.ssl.KeyManager;", "[Ljavax.net.ssl.TrustManager;", "java.security.SecureRandom").call(this, a, tmf.getTrustManagers(), c);
    }

  } catch (error) {
    console.log("[!] " + error);    
  }
})
```

<img src="/images/htb-android-pinned/hooked.png">

<img src="/images/htb-android-pinned/burp.png">

> **<gg>Flag: HTB{trust_n0_1_n0t_3v3n_@_c3rt!}</gg>**