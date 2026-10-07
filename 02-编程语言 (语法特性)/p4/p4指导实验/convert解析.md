## 🔍 逐段解析（含示例）

### 1. 正则表达式 & MAC 地址处理

```python
mac_pattern = re.compile(r'^([\da-fA-F]{2}:){5}([\da-fA-F]{2})$')
```

匹配形如 `aa:bb:cc:dd:ee:ff` 的 MAC 地址。

**示例：**

```python
# ✅ 匹配成功
"aa:bb:cc:dd:ee:ff"   # True
"00:11:22:33:44:55"   # True
"AA:BB:CC:DD:EE:FF"   # True（大小写不敏感）

# ❌ 匹配失败
"aa:bb:cc:dd:ee"      # False（只有5组）
"aa:bb:cc:dd:ee:ff:11" # False（7组）
"aa:bb:cc:dd:ee:gg"   # False（gg不是十六进制）
"aa-bb-cc-dd-ee-ff"   # False（分隔符是-不是:）
```



```python
def encodeMac(mac_addr_string):
    return bytes.fromhex(mac_addr_string.replace(':', ''))
```

去掉冒号，将十六进制字符串转为字节串。

**示例：**

```python
encodeMac("aa:bb:cc:dd:ee:ff")   
# 过程: "aa:bb:cc:dd:ee:ff" → "aabbccddeeff" → bytes.fromhex()
# 输出: b'\xaa\xbb\xcc\xdd\xee\xff'

encodeMac("00:11:22:33:44:55")   
# 输出: b'\x00\x11\x22\x33\x44\x55'

encodeMac("FF:EE:DD:CC:BB:AA")   
# 输出: b'\xff\xee\xdd\xcc\xbb\xaa'
```



------



```python
def decodeMac(encoded_mac_addr):
    return ':'.join(s.hex() for s in encoded_mac_addr)
```

将每个字节转为两位十六进制，并用冒号连接。

**示例：**

```python
decodeMac(b'\xaa\xbb\xcc\xdd\xee\xff')
# 过程: 每个字节转hex → ['aa','bb','cc','dd','ee','ff'] → 用':'连接
# 输出: "aa:bb:cc:dd:ee:ff"

decodeMac(b'\x00\x11\x22\x33\x44\x55')
# 输出: "00:11:22:33:44:55"

decodeMac(b'\xff\xee\xdd\xcc\xbb\xaa')
# 输出: "ff:ee:dd:cc:bb:aa"
```



------

### 2. IPv4 地址处理

python

```
def encodeIPv4(ip_addr_string):
    return socket.inet_aton(ip_addr_string)
```



使用 `inet_aton` 将点分十进制字符串转为 4 字节网络字节序。

**示例：**

python

```
encodeIPv4("192.168.1.1")
# 输出: b'\xc0\xa8\x01\x01'
# 计算: 192→0xc0, 168→0xa8, 1→0x01, 1→0x01

encodeIPv4("10.0.0.1")
# 输出: b'\x0a\x00\x00\x01'

encodeIPv4("255.255.255.0")
# 输出: b'\xff\xff\xff\x00'
```



------

python

```
def decodeIPv4(encoded_ip_addr):
    return socket.inet_ntoa(encoded_ip_addr)
```



反向转换。

**示例：**

python

```
decodeIPv4(b'\xc0\xa8\x01\x01')
# 输出: "192.168.1.1"

decodeIPv4(b'\x0a\x00\x00\x01')
# 输出: "10.0.0.1"

decodeIPv4(b'\xff\xff\xff\x00')
# 输出: "255.255.255.0"
```



------

### 3. IPv6 地址处理

python

```
def matchesIPv6(ip_addr_string):
    try:
        socket.inet_pton(socket.AF_INET6, ip_addr_string)
        return True
    except socket.error:
        return False
```



使用 `inet_pton` 验证 IPv6 地址格式。

**示例：**

python

```
# ✅ 合法格式
matchesIPv6("2001:0db8:85a3:0000:0000:8a2e:0370:7334")  # True
matchesIPv6("::1")          # True（localhost缩写）
matchesIPv6("fe80::1")      # True（链路本地）
matchesIPv6("2001:db8::1")  # True（省略前导零）

# ❌ 非法格式
matchesIPv6("192.168.1.1")           # False（这是IPv4）
matchesIPv6("2001:0db8::85a3::7334") # False（双冒号只能出现一次）
matchesIPv6("gggg::1")               # False（非法字符）
```



