---
title: golang实现ECC加密
date: 2018-05-13 10:46:39
tags:
---

## 简介
椭圆曲线密码学（英语：Elliptic curve cryptography，缩写为 ECC），一种建立公开密钥加密的算法，基于椭圆曲线数学。椭圆曲线在密码学中的使用是在1985年由Neal Koblitz和Victor Miller分别独立提出的。

ECC的主要优势是在某些情况下它比其他的方法使用更小的密钥——比如RSA加密算法——提供相当的或更高等级的安全。

椭圆曲线密码学的许多形式有稍微的不同，所有的都依赖于被广泛承认的解决椭圆曲线离散对数问题的困难性上.

## 数学原理
不管是RSA还是ECC或者其它，公钥加密算法都是依赖于某个正向计算很简单（多项式时间复杂度），而逆向计算很难（指数级时间复杂度）的数学问题。

椭圆曲线依赖的数学难题是:

    k为正整数，P是椭圆曲线上的点（称为基点）, k*P=Q , 已知q和P，很难计算出k

## 密钥生成
golang 标准包中直接提供了密钥生成的方法
    
    // 初始化椭圆曲线
    pubkeyCurve := elliptic.P256()
    // 随机挑选基点,生成私钥
	p, err := ecdsa.GenerateKey(pubkeyCurve, rand.Reader)
	if err != nil {
		fmt.Println("1", err)
		os.Exit(1)
	}
	PRIVATE = p

## 加密和解密
golang标准包中没有提供加密和解密算法,但是以太坊go-ethereum实现了相关算法,这里对其进行二次封装

    // 将标准包生成私钥转化为ecies私钥
    prv2 = ecies.ImportECDSA(PRIVATE)
    // ecc(ecies)加密
    func ECCEncrypt(pt []byte) ([]byte, error) {
        ct, err := ecies.Encrypt(rand.Reader, &prv2.PublicKey, pt, nil, nil)
        return ct, err
    }
    // ecc(ecies)解密
    func ECCDecrypt(ct []byte) ([]byte, error) {
        pt, err := prv2.Decrypt(ct, nil, nil)
        return pt, err
    }

## 签名和验签
golang标准包中提供了相关方法

    func EccSign(pt []byte) (sign []byte, err error) {
        // 根据明文plaintext和私钥，生成两个big.Ing
        r, s, err := ecdsa.Sign(rand.Reader, PRIVATE, pt)
        if err != nil {
            fmt.Println(err)
            return nil, err
        }
        rs, err := r.MarshalText()
        if err != nil {
            return nil, err
        }
        ss, err := s.MarshalText()
        if err != nil {
            return nil, err
        }
        // 将r，s合并（以“+”分割），作为签名返回
        var b bytes.Buffer
        b.Write(rs)
        b.Write([]byte(`+`))
        b.Write(ss)
        return b.Bytes(), nil
    }

    func EccSignVer(pt, sign []byte) bool {
        var rint, sint big.Int
        // 根据sign，解析出r，s
        rs := bytes.Split(sign, []byte("+"))
        rint.UnmarshalText(rs[0])
        sint.UnmarshalText(rs[1])
        // 根据公钥，明文，r，s验证签名
        v := ecdsa.Verify(&PRIVATE.PublicKey, pt, &rint, &sint)
        return v
    }

## Q&A
### ECC,ECIES,ECDSA,ECDH 是什么关系,有哪些区别?

- ECC,全称椭圆曲线密码学（英语：Elliptic curve cryptography，缩写为 ECC）,主要是指相关数学原理
- ECIES,在ECC原理的基础上实现的一种公钥加密方法,和RSA类似
- ECDSA,在ECC原理上实现的签名方法
- ECDH在ECC和DH的基础上实现的密钥交换算法

### RSA算法的密钥对可以通过.pem文件来保存,ECC的密钥对要如何保存呢?

go-ethereum提供了两种对外接口.

密钥对和[]byte之间的转换

    // 私钥 -> []byte
    // FromECDSA exports a private key into a binary dump.
    func FromECDSA(priv *ecdsa.PrivateKey) []byte {
        if priv == nil {
            return nil
        }
        return math.PaddedBigBytes(priv.D, priv.Params().BitSize/8)
    }

    // []byte -> 私钥
    // ToECDSA creates a private key with the given D value.
    func ToECDSA(d []byte) (*ecdsa.PrivateKey, error) {
        return toECDSA(d, true)
    }

    // 公钥 -> []byte
    func FromECDSAPub(pub *ecdsa.PublicKey) []byte {
        if pub == nil || pub.X == nil || pub.Y == nil {
            return nil
        }
        return elliptic.Marshal(S256(), pub.X, pub.Y)
    }

    // []byte -> 公钥
        func ToECDSAPub(pub []byte) *ecdsa.PublicKey {
        if len(pub) == 0 {
            return nil
        }
        x, y := elliptic.Unmarshal(S256(), pub)
        return &ecdsa.PublicKey{Curve: S256(), X: x, Y: y}
    }

**公钥转换的时候需要注意,eth中默认使用的椭圆曲线是S256().生成密钥对时,如果使用的是其他椭圆曲线(如P256),需要对FromECDSAPub,ToECDSAPub稍作修改**
    
密钥对和文件之间的转换,其实就是在中间加一层hex编码

    // 私钥 -> 文件
    // SaveECDSA saves a secp256k1 private key to the given file with
    // restrictive permissions. The key data is saved hex-encoded.
    func SaveECDSA(file string, key *ecdsa.PrivateKey) error {
        k := hex.EncodeToString(FromECDSA(key))
        return ioutil.WriteFile(file, []byte(k), 0600)
    }

    // 文件 -> 私钥
    func LoadECDSA(file string) (*ecdsa.PrivateKey, error) {
        buf := make([]byte, 64)
        fd, err := os.Open(file)
        if err != nil {
            return nil, err
        }
        defer fd.Close()
        if _, err := io.ReadFull(fd, buf); err != nil {
            return nil, err
        }

        key, err := hex.DecodeString(string(buf))
        if err != nil {
            return nil, err
        }
        return ToECDSA(key)
    }

## 相关链接
[椭圆曲线数学原理 (上篇)](https://zhuanlan.zhihu.com/p/35225057)

[椭圆曲线数学原理 (下篇)](https://zhuanlan.zhihu.com/p/35587405)

[ECIES,ECDSA,ECDH三者的区别](https://crypto.stackexchange.com/questions/12823/ecdsa-vs-ecies-vs-ecdh)

[以太坊ecies实现](https://github.com/ethereum/go-ethereum/tree/master/crypto/ecies)
