---
title: 加密解密解决方案
tags:
  - encrypt
categories: 编程
date: 2024-12-06 21:43:19
---

# Encrypt

## 3种加密类型

### 数字签名

- 不安全,容易被破解, 因为没用采用秘钥的方式, 但是执行效率快
- 比如: MD5  Sha56  Base64

### 对称加密算法

- 较安全, 因为采用的是单秘钥的方式, 一把秘钥同时支持加密解密, 执行效率中等
- 比如: AES  DES

### 非对称加密算法

- 很安全, 因为采用的是公私钥结合的方式, 公钥加密, 私钥解密, 执行效率最慢
- 比如: RSA
	- RSA密钥本质上是二进制数据,但二进制数据不方便: Base64编码将二进制数据转换成可打印的可读ASCII字符,便于处理和传输

![17334951657141733495165602.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17334951657141733495165602.png)

## 证书
- 生成证书的过程，实际上是**创建了一对 RSA 公钥和私钥**，然后把它们存储在**密钥库**（keystore）中，比如：
- `.jks`（Java Keystore，Java 使用的密钥库）
- `.p12`（PKCS12 标准密钥库，兼容性更好）
- `.pem`（开放标准的密钥格式）
## 例子
- 采用了AES对称加密(单秘钥)和RSA非对称加密(公私钥)
	- 先用AES秘钥加密请求体, 然后用RSA公钥加密AES秘钥, 放入请求头
- 好处: 
	- 双重加密, 不易被破解 
	- 要解密两次, 性能差
```java
// 采用java.util和Hutool工具包
@Test
public void encryptJson() {
    Log log = LogFactory.get();
    log.info("*******************加密中****************************");

    // 1. 生成 RSA 公钥和私钥（Base64 编码）
    RSA rsa = new RSA();
    String privateKeyBase64 = rsa.getPrivateKeyBase64();
    String publicKeyBase64 = rsa.getPublicKeyBase64();

    // 将 Base64 字符串还原为 Key 对象 -> 枚举
    RSA rsaFromBase64 = new RSA(privateKeyBase64, publicKeyBase64);

    // 2. 公钥加密 AES 密钥
    String aesKey = "1234567890abcdef"; // 固定一个 16 字节的字符串作为 AES 密钥
    byte[] encryptedAesKey = rsaFromBase64.encrypt(aesKey.getBytes(), KeyType.PublicKey);

    // 3. 用 AES 密钥加密 JSON 字符串
    AES aes = SecureUtil.aes(aesKey.getBytes()); // 使用 AES 密钥初始化 AES 实例
    String encryptedHex = aes.encryptHex("Im JsonString");

    log.info("********************解密中*************************");

    // 4. 私钥解密 AES 密钥
    byte[] decryptedAesKeyBytes = rsaFromBase64.decrypt(encryptedAesKey, KeyType.PrivateKey);
    String decryptedAesKey = new String(decryptedAesKeyBytes);

    // 5. 用解密后的 AES 密钥解密 JSON 数据
    AES decryptedAes = SecureUtil.aes(decryptedAesKey.getBytes());
    String decryptedJsonString = decryptedAes.decryptStr(encryptedHex);

    log.info("********************测试数据*************************");
    log.info("Public Key (Base64): {}", publicKeyBase64);
    log.info("Private Key (Base64): {}", privateKeyBase64);
    log.info("Encrypted Hex: {}", encryptedHex);
    log.info("Decrypted JSON String: {}", decryptedJsonString);
}
```
![17334951277141733495127306.png](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17334951277141733495127306.png)
