# blockchain-how-to-create-wallet-address
this article describes how to transform a private key into a wallet address

在区块链，特别是比特币网络，一个非常关键的组件是钱包。它主要用来实现“价值转移”，既然要转移，那就必须要有转移人和接收人，在转移过程中，我们必须确保转移的发送必须由资产的所有者发起,这就是私钥的作用，一笔交易要生效必须由资产的所有人使用它的私钥确认后才能发起，同时要有办法准确找到价值的正确接受者，这就是公钥的作用，公钥类似于银行账号用于接收转移的资产。

在前面章节我们介绍过，私钥是一个随机数，而公钥是将椭圆曲线上的G点与私钥对应的数字进行“乘法”后所得的结果。有了公钥和私钥，我们就很容易确认一笔交易的合法性。交易的发起者将他的公钥公布出来，然后使用其私钥对交易内容进行数字签名，任何人都可以用公钥来确认这笔交易的签名是否属于该发起者，只有掌握私钥的人才能实现正确的签名，在数学上可以证明，没有任何人在没有拿到私钥的情况下能伪造合法的签名。

下面我们看看钱包地址生成的具体步骤。首先假设我们有一个私钥记为N, 让它与椭圆曲线上的G点做 “乘法”操作后得到公钥P，因为公钥依然是椭圆曲线上的一点，因此P由x,y两个坐标组成，他们分别是32字节大小的数值，我们需要对这两个数值进行预处理。处理方法有两种，一种是不压缩形式，它的做法是先将这两个32字节的数值转换成以“大端”形式存储的数组，然后将这两个数组合成一个数组，最后在数组的前头再插入一个字节0x04，我们看看具体的实现方法:
```python

class S256Point(EllipticPoint):
    ...

    def sec(self):
        """
        将32字节的x,y转换成大端形式数组，然后两个数组合成一个，最后在合成的数组前头插入一个字节0x04
        """
        return b'\x04' + self.x.num.to_bytes(32, 'big') + self.y.num.to_bytes(32, 'big')

privKey = 0x038109007313a5807b2eccc082c8c3fbb988a973cacf1a7df9ce725c31b14776
pubKey = privKey * G
print(pubKey)
P = S256Point(pubKey.x, pubKey.y)
uncompress_pub_key = P.sec()
print(", ".join(hex(b) for b in uncompress_pub_key))
```
上面代码运行后结果如下：
```python
EllipticPoint(x:LimitFieldElement_0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffefffffc2f_(0x2a406624211f2abbdc68da3df929f938c3399dd79fac1b51b0e4ad1d26a47aa),y:LimitFieldElement_0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffefffffc2f_(0x9f3bc9f3948a19dabb796a2a744aae50367ce38a3e6b60ae7d72159caeb0c102),a:0000000000000000000000000000000000000000000000000000000000000000, b:0000000000000000000000000000000000000000000000000000000000000007)
0x4, 0x2, 0xa4, 0x6, 0x62, 0x42, 0x11, 0xf2, 0xab, 0xbd, 0xc6, 0x8d, 0xa3, 0xdf, 0x92, 0x9f, 0x93, 0x8c, 0x33, 0x99, 0xdd, 0x79, 0xfa, 0xc1, 0xb5, 0x1b, 0xe, 0x4a, 0xd1, 0xd2, 0x6a, 0x47, 0xaa, 0x9f, 0x3b, 0xc9, 0xf3, 0x94, 0x8a, 0x19, 0xda, 0xbb, 0x79, 0x6a, 0x2a, 0x74, 0x4a, 0xae, 0x50, 0x36, 0x7c, 0xe3, 0x8a, 0x3e, 0x6b, 0x60, 0xae, 0x7d, 0x72, 0x15, 0x9c, 0xae, 0xb0, 0xc1, 0x2

```
从输出结果可以看到，x,y两部分被分解成32字节的数组，然后两个数组合并成一个，最后在数组的最前头多了一个字节0x04。

