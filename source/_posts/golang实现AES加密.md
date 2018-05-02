---
title: golang实现AES加密
date: 2018-05-02 18:48:36
tags:
---

## AES算法
AES加密算法（Advanced Encryption Standard）：是美国联邦政府采用的一种区块加密标准。这个标准用来替代原先的DES。

## 原理

### 密钥扩展
1. 根据初始密钥长度（16，24，32）进行扩展，执行AES-128, AES-192, AES-256算法

### 明文加密/解密

主要包括四个子程序：
1. 字节替代(SubBytes):
矩阵中的各字节通过一个8位的S-box（Inverse S-box）进行转换

2. 行移位(ShiftRows)
第n行向左（右）移动n-1位，n从1开始

3. 列混淆(MixColumns)
每一列都在modulox^4+1之下，和一个固定多项式c(x)作乘法。

4. 轮密钥加(AddRoundKey)
将每个状态中的字节与**该回合**密钥做异或运算。

## 代码实现
    func AesEncrypt(origData, key []byte) ([]byte, error) {
        block, err := aes.NewCipher(key)
        if err != nil {
            return nil, err
        }
        blockSize := block.BlockSize()
        origData = PKCS5Padding(origData, blockSize)
        blockMode := cipher.NewCBCEncrypter(block, key[:blockSize])
        crypted := make([]byte, len(origData))
        blockMode.CryptBlocks(crypted, origData)
        return crypted, nil
    }

    func AesDecrypt(crypted, key []byte) ([]byte, error) {
        block, err := aes.NewCipher(key)
        if err != nil {
            return nil, err
        }
        blockSize := block.BlockSize()
        blockMode := cipher.NewCBCDecrypter(block, key[:blockSize])
        origData := make([]byte, len(crypted))
        blockMode.CryptBlocks(origData, crypted)
        origData = PKCS5UnPadding(origData)
        return origData, nil
    }

## Q&A
1. 字节替代中，Forward S-box，Inverse S-box 构造过程？是否与密钥或者明文相关？
S盒的构造方式如下：
（1） 按字节值得升序逐行初始化S盒。在行y列x的字节值是{yx}。
（2） 把S盒中的每个字节映射为它在有限域GF中的逆;{00}映射为它自身{00}。
（3） 把S盒中的每个字节的8个构成位记为(b7, b6, b5, b4, b3, b2, b1)。对S盒的每个字节的每个位做如下的变换：

2. 列混淆中，加密与解密的多项式如何构造

3. 轮密钥加，每个回合的密钥如何生成的


## 相关链接
[AES加密算法和RSA加密算法](https://www.jianshu.com/p/436e82a3a91a)
[wiki](https://zh.wikipedia.org/wiki/%E9%AB%98%E7%BA%A7%E5%8A%A0%E5%AF%86%E6%A0%87%E5%87%86)



