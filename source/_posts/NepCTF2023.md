---
title: NepCTF2023
date: 2023-08-13 19:21:01
tags:
---
## MISC

### 与AI共舞的哈夫曼

三血拿下！扔进vscode，然后手动def  `decompress`函数，等待copilot的AI补全，然后根据报错改改，生成的代码如下：

```python
def decompress(compressed_file, decompressed_file):
    with open(compressed_file, "rb") as f:
        data = f.read()

    # Read frequency information
    frequencies = {}
    num_of_chars = data[0]
    i = 1
    for j in range(num_of_chars):
        char = data[i]
        freq = (
            (data[i + 1] << 24) | (data[i + 2] << 16) | (data[i + 3] << 8) | data[i + 4]
        )
        frequencies[char] = freq
        i += 5

    root = build_huffman_tree(frequencies)

    # Read compressed data
    current_node = root
    decompressed_data = ""
    for byte in data[i:]:
        byte = bin(byte)[2:].rjust(8, "0")
        for bit in byte:
            if bit == "0":
                current_node = current_node.left
            else:
                current_node = current_node.right

            if current_node.char is not None:
                decompressed_data += chr(current_node.char)
                current_node = root

    """ # Remove padding
    padding = int(decompressed_data[-8:], 2)
    decompressed_data = decompressed_data[:-padding] """

    with open(decompressed_file, "w") as f:
        f.write(decompressed_data)
```

得到flag为：`Nepctf{huffman_zip_666}`

### 小叮弹钢琴

打开midi文件，前半部分的长短很容易想到摩斯密码，解码拿到：`YOUSHOULDUSETHISTOXORSOMETHING`，后面一串很明显0x开头的十六进制，顺时针旋转90°可以读出来`0x370a05303c290e045005031c2b1858473a5f052117032c39230f005d1e17`

写个脚本xor一下（摩斯结果的大小写卡了我好久）

```python
import binascii

# key = b"YOUSHOULDUSETHISTOXORSOMETHING"
key = b"youshouldusethistoxorsomething"
x = 0x370A05303C290E045005031C2B1858473A5F052117032C39230F005D1E17
xor = bytes([a ^ b for a, b in zip(key, binascii.unhexlify(hex(x)[2:]))])
print(xor)
# b'NepCTF{h4ppy_p14N0}NepCTF{h4pp'
```

### ConnectedFive

手动玩一下到42分就好了：

![img](2023-08-12-18-37-15-image.png)

### 你也喜欢三月七么

~~做人就得脑洞大开~~。看题目描述，咸的是salt（即官方群名`NepCTF2023`）,题目有iv和256，想到aes256，十六字节的key试了好久，解密不成功。后面看题目`啥256`和salt好像指向sha256编码，但是结果应该是256字节的一串，猜测可能要截断，确实，解密出了有意义的信息。（后面管理说题目里的“关键”就是指`key`，有被冷到，嘶~）

```python
from hashlib import sha256
from Crypto.Cipher import AES
import binascii

key_length = 16
iv_hex = "88219bdee9c396eca3c637c0ea436058"
ciphertext_hex = "b700ae6d0cc979a4401f3dd440bf9703b292b57b6a16b79ade01af58025707fbc29941105d7f50f2657cf7eac735a800ecccdfd42bf6c6ce3b00c8734bf500c819e99e074f481dbece626ccc2f6e0562a81fe84e5dd9750f5a0bb7c20460577547d3255ba636402d6db8777e0c5a429d07a821bf7f9e0186e591dfcfb3bfedfc"

# Convert hex strings to bytes
iv = binascii.unhexlify(iv_hex)
ciphertext = binascii.unhexlify(ciphertext_hex)
salt = b"NepCTF2023"
key = sha256(salt).digest()[:key_length]
cipher = AES.new(key, AES.MODE_CBC, iv)
plaintext = cipher.decrypt(ciphertext)
print(plaintext)
#b'6148523063484d364c793970625763784c6d6c745a3352774c6d4e76625338794d44497a4c7a41334c7a49304c336c5061316858553070554c6e42755a773d3d'
```

赛博厨子hex+base64，得到一个网址：<https://img1.imgtp.com/2023/07/24/yOkXWSJT.png>

下载图片，发现又是不知名的文字。

![img](dded7465e9b53c29b1e905113b2886dc4398a3e2.png)

各种识图工具往上扔，都识别不出来。后来联想到题目背景星穹铁道、个人玩**原神**和星穹铁道的经验、以及LitCTF的~~珠玉在前~~，猜测是米哈游的游戏内文字。百度“星穹铁道字母对照表”，找到[文字对照表 - 星穹铁道列车智库 - 灰机wiki](http://starrail.huijiwiki.com/wiki/%E6%96%87%E5%AD%97%E5%AF%B9%E7%85%A7%E8%A1%A8)然后凭借眼力搜索与一点必要的猜测，得到flag为`NepCTF{HRP_aIways_likes_March_7th}`

### 陌生的语言

hint1:A同学的英文名为“Atsuko Kagari”。搜索可知来自《小魔女学园》，然后搜各种文章，发现没有这两种文字的相关信息。某次搜索时百度智能联想到设定，同时结合前期搜索到的动画官方站wiki<http://littlewitchacademia.jp/tv1st/witchpedia/02.html>，确定两种文字为：`古代ドラゴン語`、`ルーナ文字`，谷歌搜索前几个结果未见码表，直到一推特链接<https://twitter.com/i/events/946875269771440128>，跟踪，发现龙语码表：<https://www.google.co.jp/search?q=alphabets+of+dinotopia&num=20&newwindow=1&dcr=0&source=lnms&tbm=isch&sa=X&ved=0ahUKEwi-1ZHZqLDYAhVOQLwKHU6dDKQQ_AUICigB&biw=1345&bih=963#imgrc=okTWVTFWkL_U-M>，得到flag的第一部分：`NEPNEPABELIEVING` ，在推特内搜索另一语言，终于找到码表：<https://twitter.com/araymkm_0F/status/828267705853546496>，得到flag的第二部分：`HEARTISYOURMAGIC` 根据平台所给flag格式怎么交都不对，直到更新hint：flag格式请选手根据自身语感自行添加下划线，故最后flag为：`NepCTF{NEPNEP_A_BELIEVING_HEART_IS_YOUR_MAGIC}`

### codes

我真傻，真的，我单知道C语言读取环境变量有很多种方法，肯定会禁用好多，却没想到编译前还会检查代码里是否含有`env`，改个变量名就能嘎嘎绕。以下是利用main函数的第三个参数实现打印系统变量的代码：

```c
#include<stdio.h>
int main(int argc,char** argv,char** e)
{
char** en;
for(en = e; *en !=0; en++)
{
char* this = *en;
printf("%s\n", this);
}
}
//Nepctf{easy_codes_427e8401-0944-47ad-8c45-6e9fc992ba8f_[TEAM_HASH]}
```
