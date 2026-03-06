# Dockerfile

Dockerfile是由一行行指令语句构成的一个创建docker镜像的配置文件。Dockerfile是由三个部分组成：基础镜像（必须的）、运行指令、容器默认执行命令。

## 一、FROM

```bash
FROM 指定基础镜像，目的是为了给构建镜像提供一个基础环境
```



## 二、MAINTAINER

```bash
指定维护者信息
```



## 三、RUN

```bash
基于FROM指定的docker镜像运行一个指令，将结果反映到新生成的镜像。RUN指令后面执行的命令必须是镜像中已经存在了的命令。
```



## 四、CMD

```bash
指定容器运行的默认命令

CMD ["/usr/sbin/php-fpm","-c","/etc/php-fpm.d/www.conf"]

CMD ["nginx","-g","daemon off;"]
```



## 五、ADD和COPY

```bash
ADD : 将本地文件添加到镜像
	ADD支持自动解压，但是仅仅支持解压tar包
	ADD支持远程下载，但是不会解压下载内容
	
COPY ： 将文件复制到镜像
```



## 六、ENV

```bash
设置一个容器的环境变量
```



## 七、EXPOSE

```bash
指定容器需要向外界暴露的端口，实际上没有暴露，只有指定了EXPOSE才能够使用-P, 可以指定多个端口
```



## 八、ARG

```bash
指定运行时参数，用于构建docker镜像时传入参数 --build-arg=USER=root
```



## 九、VOLUME

```bash
提示需要挂载的目录，没有实现挂载
```



## 十、WORKDIR

```bash
设置工作目录

1、程序运行的开始目录
2、进入容器的最初目录
```



## 十一、ONBUILD

```bash
ONBUILD 后面跟的是Dockerfile指令不是linux命令

构建触发器  ： 当前镜像被用作基础镜像时触发
```

