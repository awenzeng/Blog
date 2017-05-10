---
layout: post
title: "Android常用加解密算法"
date: 5/10/2017 8:54:22 PM 
comments: true
tags: 
	- 技术 
	- Android
	- 加解密算法
---
---
数据安全，不管是对于企业还是个人都是十分重要。而作为一个移动开发者，我们更需要去考虑数据传输的安全性，去保护企业及个人信息安全。在Java,Android语言中，有许多的常用加解密算法，例如：对称加密算法AES,DES,3DES，非对称加密算法RSA,经典哈希算法MD5,SHA.

>对称加密算法：加密秘钥和解密秘钥相同 例：AES,DES,3DES
>
>非对称加密算法：有公钥和秘钥，公钥加密私钥解密，私钥加密公钥解密。例：RSA
>
>哈希算法：加解密是不可逆的 例：MD5,SHA



# 常用加解密算法
---

# 1.AES算法(Advanced Encryption Standard 高级数据加密标准)

- AES：高级数据加密标准，能够有效抵御已知的针对DES算法的所有攻击 

- 特点：密钥建立时间短、灵敏性好、内存需求低、安全性高 

- Java实现

1).生成秘钥
```java
KeyGenerator keyGen = KeyGenerator.getInstance("AES");//密钥生成器
keygen.init(128);  //默认128，获得无政策权限后可为192或256
SecretKey secretKey = keyGen.generateKey();//生成密钥
byte[] key = secretKey.getEncoded();//密钥字节数组
```
<!-- more -->
2).AES加密
```java
SecretKey secretKey = new SecretKeySpec(key, "AES");//恢复密钥
Cipher cipher = Cipher.getInstance("AES");//Cipher完成加密或解密工作类
cipher.init(Cipher.ENCRYPT_MODE, secretKey);//对Cipher初始化，解密模式
byte[] cipherByte = cipher.doFinal(data);//加密data
```
3).AES解密
```java
SecretKey secretKey = new SecretKeySpec(key, "AES");//恢复密钥
Cipher cipher = Cipher.getInstance("AES");//Cipher完成加密或解密工作类
cipher.init(Cipher.DECRYPT_MODE, secretKey);//对Cipher初始化，解密模式
byte[] cipherByte = cipher.doFinal(data);//解密data
```

# 2.DES算法(Data Encryption Standard 数据加密标准)
- DES：数据加密标准，是对称加密算法领域中的典型算法 
- 特点：密钥偏短（56位）、生命周期短（避免被破解） 
- Java实现

1）生成密钥
```java
KeyGenerator keyGen = KeyGenerator.getInstance("DES");//密钥生成器
keyGen.init(56);//初始化密钥生成器
SecretKey secretKey = keyGen.generateKey();//生成密钥
byte[] key = secretKey.getEncoded();//密钥字节数组
```
2）加密
```java
SecretKey secretKey = new SecretKeySpec(key, "DES");//恢复密钥
Cipher cipher = Cipher.getInstance("DES");//Cipher完成加密或解密工作类
cipher.init(Cipher.ENCRYPT_MODE, secretKey);//对Cipher初始化，加密模式
byte[] cipherByte = cipher.doFinal(data);//加密data
```
3）解密
```java
SecretKey secretKey = new SecretKeySpec(key, "DES");//恢复密钥
Cipher cipher = Cipher.getInstance("DES");//Cipher完成加密或解密工作类
cipher.init(Cipher.DECRYPT_MODE, secretKey);//对Cipher初始化，解密模式
byte[] cipherByte = cipher.doFinal(data);//解密data
```

# 3.3DES算法(Triple DES、DESede，三重DES加密算法)
- 3DES：将密钥长度增至112位或168位，通过增加迭代次数提高安全性 
- 缺点：处理速度较慢、密钥计算时间较长、加密效率不高 
- Java实现

1）生成密钥
```java
KeyGenerator keyGen = KeyGenerator.getInstance("DESede");//密钥生成器
keyGen.init(168);  //可指定密钥长度为112或168，默认为168   
SecretKey secretKey = keyGen.generateKey();//生成密钥
byte[] key = secretKey.getEncoded();//密钥字节数组
```

2）3DES加密
```java
SecretKey secretKey = new SecretKeySpec(key, "DESede");//恢复密钥
Cipher cipher = Cipher.getInstance("DESede");//Cipher完成加密或解密工作类
cipher.init(Cipher.ENCRYPT_MODE, secretKey);//对Cipher初始化，解密模式
byte[] cipherByte = cipher.doFinal(data);//加密data
```
3）3DES解密
```java
SecretKey secretKey = new SecretKeySpec(key, "DESede");//恢复密钥
Cipher cipher = Cipher.getInstance("DESede");//Cipher完成加密或解密工作类
cipher.init(Cipher.DECRYPT_MODE, secretKey);//对Cipher初始化，解密模式
byte[] cipherByte = cipher.doFinal(data);//解密data
```
# 4.RSA算法
RSA公钥加密算法是1977年由罗纳德·李维斯特（Ron Rivest）、阿迪·萨莫尔（Adi Shamir）和伦纳德·阿德曼（Leonard Adleman）一起提出的。1987年首次公布，当时他们三人都在麻省理工学院工作。RSA就是他们三人姓氏开头字母拼在一起组成的。
> RSA算法原理如下：
>1.随机选择两个大质数p和q，p不等于q，计算N=pq； 
>2.选择一个大于1小于N的自然数e，e必须与(p-1)(q-1)互素。 
>3.用公式计算出d：d×e = 1 (mod (p-1)(q-1)) 。
>4.销毁p和q。

1)生成秘钥

