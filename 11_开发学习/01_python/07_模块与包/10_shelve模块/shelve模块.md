# shelve模块

> shelve模块比pickle模块简单，只有一个open函数，返回类似字典的对象，可读可写;key必须为字符串，而值可以是python所支持的数据类型

```python
import shelve

f=shelve.open(r'sheve.txt')
f['stu1_info']={'name':'trevor','age':18,'hobby':['piao','smoking','drinking']}

print(f['stu1_info']['hobby'])
f.close()

# 结果
['piao', 'smoking', 'drinking']
```

