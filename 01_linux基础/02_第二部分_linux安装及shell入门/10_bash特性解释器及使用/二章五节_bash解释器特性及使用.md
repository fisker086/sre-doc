# Bash解释器特性及使用

## 一、tab自动补全

>命令和文件自动补全 注意：Tab只能补全命令和文件

```bash
[root@xiaowu ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth0
```



## 二、常用快捷键

```bash
tab：命令或者路径的补全键
ctrl+a：光标回到命令行首
ctrl+e：光标回到命令行尾
ctrl+insert：复制命令行内容
shift+insert：粘贴命令行内容
ctrl+k：（剪切）删除光标到行尾的内容
ctrl+u：（剪切）删除光标到行首的内容
ctrl+c：中断终端正在执行的任务或者删除整行
ctrl+d：退出当前shell命令行
ctrl+r：搜索命令使用过的历史命令记录
ctrl+z：暂停在终端运行的任务
ctrl+l：清屏等同于clear
alt+. ：引用上一个命令的最后一个参数，等介于!$
```

## 三、历史命令

### 1、查看历史命令

```bash
[root@xiaowu ~]# history
```

### 2、清空历史命令

```bash
[root@xiaowu ~]# history
```

### 3、查看历史命令保存文件

```bash
cat ~/.bash_history
```

### 4、修改保存文件保存行数

```bash
[root@xiaowu ~]# vim /etc/profile
...
HISTSIZE=2000	#历史命令保存为2000条
...
```

### 5、运行历史命令

```bash
1.键盘上下键
2.ctrl+r 	#搜索历史命令关键字
3.!72		#执行历史命令中第72条命令
4.!`字符串`	#运行历史命令中最近的以`字符串`开头的命令
5.!$	#引用上一个命令的最后一个参数
	[root@xiaowu ~]# ll /home
	drwx------ 2 mysql  mysql  62 Apr 14 19:49 mysql
	drwx------ 2 xiaowu xiaowu 62 Jul 20 19:46 xiaowu
	[root@xiaowu ~]# cd !$
	cd /home
	[root@xiaowu /home]#
```



## 四、别名

### 1、查看当前系统所有别名

```bash
[root@xiaowu ~]# alias
```

### 2、创建别名

```bash
[root@xiaowu ~]# alias xiaowu='cat /etc/passwd'
[root@xiaowu ~]# xiaowu
```

### 3、删除别名

```bash
[root@xiaowu ~]# unalias xiaowu
[root@xiaowu ~]# xiaowu
```



## 五、复合命令

### 1、;

```bash
cd /etc/sysconfig/network-scripts/;ll;cat ifcfg-eth0
```

> 效果
>
> ```bash
> cd /etc/sysconfig/network-scripts/====>ll=====>cat ifcfg-eth0
> ```

> 注意
>
> ```bash
> 如果三段命令中有错误，错误的命令不能执行，正确的命令还会执行
> ```



### 2、&&

```bash
cd /etc/sysconfig/network-scripts/||ll||cat ifcfg-eth0
```

> 效果
>
> ```bash
> cd /etc/sysconfig/network-scripts/====>ll=====>cat ifcfg-eth0
> ```

> 注意
>
> ```bash
> 如果前面的命令中有错误，命令就会终止执行，也就说不会在执行后面的命令了，即使是后面的命令是正确的
> ```

### 3、||

```bash
cd /etc/sysconfig/network-scripts/||ll||cat ifcfg-eth0
```

> 效果
>
> ```bash
> cd /etc/sysconfig/network-scripts/
> ```

> 注意
>
> ```bash
> 如果每个命令被双竖线(||)分隔符分隔，如果命令遇到可以成功执行的命令，那么命令停止执行，即使后面还有正确的命令则后面的所有命令都将得不到执行。假如命令一开始就执行失败，那么就会执行 || 后的下一个命令，直到遇到有可以成功执行的命令为止，假如所有的都失败，则所有这些失败的命令都会被尝试执行一次：
> ```



