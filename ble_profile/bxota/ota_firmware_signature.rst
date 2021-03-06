OTA固件签名
============

通过OTA固件签名，可以保证芯片只会接收采用经过签名的固件，可以消除使用未签名固件非法进行固件升级的风险。

固件签名环境要求：

Python 3, python-ecsda(https://github.com/warner/python-ecdsa)

#. 如果已经生成签名密钥对，跳过此步，若尚未生成，执行tools/signing/key_gen.py生成密钥对：

    signing_key.pem （私钥，用于签名）
    
    verifying_key.pem（公钥，用于验证）
    
    verifying_key.bin （verifying_key.pem的二进制格式）
    
    verifying_key.txt （verifying_key.pem的数组形式，用于拷贝到固件工程C代码中）
    
#. 将公钥加入固件，在bx_app_config.h中定义宏BXOTAS_VERIFYING_KEY

    #define BXOTAS_VERIFYING_KEY {0x62, 0xd2, 0x09, 0x03, 0x2a, 0x4f, 0xbe, 0x54, 0x83, 0xf6, 0xc8, 0xda, 0x20, 0xeb, 0x43, 0x9d, 0xc7, 0x8c, 0x79, 0xb0, 0x2b, 0x3f, 0x55, 0x37, 0xe7, 0x1d, 0x83, 0x69, 0x9f, 0x4b, 0x9c, 0xd9, 0xec, 0xd1, 0xbd, 0x02, 0x59, 0x5d, 0xbe, 0x73, 0xc1, 0x80, 0x08, 0x0b, 0x35, 0x6c, 0x27, 0xbc, 0x1c, 0x65, 0x88, 0xbe, 0x2f, 0xc6, 0x2a, 0xeb, 0xd8, 0x9e, 0x48, 0x3b, 0x49, 0xc1, 0xe5, 0xde, }
    
#. 利用tools/signing/signing.py生成固件签名,产生signature.bin：

    python3 signing.py [ota_firmware].bin signing_key.pem

#. 通过APP进行固件升级时，需选择固件和相应的签名文件，才能升级成功。