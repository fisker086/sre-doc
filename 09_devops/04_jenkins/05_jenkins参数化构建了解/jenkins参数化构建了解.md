# 参数化构建详解

```bash
主要用来区分分支，使用传参的方式，将分支名称传入脚本中进行拉取代码。
```



## 一、git branch list（and more） git分支和标签

```bash
作用：
	生成多个选项，以便选择git分支或者标签

需要提前下载Git Parameter插件
```

![5](5.png)

![1](1.png)

![2](2.png)



## 二、字符串参数（string parameter）和文本参数（test parameter）

![3](3.png)

**添加一个构件build**

![6](6.png)

![4](4.png)

**查看控制台输出结果**

![7](7.png)

```bash
字符换和文本是比较常用的，他们最大的区别是，文本可以写多行。
```



## 三、密码参数(passwd parameter)

```bash
考虑到安全的因素，需要通过参数方式传递的密码类型
```

![8](8.png)

![9](9.png)

![10](10.png)

![11](11.png)



## 四、凭据参数(Credential parameters)

```bash
不直接公开凭据，而是公开凭据的ID
```

![12](12.png)

![13](13.png)

**查看**

![14](14.png)

![15](15.png)



## 五、布尔参数（booleanParam）

```bash
布尔参数：就是真假，值为true和false
```

![16](16.png)

![19](19.png)



**检查**

![17](17.png)

![18](18.png)



## 六、隐藏参数（Hidden parameters）

```bash
创建项目时会使用到的参数，但不想版本构建的时候别人看到
```

![20](20.png)

![21](21.png)

**查看**

![22](22.png)

![23](23.png)

## 七、下拉参数（active choices parameter）

```bash
选项
```



### 1、一级

![24](24.png)

![25](25.png)

**检查**

![26](26.png)

![27](27.png)



### 2、二级（active choices reactive parameter）

![24](24.png)

![28](28.png)

![29](29.png)

![30](30.png)

![31](31.png)



































