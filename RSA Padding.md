
## Padding Type
```c
# define RSA_PKCS1_PADDING       1
# define RSA_SSLV23_PADDING      2
# define RSA_NO_PADDING          3
# define RSA_PKCS1_OAEP_PADDING  4
# define RSA_X931_PADDING        5
/* EVP_PKEY_ only */
# define RSA_PKCS1_PSS_PADDING   6
```

## RSA signature and encryption schemes
PKCS-V1_5方法包括签名和加密，也能够直接验签和解密  
RSA-PSS和RSA-OAEP都需要外部提供参数：
1. hash function
2. mask generation function（MGF）
### RSA signature schemes
#### RSASSA-PKCS1-v1_5
RFC: https://datatracker.ietf.org/doc/html/rfc2313  
```c
/*
 * The format is
 * 00 || 01 || PS || 00 || D
 * PS - padding string, at least 8 bytes of FF
 * D  - data.
 */
```
#### RSASSA-PSS (RSASSA = RSA Signature Scheme with Appendix)
RFC: https://datatracker.ietf.org/doc/html/rfc3447  
RSASSA-PSS is a probabilistic signature scheme (PSS) with appendix.  
该方法需要签名的元数据来验证签名（签名中无法提取元数据）  
* RSASSA-PSS参数如下：  
```c
hashAlgorithm       sha1, [minimum recommended SHA-256].
maskGenAlgorithm    mgf1SHA1 (the function MGF1 with SHA-1)
saltLength          默认是20 bytes，一般使用hash函数的输出长度hLen，如果salt length为零，会导致确定性的签名结果,
trailerField        trailerFieldBC (the byte 0xbc)
```
* 签名流程：  
**padding -> 私钥加密 -> signature**  
* padding流程：  
```c
   __________________________________________________________________

                                  +-----------+
                                  |     M     |
                                  +-----------+
                                        |
                                        V
                                      Hash
                                        |
                                        V
                          +--------+----------+----------+
                     M' = |Padding1|  mHash   |   salt   |
                          +--------+----------+----------+
                                         |
               +--------+----------+     V
         DB =  |Padding2|maskedseed|   Hash
               +--------+----------+     |
                         |               |
                         V               |    +--+
                        xor <--- MGF <---|    |bc|
                         |               |    +--+
                         |               |      |
                         V               V      V
               +-------------------+----------+--+
         EM =  |    maskedDB       |maskedseed|bc|
               +-------------------+----------+--+
   __________________________________________________________________

   Figure 2: EMSA-PSS encoding operation.  Verification operation
   follows reverse steps to recover salt, then forward steps to
   recompute and compare H.
```
* 验签流程：  
**signature -> 公钥解密 -> verify the m and EM**  

#### Differences between signature schemes RSASSA-PKCS-v1_5 and RSASSA-PSS
1. PKCSV1_5的结果是确定的，每次签名都会产生相同的结果。PSS是随机的，每次都会产生不一样的签名（除非salt length == 0）。
2. PKCSV1_5可以单独完成验签，PSS需要提供额外参数，比如s

### RSA encryption schemes
#### RSAES-PKCS-v1_5

#### RSAES-OAEP (Optimal Asymmetric Encryption Padding)
Encoding流程：  
```c
   __________________________________________________________________

                             +----------+---------+-------+
                        DB = |  lHash   |    PS   |   M   |
                             +----------+---------+-------+
                                            |
                  +----------+              V
                  |   seed   |--> MGF ---> xor
                  +----------+              |
                        |                   |
               +--+     V                   |
               |00|    xor <----- MGF <-----|
               +--+     |                   |
                 |      |                   |
                 V      V                   V
               +--+----------+----------------------------+
         EM =  |00|maskedSeed|          maskedDB          |
               +--+----------+----------------------------+
   __________________________________________________________________

   Figure 1: EME-OAEP encoding operation.  lHash is the hash of the
   optional label L.  Decoding operation follows reverse steps to
   recover M and verify lHash and PS.
```
#### Differences between encryption schemes RSAES-PKCS-v1_5 vs RSAES-OAEP

## Openssl
### PKCS #1 v2.0 EMSA-PKCS1-v1_5
```C
/*
 * The format is
 * 00 || 01 || PS || 00 || D
 * PS - padding string, at least 8 bytes of FF
 * D  - data.
 */
```

### PKCS #1 v2.0 EME-OAEP

### RSASSA-PSS

## Boringssl
### PKCS #1 v2.0 EMSA-PKCS1-v1_5
> RSA_add_pkcs1_prefix
>> RSA_add_pkcs1_prefix builds a version of |digest| prefixed with the  
>> DigestInfo header for the given hash function and sets |out_msg| to point to  
>> it. On successful return, if |*is_alloced| is one, the caller must release  
>> |*out_msg| with |OPENSSL_free|.  
### PKCS #1 v2.0 EME-OAEP
> RSA_padding_add_PKCS1_OAEP_mgf1
>> RSA_padding_add_PKCS1_OAEP_mgf1 writes an OAEP padding of |from| to |to|  
>> with the given parameters and hash functions. If |md| is NULL then SHA-1 is  
>> used. If |mgf1md| is NULL then the value of |md| is used (which means SHA-1  
>> if that, in turn, is NULL).  
### RSASSA-PSS
> RSA_padding_add_PKCS1_PSS_mgf1
>> RSA_padding_add_PKCS1_PSS_mgf1 writes a PSS padding of |mHash| to |EM|,  
>> where |mHash| is a digest produced by |Hash|. |RSA_size(rsa)| bytes of  
>> output will be written to |EM|. The |mgf1Hash| argument specifies the hash  
>> function for generating the mask. If NULL, |Hash| is used. The |sLen|  
>> argument specifies the expected salt length in bytes. If |sLen| is -1 then  
>> the salt length is the same as the hash length. If -2, then the salt length  
>> is maximal given the space in |EM|.  