------

python

```
def encodeIPv6(ip_addr_string):
    return socket.inet_pton(socket.AF_INET6, ip_addr_string)
```



转为 16 字节网络字节序。

**示例：**

python

```
encodeIPv6("::1")
# 输出: b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01'

encodeIPv6("2001:db8::1")
# 输出: b'\x20\x01\x0d\xb8\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01'

encodeIPv6("fe80::1")
# 输出: b'\xfe\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01'
```



------

python

```
def decodeIPv6(encoded_ip_addr):
    return socket.inet_ntop(socket.AF_INET6, encoded_ip_addr)
```



反向转换。

**示例：**

python

```
decodeIPv6(b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01')
# 输出: "::1"

decodeIPv6(b'\x20\x01\x0d\xb8\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01')
# 输出: "2001:db8::1"

# 注意: 输出是压缩格式（省略前导零和连续零）
```



------

### 4. 整数编码/解码（支持负数补码）

python

```
def bitwidthToBytes(bitwidth):
    return int(math.ceil(bitwidth / 8.0))
```



位宽转字节数（向上取整）。

**示例：**

python

```
bitwidthToBytes(1)   # 1位 → 1字节（向上取整）
bitwidthToBytes(8)   # 8位 → 1字节
bitwidthToBytes(9)   # 9位 → 2字节（向上取整）
bitwidthToBytes(16)  # 16位 → 2字节
bitwidthToBytes(17)  # 17位 → 3字节
```



------

python

```
def encodeNum(number, bitwidth):
    byte_len = bitwidthToBytes(bitwidth)
    if number < 0:
        if number < -(2 ** (bitwidth-1)):
            raise Exception(...)
        number = (2 ** bitwidth) + number   # 转为补码对应的正数
    num_str = '%x' % number
    if number >= 2 ** bitwidth:
        raise Exception(...)
    return bytes.fromhex('0' * (byte_len * 2 - len(num_str)) + num_str)
```



负数转为补码（在指定位宽下）。

**示例：**

python

```
# 正数编码
encodeNum(5, 16)   
# 过程: 5 → 十六进制"5" → 补零到4字节 → "00000005"
# 输出: b'\x00\x00\x00\x05'

encodeNum(255, 16) 
# 过程: 255 → "ff" → 补零到4字节 → "000000ff"
# 输出: b'\x00\x00\x00\xff'

encodeNum(1337, 40)  # 5字节
# 过程: 1337 → "539" → 补零到10位 → "0000000539"
# 输出: b'\x00\x00\x00\x05\x39'

# 负数编码（补码）
encodeNum(-1, 8)     
# 过程: -1 → 2^8 + (-1) = 255 → "ff"
# 输出: b'\xff'

encodeNum(-128, 8)   
# 过程: -128 → 2^8 + (-128) = 128 → "80"
# 输出: b'\x80'

encodeNum(-1, 16)    
# 过程: -1 → 2^16 + (-1) = 65535 → "ffff"
# 输出: b'\xff\xff'

# 超出范围会报错
encodeNum(300, 8)    # 300 >= 2^8 (256) → 报错
encodeNum(-129, 8)   # -129 < -(2^7) = -128 → 报错
```



------

python

```
def decodeNum(encoded_number):
    return int(encoded_number.hex(), 16)
```



直接将字节串的十六进制表示转为整数。

**示例：**

python

```
decodeNum(b'\x00\x00\x00\x05')
# 过程: "00000005" → 转为16进制整数
# 输出: 5

decodeNum(b'\xff')
# 过程: "ff" → 转为16进制整数
# 输出: 255

decodeNum(b'\xff\xff')
# 过程: "ffff" → 转为16进制整数
# 输出: 65535

# ⚠️ 注意: decodeNum不会把补码转回负数
# 如果你编码了 -1 (得到 b'\xff')
# decodeNum(b'\xff') 返回 255，而不是 -1
```



------

### 5. 通用编码函数 `encode()`

python