## 六、命令的运行顺序

```bash
bash shell查找命令顺序：
==>以路径（绝对路径，相对路径）开始命令，例如：/bin/ls 或 cd /bin; ./ls 
	==> alias 
		==> Compound Commands 
			==> function 
				==> build_in，如cd，kill，pwd、alias、echo等，可以用"type -a 命令"查看 
					==> hash 
						==> $PATH，环境变量，查看环境变量echo $PATH，例如/bin/ls 
							==> error: command not found
```

> 查看命令的位置
>
> ```bash
> which 命令
> ```



## 七、查看命令帮助

### 1、man线上查询及帮助命令

#### 1.语法格式

```bash
man [选项] [参数] 命令
```

#### 2.选项

```bash
-a		//在所有的man帮助手册中搜索
-f  	//等价于whatis指令，显示给定关键字的简短描述信息
-P		//指定内容时使用分页程序
-M		//指定man手册搜索的路径
-k		//关键词搜索
```

#### 3.数字参数

```bash
数字：（指定要搜索帮助的关键字）
	1 		//标准用户命令
	2		//系统调用
	3		//库调用
	4		//特殊文件（设备文件）的访问入口
	5 		//文件格式（配置文件的语法），指定程序的运行特性
	6		//游戏
	7		//杂项
	8 		//系统管理命令
	9		//跟kernel有关的文件
	注：kernel：内核
```

#### 4.手册格式

```bash
NAME			name			命令名称及功能简要说明
SYNOPSIS		synopsis		用法说明包括可用选项
DESCRIPTION		description		命令功能的详细说明，可能包括每一个选项的意义
OPTIONS			options			说明每一项的意义
FILES			files			此命令的相关配置文件
BUGS			bugs			报告程序bug的方式
EXAMPLES		examples		使用示例
SEE ALSO		see also		另外按照
COMMANDS		commands		执行程序时，可以执行的命令
COPYRIGHT		copyright		版权信息相关说明
AUTHOR			author			作者介绍
```

#### 5.man手册的使用方法

```bash
[Page Down]/空格	//向文件尾部翻一屏
[Page Up]/b			//向文件首部翻一屏
[Home]				//跳转到第一页
[end]				//跳转到最后一页
ctrl+d				//向文件尾部翻半凭
ctrl+u				//向文件首部翻半凭
回车				//一次向文件尾部翻一行
k					//一次向文件首部翻一行
G					//跳转至最后一行
NG					//跳转至指定行
1G					//跳转至文件第一行，首部
/xxx				//从文件首部向文件尾部依次查找xxx
?xxx				//从文件尾部向文件首部依次查找xxx
n					//使用“/”或“?”搜索时，在当前搜索方向下一个匹配查询
N					//使用“/”或“?”搜索时，在当前搜索方向前一个匹配查询
q					//结束本次man帮助
/-h   				#/搜索-h参数
```



### 2、help命令（查看命令帮助文档）

#### 1.help内置命令（用于内部命令）

##### 1）语法格式

```bash
help [选项] [内置命令]
```

##### 2）选项参数

```bash
-d 		//输出内置命令的简单描述
-m		//以man帮助的格式显示
-s		//只输出命令的使用语法
```



#### 2.--help 外部命令

```bash
命令 --help
[root@xiaowu ~]# ls --help
```

### 3、百度、谷歌搜索

### 4、查看官方文档

## 八、常用小命令

### 1、设置主机名

```bash
[root@xiaowu ~]# hostnamectl set-hostname 123
[root@xiaowu ~]# bash
[root@123 ~]#
```

### 2、设置系统默认启动级别

```bash
runlevel 		#查看当前运行的级别
systemctl get-default			#查看开机默认运行级别
 systemctl set-default graphical.target  #图形界面
 systemctl set-default multi-user.target #字符终端
```

### 3、查看网络

```bash
ifconfig
ip address	#可以简写为ip a
ifconfig eth0	#查看某张网卡的网络信息
```