第二种预处理方式是压缩模式。在前面章节中我们看到，在求余模式下， -x % p 等价于 (p-x) % p，因此基于有限域的椭圆曲线，如果点(x ,y)在曲线上，那么点(x , p - y)同样在曲线上。同时有限域的元素个数p我们都选取为素数，因此如果y是偶数，那么p - y就是奇数，基于这个特性，给定椭圆曲线上一点(x, y)，我们其实只需要存储x的值，然后确定y是奇数还是偶数即可，因为我们可以通过x计算出y或者p - y，因此给定曲线上一点，在压缩模式下，它的预处理步骤如下：
1，选取数组起始字节，如果y是偶数，那么起始字节取值0x02,要不然取值0x03
2，将x的值转换成32个字节的数组，以大端的模式存储，由此我们前面的sec函数可以修改如下：
```python
    def sec(self, compressed=True):
        """
        将32字节的x,y转换成大端形式数组，然后两个数组合成一个，最后在合成的数组前头插入一个字节0x04
        """
        if compressed:
            if self.y.num % 2 == 0:
                return b'\x02' + self.x.num.to_bytes(32, 'big')
            else:
                return b'\x03' + self.num.to_bytes(32, 'big')
        return b'\x04' + self.x.num.to_bytes(32, 'big') + self.y.num.to_bytes(32, 'big')
```
基于以上修改，我们前面的测试代码在压缩模式下再运行一遍，所得结果如下：
```python
EllipticPoint(x:LimitFieldElement_0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffefffffc2f_(0x2a406624211f2abbdc68da3df929f938c3399dd79fac1b51b0e4ad1d26a47aa),y:LimitFieldElement_0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffefffffc2f_(0x9f3bc9f3948a19dabb796a2a744aae50367ce38a3e6b60ae7d72159caeb0c102),a:0000000000000000000000000000000000000000000000000000000000000000, b:0000000000000000000000000000000000000000000000000000000000000007)
0x2, 0x2, 0xa4, 0x6, 0x62, 0x42, 0x11, 0xf2, 0xab, 0xbd, 0xc6, 0x8d, 0xa3, 0xdf, 0x92, 0x9f, 0x93, 0x8c, 0x33, 0x99, 0xdd, 0x79, 0xfa, 0xc1, 0xb5, 0x1b, 0xe, 0x4a, 0xd1, 0xd2, 0x6a, 0x47, 0xaa

```
在上面结果中，我们只存储了x部分，同时起始字节是0x02，因为点的y部分是偶数。采用压缩方式能够把公钥长度从64字节减少到33字节，这就非常有利于在网络上传输。但这么做也带来也给问题是，当我们接收到x后，如果把y的值计算出来。下面我们推导一下计算过程。

给定 w ^ 2 = v, 如果我们知道v, 如何在不用计算开方的情况下得到w。这个问题在求余的基础上变得比较容易。前面章节我们提到过，如果给定有限域集合的元素个数为p（p为素数)，w为集合中的一个元素，那么我们有费马小定理 w ^ (p-1) % p = 1, 在等式两边同时乘以w ^ 2就有：

w ^ 2 * w ^(p-1) % p = w ^ 2 \* 1 => [w ^ (p+1)] % p= (w ^ 2) % p

由于p是素数，因此p是奇数，于是p+1是偶数，因此 (p+1) / 2是一个整数，因此上面的等式左右两边的指数除以2就有：

w % p = (w ^ (p+1)/2) % p

注意到*, (p+1) / 2 = (2 \* (p+1)) / 4，**如果(p+1)/4是一个整数的话*** 那就有：
w % p= w ^ ((p+1)/2) % p= w ^ (2\*(p+1)/4) % p =  (w^2) ^ ((p+1)/4) % p

幸运的是对比特币的椭圆曲线，我们设置的有限域集合中元素的个数p能满足 p % 4 == 3,  因此就有 (p+1) % 4 == 0，也就是 (p+1) / 4 正好是一个整数。这里我们另 v = w ^ 2，那么就有：
v = w ^ 2, w % p = v ^ ((p+1)/4) % p