通过openssl工具生成RSA的公钥和私钥
[RSA密钥的生成与配置](http://blog.csdn.net/tsuliuchao/article/details/8447690)

2)获取公钥
```java
    private PublicKey getPublicKey(){
        PublicKey publicKey = null;
        try {
            InputStream in =  RSAApp.instance.getResources().getAssets().open("rsa_public_key.pem");
            BufferedReader br = new BufferedReader(new InputStreamReader(in));
            String readLine = null;
            StringBuilder sb = new StringBuilder();
            while ((readLine = br.readLine()) != null) {
                if (readLine.charAt(0) == '-') {
                    continue;
                } else {
                    sb.append(readLine);
                }
            }
            KeyFactory keyFactory = KeyFactory.getInstance("RSA");
            byte[] buffer=Base64.decode(sb.toString());
            EncodedKeySpec keySpec = new X509EncodedKeySpec(buffer);
            publicKey = keyFactory.generatePublic(keySpec);
            return publicKey;
        } catch (Exception e) {
        }
        return publicKey;
    }
```

3)获取私钥
```java
 private PrivateKey getPrivateKey(){
        PrivateKey privateKey = null;
        try {
            InputStream in = RSAApp.instance.getResources().getAssets().open("pkcs8_rsa_private_key.pem");
            BufferedReader br = new BufferedReader(new InputStreamReader(in));
            String readLine = null;
            StringBuilder sb = new StringBuilder();
            while ((readLine = br.readLine()) != null) {
                if (readLine.charAt(0) == '-') {
                    continue;
                } else {
                    sb.append(readLine);
                }
            }
            in.close();
            KeyFactory keyFactory = KeyFactory.getInstance("RSA");
            byte[] buffer= Base64.decode(sb.toString());
            EncodedKeySpec privateKeySpec = new PKCS8EncodedKeySpec(buffer);
            privateKey = keyFactory.generatePrivate(privateKeySpec);
            return privateKey;

        } catch (Exception e) {
        }
        return privateKey;
    }
```

4)公钥加密
```java
    public String encode(String str) {
        if (TextUtils.isEmpty(str)) {
            return str;
        }
        try {
            Cipher cipher = Cipher.getInstance("RSA/ECB/PKCS1Padding");
            cipher.init(Cipher.ENCRYPT_MODE, getPublicKey());
            byte[] enBytes = cipher.doFinal(str.getBytes());
            return Base64.encode(enBytes);
        } catch (Exception e) {
        }
        return str;
    }
```

5)私钥解密
```java
    public String decode(String str) {
        if (TextUtils.isEmpty(str)) {
            return str;
        }
        try {
            Cipher cipher = Cipher.getInstance("RSA/ECB/PKCS1Padding");
            cipher.init(Cipher.DECRYPT_MODE, getPrivateKey());
            byte[] base64 = Base64.decode(str);
            byte[] deBytes = cipher.doFinal(base64);
            return new String(deBytes, "UTF-8");
        } catch (Exception e) {
        }
        return "";
    }
```

# 5.MD5算法
**MD5加密有哪些特点？**

- 压缩性：任意长度的数据，算出的MD5值长度都是固定的。
- 容易计算：从原数据计算出MD5值很容易。
- 抗修改性：对原数据进行任何改动，哪怕只修改1个字节，所得到的MD5值都有很大区别。
- 强抗碰撞：已知原数据和其MD5值，想找到一个具有相同MD5值的数据（即伪造数据）是非常困难的。

**MD5应用场景：**

- 一致性验证
- 数字签名
- 安全访问认证

**java加密**
```java
 public static String md5(String string) {
        if (TextUtils.isEmpty(string)) {
            return "";
        }
        MessageDigest md5 = null;
        try {
            md5 = MessageDigest.getInstance("MD5");
            byte[] bytes = md5.digest(string.getBytes());
            String result = "";
            for (byte b : bytes) {
                String temp = Integer.toHexString(b & 0xff);
                if (temp.length() == 1) {
                    temp = "0" + temp;
                }
                result += temp;
            }
            return result;
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
        return "";
    }
```

# 参考文献
[Java利用DES/3DES/AES 三种算法分别实现对称加密](http://blog.csdn.net/smartbetter/article/details/54017759)
[Android数据加密之Rsa加密](http://www.cnblogs.com/whoislcj/p/5470095.html)
[Android数据加密之Aes加密](http://www.cnblogs.com/whoislcj/p/5473030.html)
[Android数据加密之Des加密](http://www.cnblogs.com/whoislcj/p/5580950.html)
[Android数据加密之MD5加密](http://www.cnblogs.com/whoislcj/p/5885006.html)

>注意：不管是对称加密还是非对称加密，加密的填充方式也是十分重要的。
>
>android系统的RSA实现填充方式是"RSA/None/NoPadding"，而标准JDK实现是"RSA/None/PKCS1Padding",当与服务器交互时，需要注意加密的填充方式。

| 算法 | 密钥长度 | 默认密钥长度 | 工作模式 | 填充方式 |
| :-----: | :----: | :---------: | :---: | :---: | 
| DES | 56 |  56    |ECB、CBC、PCBC、CTR、CTS、CFB、CFB8-CFB128、OFB、OFB8-OFB128|NoPadding、PKCS5Padding、ISO10126Padding|
| 3DES| 112、168 |  168   |ECB、CBC、PCBC、CTR、CTS、CFB、CFB8-CFB128、OFB、OFB8-OFB128|NoPadding、PKCS5Padding、ISO10126Padding|
| AES | 128、192、256   |  128  |ECB、CBC、PCBC、CTR、CTS、CFB、CFB8-CFB128、OFB、OFB8-OFB128|NoPadding、PKCS5Padding、ISO10126Padding|

