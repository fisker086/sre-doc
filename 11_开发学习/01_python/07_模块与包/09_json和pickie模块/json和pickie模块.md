# json和pickie模块

## 一、什么是序列化

> 序列化指的是把内存的数据类型转换成一个特定的格式的内容
>
> 该格式的内容可用于存储或者传输给其他平台使用

> 内存中的数据类型--->序列化--->特定的格式（json格式或者pickle格式）
>
> 内存中的数据类型<---反序列化 <---特定的格式（json格式或者pickle格式）

## 二、为什么要序列化

> 持久化存储
>
> 跨平台数据交互

**强调**

> 针对用途一：可以使一种专用的格式=> pickle只有python可以识别
>
> 针对用途二：应该是一种通用，能够被所有语言识别的格式=> json

## 三、json

> ​    如果我们要在不同的编程语言之间传递对象，就必须把对象序列化为标准格式，比如XML，但更好的方法是序列化为JSON，因为JSON表示出来就是一个字符串，可以被所有语言读取，也可以方便地存储到磁盘或者通过网络传输。JSON不仅是标准格式，并且比XML更快，而且可以直接在Web页面中读取，非常方便。

> JSON表示的对象就是标准的JavaScript语言的对象，JSON和Python内置的数据类型对应如下：

![1Json和Python内置的数据类型对应如下](https://lskypro.picturebed.dudewu.top/i/2024/10/23/6718c42e71bb1.png)

### 1、进行序列化和反序列化操作

```python
import json
dic={'name':'trevor', 'age':22, 'sex':'male'}
print(type(dic))

j=json.dumps(dic)
print(j)
print(type(j))

# 结果
<class 'dict'>
{"name": "trevor", "age": 22, "sex": "male"}
<class 'str'>
```

### 2、序列化的结果写入文件

#### 1.复杂方法

```python
import json
dic={'name':'trevor', 'age':22, 'sex':'male'}

json_res=json.dumps(dic)

with open('test.json', mode='wt', encoding='utf-8') as f:
    f.write(json_res)
```

#### 2.简单方法

```python
import json
dic={'name':'trevor', 'age':22, 'sex':'male'}

with open('test.json', mode='wt', encoding='utf-8') as f:
    json.dump(dic, f)
```

### 3、从json文件中读取进行反序列化

#### 1.复杂方法

```python
import json

with open('test.json', mode='rt', encoding='utf-8') as f:
    json_res=f.read()
    dic=json.loads(json_res)
    print(dic, type(dic))

# 结果
{'name': 'trevor', 'age': 22, 'sex': 'male'} <class 'dict'>
```

#### 2.简单方法

```python
import json

with open('test.json', mode='rt', encoding='utf-8') as f:
    dic=json.load(f)
    print(dic, type(dic))
```

## 四、补充

> json格式兼容的是所有语言通用的数据类型
>
> json格式里面没有单引号这么一说

## 五、猴子补丁

### 1、什么是猴子补丁

>​      属性在运行时的动态替换，叫做猴子补丁（Monkey Patch）。
>​      猴子补丁的核心就是用自己的代码替换所用模块的源代码，详细地如下
>1，这个词原来为Guerrilla Patch，杂牌军、游击队，说明这部分不是原装的，在英文里guerilla发音和gorllia(猩猩)相似，再后来就写了monkey(猴子)。
>2，还有一种解释是说由于这种方式将原来的代码弄乱了(messing with it)，在英文里叫monkeying about(顽皮的)，所以叫做Monkey Patch。

### 2、猴子补丁的功能(一切皆对象)

> 1.拥有在模块运行时替换的功能, 例如: 一个函数对象赋值给另外一个函数对象(把函数原本的执行的功能给替换了)

```python
class Monkey:
    def hello(self):
        print('hello')

    def world(self):
        print('world')


def other_func():
    print("from other_func")



monkey = Monkey()
monkey.hello = monkey.world
monkey.hello()
monkey.world = other_func
monkey.world()
```

### 3、monkey patch的应用场景

```python
#如果我们的程序中已经基于json模块编写了大量代码了，发现有一个模块ujson比它性能更高，但用法一样，我们肯定不会想所有的代码都换成ujson.dumps或者ujson.loads,那我们可能会想到这么做
import ujson as json
#但是这么做的需要每个文件都重新导入一下，维护成本依然很高,此时我们就可以用到猴子补丁了只需要在入口处加上, 只需要在入口加上:
import json
import ujson
def monkey_patch_json():
    json.__name__ = 'ujson'
    json.dumps = ujson.dumps
    json.loads = ujson.loads

monkey_patch_json() # 之所以在入口处加，是因为模块在导入一次后，后续的导入便直接引用第一次的成果

#其实这种场景也比较多, 比如我们引用团队通用库里的一个模块, 又想丰富模块的功能, 除了继承之外也可以考虑用MonkeyPatch.采用猴子补丁之后，如果发现ujson不符合预期，那也可以快速撤掉补丁。个人感觉MonkeyPatch带了便利的同时也有搞乱源代码的风险!
```

## 六、pickle

### 1、使用

```python
import pickle

dic={'name':'alvin','age':23,'sex':'male'}
print(type(dic))#<class 'dict'>
j=pickle.dumps(dic)
print(type(j))#<class 'bytes'>
  
f=open('序列化对象_pickle','wb')#注意是w是写入str,wb是写入bytes,j是'bytes'
f.write(j)  #-------------------等价于pickle.dump(dic,f)
f.close()
#-------------------------反序列化
import pickle
f=open('序列化对象_pickle','rb')
 
data=pickle.loads(f.read())#  等价于data=pickle.load(f)
print(data['age']) 
```

### 2、python2和python3的兼容性问题

```python
# coding:utf-8
import pickle

with open('a.pkl',mode='wb') as f:
    # 一：在python3中执行的序列化操作如何兼容python2
    # python2不支持protocol>2，默认python3中protocol=4
    # 所以在python3中dump操作应该指定protocol=2
    pickle.dump('你好啊',f,protocol=2)

with open('a.pkl', mode='rb') as f:
    # 二：python2中反序列化才能正常使用
    res=pickle.load(f)
    print(res)
```

> Pickle的问题和所有其他编程语言特有的序列化问题一样，就是它只能用于Python，并且可能不同版本的Python彼此都不兼容，因此，只能用Pickle保存那些不重要的数据，不能成功地反序列化也没关系。 