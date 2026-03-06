# hashlib模块

## 一、介绍

>​     1、什么叫hash:hash是一种算法（3.x里代替了md5模块和sha模块，主要提供 SHA1, SHA224, SHA256, SHA384, SHA512 ，MD5 算法），该算法接受传入的内容，经过运算得到一串hash值
>
>​     2、hash值的特点是：
>
>​        1.只要传入的内容一样，得到的hash值必然一样=====>要用明文传输密码文件完整性校验
>​        2.不能由hash值返解成内容=======》把密码做成hash值，不应该在网络传输明文密码
>​        3.只要使用的hash算法不变，无论校验的内容有多大，得到的hash值长度是固定的

> hash算法就像一座工厂，工厂接收你送来的原材料（可以用m.update()为工厂运送原材料），经过加工返回的产品就是hash值

![1.哈希工厂](https://lskypro.picturebed.dudewu.top/i/2024/10/23/6718c4dd1d57a.png)

## 二、使用

### 1、学会使用

```python
import hashlib
m=hashlib.md5()# m=hashlib.sha256()

m.update('hello'.encode('utf8'))
print(m.hexdigest())

m.update('trevor'.encode('utf8'))
print(m.hexdigest())


m2=hashlib.md5()

m2.update('hellotrevor'.encode('utf8'))
print(m2.hexdigest())

'''
注意：把一段很长的数据update多次，与一次update这段长数据，得到的结果一样
但是update多次为校验大文件提供了可能。
'''

# 结果
5d41402abc4b2a76b9719d911017c592
f6e02bb6ca5172d752e2496ce2289f2b
f6e02bb6ca5172d752e2496ce2289f2b
```

### 2、文件校验

#### 1.小文件逐行校验

```python
import hashlib
m=hashlib.md5()# m=hashlib.sha256()

m.update(文件的一行)
m.update(文件的一行)
m.update(文件的一行)
m.update(文件的一行)
m.update(文件的一行)
m.update(文件的一行)
print(m.hexdigest())
```

#### 2.大文件校验

> 使用f.seek指定拿文件某部分数据进行加工

```python
import hashlib
m=hashlib.md5()# m=hashlib.sha256()

m.update(第一行)
m.update(二分之一行)
m.update(三分之一)
m.update(四份之一行)
m.update(五分之二行)
m.update(七分之三行)
...
m.update(最后一行)
print(m.hexdigest())
```



## 三、撞库

> 以上加密算法虽然依然非常厉害，但时候存在缺陷，即：通过撞库可以反解。所以，有必要对加密算法中添加自定义key再来做加密。

### 1、截取加密后的密文，判断加密算法类型

> 这里就不模拟了

```python
import hashlib

m2=hashlib.md5()

m2.update('hellotrevor'.encode('utf8'))
print(m2.hexdigest())

# 结果
f6e02bb6ca5172d752e2496ce2289f2b
```

### 2、撞库

```python
import hashlib

# 截取的密文
cryptograph='f6e02bb6ca5172d752e2496ce2289f2b'

# 制作密码字典，网上有很多现成的
passwds=[
    'Trevor',
    'trevor123',
    'hellotrevor',
    'trevor@666',
    'trevor@9999',
    't76987revor',
    ]

def make_passwd_dic(passwds):
    dic={}
    for passwd in passwds:
        m=hashlib.md5()
        m.update(passwd.encode('utf-8'))
        dic[passwd]=m.hexdigest()
    return dic

print(make_passwd_dic(passwds))

# 撞库
def break_code(cryptograph,passwd_dic):
    for k,v in passwd_dic.items():
        if v == cryptograph:
            print('密码是===>\033[46m%s\033[0m' %k)

break_code(cryptograph,make_passwd_dic(passwds))

# 结果
{'Trevor': '29b67a93c1a600beb6904551c853fdd0', 'trevor123': '845cb747985a6c6424e4d3d822d75ecb', 'hellotrevor': 'f6e02bb6ca5172d752e2496ce2289f2b', 'trevor@666': 'f52db672c02c348e1a963ae61a33cc9b', 'trevor@9999': 'b1e931c3234f5e11334aba7493b6307e', 't76987revor': '115d32475d7883756424cb0faccf0cf6'}
密码是===>hellotrevor
```

## 四、增加撞库难度

> 没有绝对安全的密码或者秘钥，只能不断提升撞库的成本

### 1、更换更难破译的加密方式

### 2、不要用个人信息放在密码里

### 3、密码加盐

> 密码拆分：将天王盖地虎放到不同的位置

```python
import hashlib

m=hashlib.md5()
'tr'
'天王'
'vor'
'盖'
'hello'
'地虎'

m.update('tr'.encode('utf-8'))
m.update('天王'.encode('utf-8'))
m.update('vor'.encode('utf-8'))
m.update('盖'.encode('utf-8'))
m.update('hello'.encode('utf-8'))
m.update('地虎'.encode('utf-8'))

print(m.hexdigest())
```