### 4、时间

#### 1.语法

```bash
date [参数选项] [标记列表]
```

#### 2.参数选项

```bash
-d 			#datestr : 显示 datestr 中所设定的时间 (非系统时间)
--help 		#显示辅助讯息
-s 			#datestr:将系统时间设为 datestr 中所设定的时间
-u 			#显示目前的格林威治时间
--version 	#显示版本编号
```

#### 3.标记列表

```bash
%H 小时(以00-23来表示)。 
%I 小时(以01-12来表示)。 
%K 小时(以0-23来表示)。 
%l 小时(以0-12来表示)。 
%M 分钟(以00-59来表示)。 
%P AM或PM。 
%r 时间(含时分秒，小时以12小时AM/PM来表示)。 
%s 总秒数。起算时间为1970-01-01 00:00:00 UTC。 
%S 秒(以本地的惯用法来表示)。 
%T 时间(含时分秒，小时以24小时制来表示)。 
%X 时间(以本地的惯用法来表示)。 
%Z 市区。 
%a 星期的缩写。 
%A 星期的完整名称。 
%b 月份英文名的缩写。 
%B 月份的完整英文名称。 
%c 日期与时间。只输入date指令也会显示同样的结果。 
%d 日期(以01-31来表示)。 
%D 日期(含年月日)。 
%j 该年中的第几天。 
%m 月份(以01-12来表示)。 
%U 该年中的周数。 
%w 该周的天数，0代表周日，1代表周一，异词类推。 
%x 日期(以本地的惯用法来表示)。 
%y 年份(以00-99来表示)。 
%Y 年份(以四位数来表示)。 
%n 在显示时，插入新的一行。 
%t 在显示时，插入tab。 
MM 月份(必要) 
DD 日期(必要) 
hh 小时(必要) 
mm 分钟(必要)
ss 秒(选择性) 
```

#### 4.查看时间

```bash
[root@xiaowu ~]# date
Wed Jul 21 09:42:03 CST 2021
[root@xiaowu ~]# date "+%Y_%m_%d %H-%M-%S"
2021_07_21 09-42-10
```

#### 5.修改时间

```bash
date -s "2018-05-17 09:51:50"
hwclock -w 			#把系统时间写入硬件时间
hwclock -s			#把硬件时间写到系统时间
```

#### 6.网络时间同步

```bash
ntpdate 时间同步服务器地址
	ntp1.aliyun.com
	ntp2.aliyun.com
	ntp3.aliyun.com
	ntp4.aliyun.com
	ntp5.aliyun.com
	ntp6.aliyun.com
	ntp7.aliyun.com
```

> 关闭ntp服务
>
> ```bash
> timedatectl set-ntp no
> ```

### 5、关机、重启、注销命令

#### 1.重启或者关机

##### 1）语法格式

```bash
shutdown [OPTION] TIME [MESSAGE]
shutdown  [参数] [时间] 消息
```

##### 2）参数选项

```bash
-r：重启系统  例如：shutdown -r now
-h：关机  例如：shutdown -h now
```

##### 3）使用范例

###### ①重启

```bash
shutdown -r +10 		// 10分钟后重启
shutdown -r 0 		// 立即重启
shutdown -r now 	// 立即重启（生产常用）
init 6 				// 立即重启：切换运行级别到6,6表示重启
reboot				// 立即重启（生产常用）
shutdown -r 11:00  	// 11:00重启
```

###### ②关机

```bash
shutdown -h now    	//立刻关机（生产常用）
shutdown -h +1      // 一分钟后关机
shutdown -h 11:00   //11:00关机
halt                //立即停止系统，需要人工关闭电源
init0               //关机：切换运行级别到0,0表示关机
poweroff            //立即停止系统，并且关闭电源
```

###### ③取消将要进行的关机或者重启

```bash
showdown -c
```

#### 2.注销

```bash
logout            # 注销退出当前用户窗口
exit			  #注销退出当前用户窗口，快捷键ctrl+d
```

