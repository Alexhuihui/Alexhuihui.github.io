---
title: 【安全】Java如何使用JWT
date: 2021-12-25 12:36:02
categories: 安全
tags: [JWT, 签名]
---

### 使用Java JWT 

引入maven依赖

```xml
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>3.18.2</version>
</dependency>
```

<!-- more --> 

### 生成密钥对

RSA生成1024位私钥：

```bash
$ openssl genrsa -out private.key 1024
```

RSA生成公钥:

```bash
$ openssl rsa -in private.key -pubout -out pub.key
```

通过java使用私钥必须要先经过PCK8转换

```bash
openssl pkcs8 -topk8 -nocrypt -in private.key -outform PEM -out java_private.key
```

### 从文件中读取公钥和私钥

```java
@Slf4j
public class PemUtil {
    private static byte[] parsePEMFile(File pemFile) throws IOException {
        if (!pemFile.isFile() || !pemFile.exists()) {
            throw new FileNotFoundException(String.format("The file '%s' doesn't exist.", pemFile.getAbsolutePath()));
        }
        PemReader reader = new PemReader(new FileReader(pemFile));
        PemObject pemObject = reader.readPemObject();
        byte[] content = pemObject.getContent();
        reader.close();
        return content;
    }

    private static PublicKey getPublicKey(byte[] keyBytes, String algorithm) {
        PublicKey publicKey = null;
        try {
            KeyFactory kf = KeyFactory.getInstance(algorithm);
            EncodedKeySpec keySpec = new X509EncodedKeySpec(keyBytes);
            publicKey = kf.generatePublic(keySpec);
        } catch (NoSuchAlgorithmException e) {
            System.out.println("Could not reconstruct the public key, the given algorithm could not be found.");
        } catch (InvalidKeySpecException e) {
            System.out.println("Could not reconstruct the public key");
        }

        return publicKey;
    }

    private static PrivateKey getPrivateKey(byte[] keyBytes, String algorithm) {
        PrivateKey privateKey = null;
        try {
            KeyFactory kf = KeyFactory.getInstance(algorithm);
            EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(keyBytes);
            privateKey = kf.generatePrivate(keySpec);
        } catch (NoSuchAlgorithmException e) {
            System.out.println("Could not reconstruct the private key, the given algorithm could not be found.");
        } catch (InvalidKeySpecException e) {
            System.out.println("Could not reconstruct the private key");
        }

        return privateKey;
    }

    public static PublicKey readPublicKeyFromFile(String filepath, String algorithm) {
        byte[] bytes = new byte[0];
        try {
            bytes = PemUtil.parsePEMFile(new File(filepath));
        } catch (IOException exception) {
            log.error(exception.getMessage());
        }
        return PemUtil.getPublicKey(bytes, algorithm);
    }

    public static PrivateKey readPrivateKeyFromFile(String filepath, String algorithm) {
        byte[] bytes = new byte[0];
        try {
            bytes = PemUtil.parsePEMFile(new File(filepath));
        } catch (IOException exception) {
            log.error(exception.getMessage());
        }
        return PemUtil.getPrivateKey(bytes, algorithm);
    }

}
```

### 使用私钥生成Token

```java
public static String generateToken(JSONObject jsonObject) {
        try {
            //加密时，使用私钥生成RSA算法对象
            Algorithm algorithm = Algorithm.RSA256(null, PRIVATE_KEY);
            DateTime date = DateUtil.date();
            return JWT.create()
                    //签发人
                    .withIssuer("auth-server")
                    //接收者
                    .withAudience("client")
                    //签发时间
                    .withIssuedAt(date)
                    //过期时间
                    .withExpiresAt(DateUtil.offsetMinute(date, 5))
                    //相关信息
                    .withClaim("data", jsonObject.toString())
                    //签入
                    .sign(algorithm);
        } catch (JWTCreationException exception) {
            //Invalid Signing configuration / Couldn't convert Claims.
            log.error(exception.getMessage());
        }
        return null;
    }
```



### 使用公钥校验Token

```java
public static boolean verifierToken(String token) {
        // 根据密钥对生成RS256算法对象
        Algorithm algorithm = Algorithm.RSA256(PUBLIC_KEY, null);
        JWTVerifier verifier = JWT.require(algorithm)
                .build();

        try {
            // 验证Token，verifier自动验证
            DecodedJWT jwt = verifier.verify(token);
            // 打印用户声明的信息
            Claim data = jwt.getClaim("data");
            JSONObject jsonObject = JSONUtil.parseObj(data.asString());
            for (String key : jsonObject.keySet()) {
                log.info("key === {}, value === {}", key, jsonObject.get(key));
            }
            return true;
        } catch (JWTVerificationException e) {
            log.error("Token无法通过验证! " + e.getMessage());
            return false;
        }
    }
```

### Tip

- 私钥保存在服务器端，被Auth server 持有，公钥可以颁发给需要验证Token的服务

- 使用JSONObject对象存储用户声明的信息，并放入payload中
- [本示例源码](https://github.com/Alexhuihui/common-util/blob/main/src/main/java/top/alexmmd/util/security/JwtUtil.java)