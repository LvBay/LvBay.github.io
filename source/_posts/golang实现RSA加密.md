---
title: golang实现RSA加密
date: 2018-05-05 09:38:14
tags:
---

## 简介
1976年之前，所有加密方式都是同一种模式：甲乙双方使用同一种加密算法和密钥，即对称加密。这种模式有一个弱点：甲方必须先把加密规则和密钥告诉乙方。这样导致保存和传递密钥，成为了一个头疼的问题。

1976年，两位美国计算机学家Whitfield Diffie 和 Martin Hellman，提出了一种崭新构思，可以在不直接传递密钥的情况下，完成解密。这被称为"Diffie-Hellman密钥交换算法"。这个算法启发了其他科学家。人们认识到，**加密和解密可以使用不同的密钥**，只要这两种规则之间存在某种对应关系即可，这样就避免了直接传递密钥。

这种新的加密模式被称为"非对称加密算法"。

RSA是非对称加密中的一种。

## 数学原理
大整数的因数分解，是一件非常困难的事情。例如你可以对3233进行因数分解（61×53），但对较大整数进行因数分解就比较困难了。

这里提供一个golang实现的暴力因数分解，有兴趣可以跑一下试试：

    // 因数分解
    // 因为大整数s可能会超过uint64的最大值，所以这里用字符串来作为输入参数
    func factorization(s string) (x, y *big.Int, err error) {
        df := big.NewInt(1)
        x = big.NewInt(2)
        n, flag := big.NewInt(1).SetString(s, 0)
        if !flag {
            return nil, nil, errors.New("invalid string")
        }
        for ; x.Cmp(n) == -1; x.Add(x, big.NewInt(1)) {
            fmt.Println("x:", x.String())
            if df.Mod(n, x).String() == "0" {
                return x, df.Div(n, x), nil
            }
        }
        return nil, nil, errors.New("end faild")
    }

## 密钥生成
A想与B进行加密通信，首先要生成一个密钥：

1. A选择两个大质数，得到乘积n，再随机选择一个整数e，计算e对于φ(n)的模反元素d。

2. 将（n，e）作为公钥，（n，d）作为私钥

可以证明，如果想通过公钥获取私钥，难度取决于大整数因数分解的效率。

关于模反元素和相关推导，可以看[链接](http://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)，比想象中简单

## 加密和解密

### 公钥加密

B想给A发送信息m

1. 将m转化为整数（字符串可以取ascii值或unicode值），且m必须小于n。

2. 根据公式
    m^e ≡ c (mod n)
已知m（明文），e（公钥的一部分），n（公钥的一部分），可以得到密文c

### 私钥解密

1. 得到密文c

2. 根据公式
    c^d ≡ m (mod n)
已知 c（密文），d（私钥的一部分），n（私钥的一部分），可以得到明文m

[证明根据根据以上公式获取的明文和原始明文一致](http://www.ruanyifeng.com/blog/2013/07/rsa_algorithm_part_two.html)

## golang代码实现RSA密钥生成

    //生成公钥和私钥 pem文件

    package encryp

    import (
        "crypto/rand"
        "crypto/rsa"
        "crypto/x509"
        "encoding/pem"
        "os"
    )

    //生成 私钥和公钥文件
    func GenRsaKey(bits int) error {
        //生成私钥文件
        privateKey, err := rsa.GenerateKey(rand.Reader, bits)
        if err != nil {
            return err
        }
        derStream := x509.MarshalPKCS1PrivateKey(privateKey)
        block := &pem.Block{
            Type:  "RSA PRIVATE KEY",
            Bytes: derStream,
        }
        file, err := os.Create("private.pem")
        if err != nil {
            return err
        }
        err = pem.Encode(file, block)
        if err != nil {
            return err
        }
        //生成公钥文件
        publicKey := &privateKey.PublicKey
        defPkix, err := x509.MarshalPKIXPublicKey(publicKey)
        if err != nil {
            return err
        }
        block = &pem.Block{
            Type:  "RSA PUBLIC KEY",
            Bytes: defPkix,
        }
        file, err = os.Create("public.pem")
        if err != nil {
            return err
        }
        err = pem.Encode(file, block)
        if err != nil {
            return err
        }
        return nil
    }

## golang实现RSA加密和解密

    // 读取公钥私钥
    var privateKey, publicKey []byte
    func init() {
        var err error
        publicKey, err = ioutil.ReadFile("public.pem")
        if err != nil {
            os.Exit(-1)
        }
        privateKey, err = ioutil.ReadFile("private.pem")
        if err != nil {
            os.Exit(-1)
        }
    }

    // 公钥加密
    func RsaEncrypt(data []byte) ([]byte, error) {
        //解密pem格式的公钥
        block, _ := pem.Decode(publicKey)
        if block == nil {
            return nil, errors.New("public key error")
        }
        // 解析公钥
        pubInterface, err := x509.ParsePKIXPublicKey(block.Bytes)
        if err != nil {
            return nil, err
        }
        // 类型断言
        pub := pubInterface.(*rsa.PublicKey)
        //加密
        return rsa.EncryptPKCS1v15(rand.Reader, pub, data)
    }

    // 私钥解密
    func RsaDecrypt(ciphertext []byte) ([]byte, error) {
        //获取私钥
        block, _ := pem.Decode(privateKey)
        if block == nil {
            return nil, errors.New("private key error!")
        }
        //解析PKCS1格式的私钥
        priv, err := x509.ParsePKCS1PrivateKey(block.Bytes)
        if err != nil {
            return nil, err
        }
        // 解密
        return rsa.DecryptPKCS1v15(rand.Reader, priv, ciphertext)
    }

## 相关链接
[RSA算法原理-欧拉函数](http://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)
[RSA算法原理](http://www.ruanyifeng.com/blog/2013/07/rsa_algorithm_part_two.html)