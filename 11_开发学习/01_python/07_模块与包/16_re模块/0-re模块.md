# re模块

## 一、什么是正则？

>​    **正则就是用一些具有特殊含义的符号组合到一起（称为正则表达式）来描述字符或者字符串的方法。或者说：正则就是用来描述一类事物的规则。**
>
>**（在Python中）它内嵌在Python中，并通过 re 模块实现。正则表达式模式被编译成一系列的字节码，然后由用 C 编写的匹配引擎执行。**
>
>**生活中处处都是正则：**
>
>  **比如我们描述：4条腿**
>
>  　　**你可能会想到的是四条腿的动物或者桌子，椅子等**
>
>  **继续描述：4条腿，活的**
>
>​     **就只剩下四条腿的动物这一类了**

## 二、常用匹配模式(元字符)

![1-常用匹配模式](https://lskypro.picturebed.dudewu.top/i/2024/10/23/6718c5d6abde1.png)

## 三、使用

### 1、匹配模式

```python
#一对一的匹配

#正则匹配
import re
#\w与\W
print(re.findall('\w','hello trevor 123')) #['h', 'e', 'l', 'l', 'o', 't', 'r', 'e', 'v', 'o', 'r', '1', '2', '3']
print(re.findall('\W','hello trevor 123')) #[' ', ' ']

#\s与\S
print(re.findall('\s','hello  trevor  123')) #[' ', ' ', ' ', ' ']
print(re.findall('\S','hello  trevor  123')) #['h', 'e', 'l', 'l', 'o', 't', 'r', 'e', 'v', 'o', 'r', '1', '2', '3']

#\n \t都是空,都可以被\s匹配
print(re.findall('\s','hello \n trevor \t 123')) #[' ', '\n', ' ', ' ', '\t', ' ']

#\n与\t
print(re.findall(r'\n','hello trevor \n123')) #['\n']
print(re.findall(r'\t','hello trevor\t123')) #['\n']

#\d与\D
print(re.findall('\d','hello trevor 123')) #['1', '2', '3']
print(re.findall('\D','hello trevor 123')) #['h', 'e', 'l', 'l', 'o', ' ', 't', 'r', 'e', 'v', 'o', 'r', ' ']

#\A与\Z
print(re.findall('\Ahe','hello trevor 123')) #['he'],\A==>^
print(re.findall('123\Z','hello trevor 123')) #['he'],\Z==>$
'''
^ 
指定匹配必须出现在字符串的开头或行的开头。

\A 
指定匹配必须出现在字符串的开头（忽略 Multiline 选项）。

$ 
指定匹配必须出现在以下位置：字符串结尾、字符串结尾的 \n 之前或行的结尾。

\Z 
指定匹配必须出现在字符串的结尾或字符串结尾的 \n 之前（忽略 Multiline 选项）。
'''
#^与$
print(re.findall('^h','hello egon 123')) #['h']
print(re.findall('3$','hello egon 123')) #['3']

# 重复匹配：| . | * | ? | .* | .*? | + | {n,m} |
#. : 匹配除了/n之外的任意一个字符，指定re.DOTALL后才能匹配换行符
print(re.findall('a.b','a1b')) #['a1b']
print(re.findall('a.b','a1b a*b a b aaab')) #['a1b', 'a*b', 'a b', 'aab']
print(re.findall('a.b','a\nb')) #[]
print(re.findall('a.b','a\nb',re.S)) #['a\nb']
print(re.findall('a.b','a\nb',re.DOTALL)) #['a\nb']同上一条意思一样

#* ：匹配左侧字符重复0次或者无穷次，性格贪婪
print(re.findall('ab*','bbbbbbb')) #[]
print(re.findall('ab*','a')) #['a']
print(re.findall('ab*','abbbb')) #['abbbb']

#? ：匹配左侧字符0次或者1次
print(re.findall('ab?','a')) #['a']
print(re.findall('ab?','abbb')) #['ab']
#匹配所有包含小数在内的数字
print(re.findall('\d+\.?\d*',"asdfasdf123as1.13dfa12adsf1asdf3")) #['123', '1.13', '12', '1', '3']

#.*默认为贪婪匹配
print(re.findall('a.*b','a1b22222222b')) #['a1b22222222b']

#.*?为非贪婪匹配：推荐使用
print(re.findall('a.*?b','a1b22222222b')) #['a1b']

#+ ：左侧字符重复一次或无数次，性格贪婪
print(re.findall('ab+','a')) #[]
print(re.findall('ab+','abbb')) #['abbb']

#{n,m} ：匹配左侧字符n-m次，性格贪婪
print(re.findall('ab{2}','abbb')) #['abb']
print(re.findall('ab{2,4}','abbb')) #['abb']
print(re.findall('ab{1,}','abbb')) #'ab{1,}' ===> 'ab+'
print(re.findall('ab{0,}','abbb')) #'ab{0,}' ===> 'ab*'

#[]
print(re.findall('a[1*-]b','a1b a*b a-b')) #[]内的都为普通字符了，且如果-没有被转意的话，应该放到[]的开头或结尾
print(re.findall('a[^1*-]b','a1b a*b a-b a=b')) #[]内的^代表的意思是取反，所以结果为['a=b']
print(re.findall('a[0-9]b','a1b a*b a-b a=b')) #[]内的^代表的意思是取反，所以结果为['a=b']
print(re.findall('a[a-z]b','a1b a*b a-b a=b aeb')) #[]内的^代表的意思是取反，所以结果为['a=b']
print(re.findall('a[a-zA-Z]b','a1b a*b a-b a=b aeb aEb')) #[]内的^代表的意思是取反，所以结果为['a=b']

#\# print(re.findall('a\\c','a\c')) #对于正则来说a\\c确实可以匹配到a\c,但是在python解释器读取a\\c时，会发生转义，然后交给re去执行，所以抛出异常
print(re.findall(r'a\\c','a\c')) #r代表告诉解释器使用rawstring，即原生字符串，把我们正则内的所有符号都当普通字符处理，不要转义
print(re.findall('a\\\\c','a\c')) #同上面的意思一样，和上面的结果一样都是['a\\c']

#():分组
print(re.findall('ab+','ababab123')) #['ab', 'ab', 'ab']
print(re.findall('(ab)+123','ababab123')) #['ab']，匹配到末尾的ab123中的ab
print(re.findall('(?:ab)+123','ababab123')) #findall的结果不是匹配的全部内容，而是组内的内容,?:可以让结果为匹配的全部内容
print(re.findall('href="(.*?)"','<a href="http://www.baidu.com">点击</a>'))#['http://www.baidu.com']
print(re.findall('href="(?:.*?)"','<a href="http://www.baidu.com">点击</a>'))#['href="http://www.baidu.com"']

#|
print(re.findall('compan(?:y|ies)','Too many companies have gone bankrupt, and the next one is 
```

### 2、re模块介绍

```python
import re
#1
print(re.findall('e','alex make love') )   #['e', 'e', 'e'],返回所有满足匹配条件的结果,放在列表里
#2
print(re.search('e','alex make love').group()) #e,只到找到第一个匹配然后返回一个包含匹配信息的对象,该对象可以通过调用group()方法得到匹配的字符串,如果字符串没有匹配，则返回None。

#3
print(re.match('e','alex make love'))    #None,同search,不过在字符串开始处进行匹配,完全可以用search+^代替match

#4
print(re.split('[ab]','abcd'))     #['', '', 'cd']，先按'a'分割得到''和'bcd',再对''和'bcd'分别按'b'分割

#5
print('===>',re.sub('a','A','alex make love')) #===> Alex mAke love，不指定n，默认替换所有
print('===>',re.sub('a','A','alex make love',1)) #===> Alex make love
print('===>',re.sub('a','A','alex make love',2)) #===> Alex mAke love
print('===>',re.sub('^(\w+)(.*?\s)(\w+)(.*?\s)(\w+)(.*?)$',r'\5\2\3\4\1','alex make love')) #===> love make alex

print('===>',re.subn('a','A','alex make love')) #===> ('Alex mAke love', 2),结果带有总共替换的个数


#6
obj=re.compile('\d{2}')

print(obj.search('abc123eeee').group()) #12
print(obj.findall('abc123eeee')) #['12'],重用了obj
```

```python
import re
print(re.findall("<(?P<tag_name>\w+)>\w+</(?P=tag_name)>","<h1>hello</h1>")) #['h1']
print(re.search("<(?P<tag_name>\w+)>\w+</(?P=tag_name)>","<h1>hello</h1>").group()) #<h1>hello</h1>
print(re.search("<(?P<tag_name>\w+)>\w+</(?P=tag_name)>","<h1>hello</h1>").groupdict()) #<h1>hello</h1>

print(re.search(r"<(\w+)>\w+</(\w+)>","<h1>hello</h1>").group())
print(re.search(r"<(\w+)>\w+</\1>","<h1>hello</h1>").group())
```