回忆一下比特币使用的椭圆曲线：y ^ 2 = x ^ 3 + 7, 将其与上面等式对应起来就有, v 对应 x ^ 3 + 7, y 对应w, 这样我们有了x之后，先计算x ^ 3 + 7的值，然后在此结果上计算其指数(p+1)/4，最后将所得结果对p求余即可得到y，如果计算所得的y是偶数，同时压缩编码的首字节是0x2，那么就可以直接返回y最为结果，如果首字节是0x3，那么我们就返回p - y，我们看看代码实现：
```python
class S256Field(LimitFieldElement):
   ...
    
    def sqrt(self):
        return self ** ((P+1) // 4)

class S256Point(EllipticPoint):
    ....
    @classmethod
    def parse(clsse, sec_array):
        """
        如果是非压缩模式，那么直接得出公钥的x,y
        """
        if sec_array[0] == 4:
            x = int.from_bytes(sec_array[1:33], 'big')
            y = int.from_bytes(sec_array[33:65], 'big')
            return S256Point(x=x, y=y)

        # 首字节如果是2那么y的值是偶数，要不然就是奇数
        is_even = (sec_array[0] == 2)
        # 先获得x部分的值
        x = S256Field(int.from_bytes(sec_array[1:], 'big'))
        #计算 y ^ 2的值，也就是 x ^ 3 + 7
        y2 = x ** 3 + S256Field(7)
        # 计算开方，也就是计算 y2 ^ ((P+1)/4)
        y = y2.sqrt()
        if y.num % 2 == 0:
            even_y = y
            odd_y = S256Field(P - even_y.num)
        else:
            even_y = S256Field(P - y.num)
            odd_y = y

        if is_even:
            return S256Point(x, even_y)
        else:
            return S256Point(x, odd_y)

```
下面我们调用上面代码测试一下效果：
```python
privKey = 0x038109007313a5807b2eccc082c8c3fbb988a973cacf1a7df9ce725c31b14776
pubKey = privKey * G

point = S256Point(pubKey.x.num, pubKey.y.num)
print(f"public key: {point}")
compress_pub_key = point.sec()
print(", ".join(hex(b) for b in compress_pub_key))

recover_pub_key = S256Point.parse(compress_pub_key)
print(f"recover pub key: {recover_pub_key}")
```
上面代码运行后结果如下：
```python
public key: S256Point(02a406624211f2abbdc68da3df929f938c3399dd79fac1b51b0e4ad1d26a47aa, 9f3bc9f3948a19dabb796a2a744aae50367ce38a3e6b60ae7d72159caeb0c102)
0x2, 0x2, 0xa4, 0x6, 0x62, 0x42, 0x11, 0xf2, 0xab, 0xbd, 0xc6, 0x8d, 0xa3, 0xdf, 0x92, 0x9f, 0x93, 0x8c, 0x33, 0x99, 0xdd, 0x79, 0xfa, 0xc1, 0xb5, 0x1b, 0xe, 0x4a, 0xd1, 0xd2, 0x6a, 0x47, 0xaa
recover pub key: S256Point(02a406624211f2abbdc68da3df929f938c3399dd79fac1b51b0e4ad1d26a47aa, 9f3bc9f3948a19dabb796a2a744aae50367ce38a3e6b60ae7d72159caeb0c102)

```
可以看到我们把公钥压缩后，又能准确的将其还原回来。完成了公钥预处理后所得数据要顺利在网络传输，我们还需要对其进行编码，这里使用的是base58编码，他是我们熟悉的base64的子集，它主要在编码数据时，去掉四个容易混淆的数字和字符，也就是数字 1 和大写字符I，数字 0 和大写字符 O，在网上有大量base58的编码实现，在这里我们给出自己的实现：
```python
#用于base58编码的字符集，这里去掉了数字 1 和大写字母I,数字 0 和大写字母O
BASE58_ALPHABET = '123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz'


def encode_base58(s):
    count = 0
    for c in s: # 统计输入的二进制数据在开头有多少个 0
        if c == 0:
            count += 1
        else:
            break

    num = int.from_bytes(s, 'big')
    prefix = '1' * count #将开头的数字 0 用字符'1'替换
    result = ''
    while num > 0:
        num, mod = divmod(num, 58)
        result = BASE58_ALPHABET[mod] + result

    return prefix + result
```
我们把上面的代码调用起来看看：
```python
s_bin = '7c076ff316692a3d7eb3c3bb0f8b1488cf72e1afcd929e29307032997a838a3d'
s_bin_base58 = encode_base58(bytes.fromhex(s_bin))
print(f"base58 encode: {s_bin_base58}")
```
代码运行后所得结果如下：
```python
base58 encode: 9MA8fRQrT4u8Zj8ZRd6MAiiyaxb2Y1CMpvVkHQu5hVM6
```

有了上面的技术准备后，我们可以开始实现钱包地址的实现。它主要包括以下几个步骤：
1，先获得公钥的预处理结果，可以是压缩格式，也可以是非压缩格式
2，将步骤1的结果先进行sha256计算，然后将计算结果再进行ripemd160哈希计算，这两步运算统称为hash160操作
3，如果要生成主网钱包地址，在步骤2结果前头加上1字节数值0x00,如果是用于测试网络的地址，那么在步骤2前头加1字节0x6f
4，将步骤3的结果连续进行两次的sha256运算，我们把这个过程称为hash256操作，然后取运算结果的前4个字节
5，将步骤4的结果添加到步骤3结果的后面
6，将步骤5的结果进行base58编码

我们看看具体的代码实现：
```python
def hash160(s):
    #先进行sha256哈希，然后再进行ripemd160哈希
    return hashlib.new("ripemd160", hashlib.sha256(s).digest()).digest()


def hash256(s):
    # 连续进行两次sha256运算
    return hashlib.sha256(hashlib.sha256(s).digest()).digest()


def encode_base58_checksum(b):
    return encode_base58(b + hash256(b)[:4])

class S256Point(EllipticPoint):
....
    def hash160(self, compressed=True):
        return hash160(self.sec(compressed))

    def address(self, compressed=True, testnet=False):
        h160 = self.hash160(compressed)
        if testnet:
            prefix = b'\x6f'
        else:
            prefix = b'\x00'

        return encode_base58_checksum(prefix + h160)
```
我们将上面实现的代码调用起来看看结果：
```python
G = S256Point(0x79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798,
              0x483ada7726a3c4655da4fbfc0e1108a8fd17b448a68554199c47d08ffb10d4b8)

privKey = 0x038109007313a5807b2eccc082c8c3fbb988a973cacf1a7df9ce725c31b14776
pubKey = privKey * G

point = S256Point(pubKey.x.num, pubKey.y.num)
print(f"mainnet address is :{point.address(compressed=True,testnet=False)}")

```
上面代码运行后所得结果如下：
```python
mainnet address is :1PRTTaJesdNovgne6Ehcdu1fpEdX7913CK
```
也就是说在给定私钥情况下，该钱包对应主网地址为1PRTTaJesdNovgne6Ehcdu1fpEdX7913CK。

更多详实的调试演示，请在b站搜索coding迪斯尼。本节代码下载地址为：

链接: https://pan.baidu.com/s/1mfWpW--FG0jbBSR9Z7OkMg 提取码: jxqi