```
def encode(x, bitwidth):
    byte_len = bitwidthToBytes(bitwidth)
    if (type(x) == list or type(x) == tuple) and len(x) == 1:
        x = x[0]
    if type(x) == str:
        if matchesMac(x):
            encoded_bytes = encodeMac(x)
        elif matchesIPv4(x):
            encoded_bytes = encodeIPv4(x)
        elif matchesIPv6(x):
            encoded_bytes = encodeIPv6(x)
        else:
            encoded_bytes = x   # 假设已是字节串
    elif type(x) == int:
        encoded_bytes = encodeNum(x, bitwidth)
    else:
        raise Exception(...)
    assert(len(encoded_bytes) == byte_len)
    return encoded_bytes
```



自动识别输入类型。

**示例：**

python

```
# 自动识别MAC地址
encode("aa:bb:cc:dd:ee:ff", 48)  # 48位 = 6字节
# 输出: b'\xaa\xbb\xcc\xdd\xee\xff'

# 自动识别IPv4
encode("192.168.1.1", 32)         # 32位 = 4字节
# 输出: b'\xc0\xa8\x01\x01'

# 自动识别IPv6
encode("::1", 128)                # 128位 = 16字节
# 输出: b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01'

# 自动识别整数
encode(5, 32)                     # 32位 = 4字节
# 输出: b'\x00\x00\x00\x05'

# 处理单元素列表/元组
encode([5], 32)                   # 自动取出元素5
# 输出: b'\x00\x00\x00\x05'

encode((5,), 32)                  # 自动取出元素5
# 输出: b'\x00\x00\x00\x05'

# 非地址格式的字符串，直接作为字节串
encode("hello", 40)               # 40位 = 5字节
# 输出: b'hello'（假设已经是字节串）

# 不支持的类型会报错
encode(3.14, 32)                  # float类型 → 报错
```



------

## ✅ 测试部分示例

python

```
if __name__ == '__main__':
    # MAC测试
    mac = "aa:bb:cc:dd:ee:ff"
    enc_mac = encodeMac(mac)
    assert(enc_mac == b'\xaa\xbb\xcc\xdd\xee\xff')
    dec_mac = decodeMac(enc_mac)
    assert(mac == dec_mac)
    print("✅ MAC测试通过")

    # IPv4测试
    ip0 = "10.0.0.1"
    enc_ipv4 = encodeIPv4(ip0)
    assert(enc_ipv4 == b'\x0a\x00\x00\x01')
    dec_ipv4 = decodeIPv4(enc_ipv4)
    assert(ip0 == dec_ipv4)
    print("✅ IPv4测试通过")

    # IPv6测试
    ip1 = "2001:0db8:85a3:0000:0000:8a2e:0370:7334"
    enc_ipv6 = encodeIPv6(ip1)
    dec_ipv6 = decodeIPv6(enc_ipv6)
    assert(ip1 == dec_ipv6)
    print("✅ IPv6测试通过")

    # 整数测试
    num = 1337
    byte_len = 5
    enc_num = encodeNum(num, byte_len * 8)
    assert(enc_num == b'\x00\x00\x00\x05\x39')
    dec_num = decodeNum(enc_num)
    assert(num == dec_num)
    print("✅ 整数测试通过")

    # 通用encode测试
    assert(encode(mac, 48) == enc_mac)
    assert(encode(ip0, 32) == enc_ipv4)
    assert(encode(num, 40) == enc_num)
    assert(encode([num], 40) == enc_num)
    print("✅ 通用encode测试通过")

    print("🎉 所有测试通过！")
```



------

## ⚠️ 注意事项（含示例）

### 1. 负数解码不恢复

python

```
# 编码 -1 为 8位补码
enc = encodeNum(-1, 8)    # b'\xff'

# 解码得到的是 255，不是 -1
decodeNum(enc)            # 255（不是 -1）
```



### 2. 字符串作为字节串的风险

python

```
# 如果字符串不是地址格式，直接当作字节串
encode("hello", 40)       # 输出: b'hello'

# 但如果字节串长度不匹配会报错
encode("hello", 32)       # 期望4字节，但"hello"是5字节 → assert失败
```



### 3. 类型检查不严谨

python

```
# 目前写法（不推荐）
if type(x) == list: ...   # 不能识别list子类

# 更推荐用
if isinstance(x, list): ...
```



------

这样每个函数都配有具体示例，应该更好理解了！有什么问题随时问 😊