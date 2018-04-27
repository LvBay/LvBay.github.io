---
title: golang实现DES加密
date: 2018-04-27 11:35:28
tags:
---

## DES简介
DES是一个分组加密算法，典型的DES以64位为分组对数据加密，加密和解密用的是同一个算法。它的密钥长度是56位（因为每个第8 位都用作奇偶校验），密钥可以是任意的56位的数，而且可以任意时候改变。其中有极少数被认为是易破解的弱密钥，但是很容易避开它们不用。所以保密性依赖于密钥。

### 概念隐喻

- 加密 - 做蛋糕
- 明文 - 原材料
- IV初始向量 - 厨师. IV长度必须与block.Size保持一致. 维基百科上说重复使用IV会导致安全隐患
- block - 一套工具
- 工作模式 - 工具使用说明书,有EBC,CBC等
- Padding填充模式 - 检查原材料.块密码只能对确定长度的数据块进行处理，而消息的长度通常是可变的.因此部分模式（即ECB和CBC）需要最后一块在加密前进行填充

### 加密过程:

- block = 密钥切割
- pdata = padding + data
- 加密算法 = 工作模式+初始向量+block 代码:cipher.NewCBCEncrypter(block,iv)
- 密文 = 加密算法 + pdata 

### 解密过程(解密算法其实是对密文的加密)

- block = 密钥切割
- 解密算法 = 工作模式+初始向量+block 代码:cipher.NewCBCDecrypter(block,iv)
- 明文padding = 解密算法 + 密文
- 明文 = unpadding + 明文padding
    

EBC(Electronic codebook)和CBC(Cipher-block Chain)的比较:
- EBC 同样的明文块会被加密成相同的密文块.
- CBC 每个明文块先与前一个密文块进行异或后，再进行加密.
- 安全强度上CBC更强
- CBC无法并行加密

### 代码实现

    // DES加密
    func DesEncrypt(origData, key []byte) ([]byte, error) {
        // 根据密钥生成block
        block, err := des.NewCipher(key)
        if err != nil {
            return nil, err
        }
        // 根据block和初始向量生成加密算法,IV长度与block.size需要保持一致
        blockMode := cipher.NewCBCEncrypter(block, IV)
        // 扩充 origData
        origData = PKCS5Padding(origData, block.BlockSize())
        crypted := make([]byte, len(origData))
        // 对padded data加密
        blockMode.CryptBlocks(crypted, origData)
        return crypted, nil
    }

    // DES解密
    func DesDecrypt(crypted, key []byte) ([]byte, error) {
        // 根据密钥生成block
        block, err := des.NewCipher(key)
        if err != nil {
            return nil, err
        }
        // 根据block和初始向量生成解密算法,IV长度与block.size需要保持一致
        blockMode := cipher.NewCBCDecrypter(block, IV)
        // 对密文解密
        origData := make([]byte, len(crypted))
        blockMode.CryptBlocks(origData, crypted)
        // 反扩充,获取原始明文
        origData = PKCS5UnPadding(origData)
        return origData, nil
    }

### 相关链接:
[DES加密算法原理](https://www.jianshu.com/p/c44a8a1b7c38)

[工作模式 ECB,CBC](https://zh.wikipedia.org/wiki/%E5%88%86%E7%BB%84%E5%AF%86%E7%A0%81%E5%B7%A5%E4%BD%9C%E6%A8%A1%E5%BC%8F)