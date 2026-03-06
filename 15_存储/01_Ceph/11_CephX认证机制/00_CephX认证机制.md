# CephX认证机制

>Ceph 使用 cephx 协议对客户端进行身份认证
>
>cephx 用于对 ceph 保存的数据进行认证访问和授权，用于对访问 ceph 的请求进行认证和授权检测，与 mon 通信的请求都要经过 ceph 认证通过，但是也可以在 mon 节点关闭 cephx认证，但是关闭认证之后任何访问都将被允许，因此无法保证数据的安全性

## 一、授权流程

>每个 mon 节点都可以对客户端进行身份认证并分发秘钥，因此多个 mon 节点就不存在单点故障和认证性能瓶颈

>mon 节点会返回用于身份认证的数据结构，其中包含获取 ceph 服务时用到的 session key,session key 通过客户端秘钥进行加密传输，而秘钥是在客户端提前配置好的，保存在/etc/ceph/ce ph.client.admin.keyring 文件中。
>
>客户端使用 session key 向 mon 请求所需要的服务，mon 向客户端提供一个 tiket，用于向实际处理数据的 OSD 等服务验证客户端身份，MON 和 OSD 共享同一个 secret，因此OSD 会信任所有 MON 发放的 tiket。
>
>tiket 存在有效期，过期后重新发放。
>
>注意：
>CephX 身份验证功能仅限制在 Ceph 的各组件之间，不能扩展到其他非 ceph 组件
>Ceph 只负责认证授权，不能解决数据传输的加密问题

## 二、访问流程

>无论 ceph 客户端是哪种类型，例如块设备、对象存储、文件系统，ceph 都会在存储池中。
>将所有数据存储为对象:
>ceph 用户需要拥有存储池访问权限，才能读取和写入数据
>ceph 用户必须拥有执行权限才能使用 ceph 的管理命令

## 三、Ceph用户

>用户是指个人(ceph 管理者)或系统参与者(MON/OSD/MDS)。
>
>通过创建用户，可以控制用户或哪个参与者能够访问 ceph 存储集群、以及可访问的存储池及存储池中的数据。
>
>ceph 支持多种类型的用户，但可管理的用户都属于 client 类型区分用户类型的原因在于，MON/OSD/MDS 等系统组件特使用 cephx 协议，但是它们为非客户端。
>
>通过点号来分割用户类型和用户名，格式为 TYPE.ID，例如 client.admin

### 1、查看配置文件

```bash
root@ceph-node-01:~#  cat /etc/ceph/ceph.client.admin.keyring
[client.admin]
        key = AQDNvBpmUO3VAxAA4tydG8+18ll5fSanDxo3uw==
        caps mds = "allow *"
        caps mgr = "allow *"
        caps mon = "allow *"
        caps osd = "allow *"
```

### 2、用命令查看指定组件信息

```bash
root@ceph-node-01:~# ceph auth get osd.0
[osd.0]
        key = AQDvxhpmJQBcMxAAesJKNIgSSgFRP7rz8wC9+A==
        caps mgr = "allow profile osd"
        caps mon = "allow profile osd"
        caps osd = "allow *"
root@ceph-node-01:~# ceph auth get client.admin
[client.admin]
        key = AQDNvBpmUO3VAxAA4tydG8+18ll5fSanDxo3uw==
        caps mds = "allow *"
        caps mgr = "allow *"
        caps mon = "allow *"
        caps osd = "allow *"
```

## 四、ceph的授权和使能

>ceph 基于使能/能力(Capabilities，简称 caps )来描述用户可针对 MON/OSD 或 MDS 使用的授权范围或级别。

### 1、语法

```bash
daemon-type ‘allow caps’ [...]
```

### 2、能力

>r：向用户授予读取权限。访问监视器(mon)以检索 CRUSH 运行图时需具有此能力。
>
>w：向用户授予针对对象的写入权限。
>
>x：授予用户调用类方法（包括读取和写入）的能力，以及在监视器中执行 auth 操作的能力。
>
>*：授予用户对特定守护进程/存储池的读取、写入和执行权限，以及执行管理命令的能力
>
>class-read：授予用户调用类读取方法的能力，属于是 x 能力的子集。
>
>class-write：授予用户调用类写入方法的能力，属于是 x 能力的子集。
>
>profile osd：授予用户以某个 OSD 身份连接到其他 OSD 或监视器的权限。授予 OSD 权限，使 OSD 能够处理复制检测信号流量和状态报告(获取 OSD 的状态信息)。
>
>profile mds：授予用户以某个 MDS 身份连接到其他 MDS 或监视器的权限。
>
>profile bootstrap-osd：授予用户引导 OSD 的权限(初始化 OSD 并将 OSD 加入 ceph 集群)，授权给部署工具，使其在引导 OSD 时有权添加密钥。
>
>profile bootstrap-mds：授予用户引导元数据服务器的权限，授权部署工具权限，使其在引导元数据服务器时有权添加密钥。

#### 1.MON能力

>包括 r/w/x 和 allow profile cap(ceph 的运行图)

```bash
mon 'allow rwx'
mon 'allow profile osd'
```

#### 2.OSD能力

>包括 r、w、x、class-read、class-write(类读取)）和 profile osd(类写入)，另外 OSD 能力
>还允许进行存储池和名称空间设置。

```bash
osd 'allow capability' [pool=poolname] [namespace=namespace-name]
```

#### 3.MDS能力

>只需要 allow 或空都表示允许

```bash
mds 'allow'
```

## 五、ceph用户管理

>用户管理功能可让 Ceph 集群管理员能够直接在 Ceph 集群中创建、更新和删除用户。
>
>在 Ceph 集群中创建或删除用户时，可能需要将密钥分发到客户端，以便将密钥添加到密钥环文件中/etc/ceph/ceph.client.admin.keyring，此文件中可以包含一个或者多个用户认证信息，凡是拥有此文件的节点，将具备访问 ceph 的权限，而且可以使用其中任何一个账户的权限，此文件类似于 linux 系统的中的/etc/passwd 文件。

### 1、列出用户

```bash
$ ceph auth list
```

### 2、TYPE.ID表示法

>针对用户采用 TYPE.ID 表示法，例如 osd.0 指定是 osd 类并且 ID 为 0 的用户(节点)，
>
>client.admin 是 client 类型的用户，其 ID 为 admin，
>
>另请注意，每个项包含一个 key=xxxx 项，以及一个或多个 caps 项。可以结合使用-o 文件名选项和 ceph auth list 将输出保存到某个文件。

```bash
$ ceph auth list -o 123.key
```

### 3、用户管理

>添加一个用户会创建用户名 (TYPE.ID)、机密密钥，以及包含在命令中用于创建该用户的所有能力,用户可使用其密钥向 Ceph 存储集群进行身份验证。



#### 1.创建用户

##### 1）ceph auth add

>此命令是添加用户的规范方法。它会创建用户、生成密钥，并添加所有指定的能力。

```bash
# 查看使用帮助
$ ceph auth -h

# 创建用户
$ ceph auth add client.tom mon 'allow r' osd 'allow rwx pool=mypool'

# 验证
$ ceph auth get client.tom
```

##### 2）ceph auth get-or-create

>ceph auth get-or-create 此命令是创建用户较为常见的方式之一，它会返回包含用户名（在方括号中）和密钥的密钥文，如果该用户已存在，此命令只以密钥文件格式返回用户名和密
>钥，还可以使用 -o 指定文件名选项将输出保存到某个文件

```bash
# 创建用户
$ ceph auth get-or-create client.jack mon 'allow r' osd 'allow rwx pool=mypool'

# 验证用户
$ ceph auth get client.jack

# 再次创建
$ ceph auth get-or-create client.jack mon 'allow r' osd 'allow rwx pool=mypool'
```

##### 3）ceph auth get-or-create-key

>此命令是创建用户并仅返回用户密钥，对于只需要密钥的客户端（例如 libvirt），此命令非常有用。如果该用户已存在，此命令只返回密钥。您可以使用 -o 文件名选项将输出保存到某个文件。
>
>创建客户端用户时，可以创建不具有能力的用户。不具有能力的用户可以进行身份验证，但不能执行其他操作，此类客户端无法从监视器检索集群地图，但是，如果希望稍后再添加能
>力，可以使用 ceph auth caps 命令创建一个不具有能力的用户。
>
>典型的用户至少对 Ceph monitor 具有读取功能，并对 Ceph OSD 具有读取和写入功能。
>
>此外，用户的 OSD 权限通常限制为只能访问特定的存储池

```bash
$ ceph auth get-or-create-key client.jack mon 'allow r' osd 'allow rwx pool=mypool'
```

#### 2.获取单个制定用户的key信息

```bash
$ ceph auth print-key client.jack
```

#### 3.修改用户

>使用 ceph auth caps 命令可以指定用户以及更改该用户的能力，设置新能力会完全覆盖当前的能力，因此要加上之前的用户已经拥有的能和新的能力，如果看当前能力，可以运行
>ceph auth get USERTYPE.USERID，如果要添加能力，使用以下格式时还需要指定现有能力：
>
>```bash
>ceph auth caps USERTYPE.USERID daemon 'allow [r|w|x|*|...] \
>[pool=pool-name] [namespace=namespace-name]' [daemon 'allow [r|w|x|*|...] \
>[pool=pool-name] [namespace=namespace-name]']
>```

```bash
#查看用户当前权限
$ ceph auth get client.jack

#修改用户权限
$ ceph auth caps client.jack mon 'allow r' osd 'allow rw pool=mypool'

#再次验证权限
$ ceph auth get client.jack
```

#### 4.删除用户

>要删除用户使用 ceph auth del TYPE.ID，其中 TYPE 是 client、osd、mon 或 mds 之一，ID 是用户名或守护进程的 ID。

```bash
$ ceph auth del client.tom
```

## 六、密钥环管理

>ceph 的秘钥环是一个保存了 secrets、keys、certificates 并且能够让客户端通认证访问 ceph的 keyring file(集合文件)，一个 keyring file 可以保存一个或者多个认证信息，每一个 key 都有一个实体名称加权限，类型为：

```bash
{client、mon、mds、osd}.name
```

>当客户端访问 ceph 集群时，ceph 会使用以下四个密钥环文件预设置密钥环设置：

```bash
/etc/ceph/<$cluster name>.<user $type>.<user $id>.keyring #保存单个用户的 keyring
/etc/ceph/cluster.keyring #保存多个用户的 keyring
/etc/ceph/keyring #未定义集群名称的多个用户的 keyring
/etc/ceph/keyring.bin #编译后的二进制文件
```

### 1、通过秘钥环文件备份与恢复用户

>使用 ceph auth add 等命令添加的用户还需要额外使用 ceph-authtool 命令为其创建用户秘钥环文件。
>创建 keyring 文件命令格式：

```bash
ceph-authtool --create-keyring FILE
```

#### 1.导出用户认证信息至 keyring 文件

>将用户信息导出至 keyring 文件，对用户信息进行备份

```bash
# 创建用户
$ ceph auth get-or-create client.user1 mon 'allow r' osd 'allow * pool=mypool'

# 验证用户
$ ceph auth get client.user1

#创建 keyring 文件
ceph-authtool --create-keyring ceph.client.user1.keyring

# 验证 keyring 文件，是个空文件
$ cat ceph.client.user1.keyring

# 导出 keyring 至指定文件
$ ceph auth get client.user1 -o ceph.client.user1.keyring

# 验证指定用户的 keyring 文件
$ cat ceph.client.user1.keyring
```

>在创建包含单个用户的密钥环时，通常建议使用 ceph 集群名称、用户类型和用户名及keyring 来 命 名 ， 并 将 其 保 存 在 /etc/ceph 目 录 中 。 例 如 为 client.user1 用 户 创 建ceph.client.user1.keyring。

#### 2.从 keyring 文件恢复用户认证信息

>可以使用 ceph auth import -i 指定 keyring 文件并导入到 ceph，其实就是起到用户备份和恢复的目的

```bash
# 验证用户的认证文件
$ cat ceph.client.user1.keyring

# 模拟误删除用户
$ ceph auth del client.user1

# 确认用户被删除
$ ceph auth get client.user1

# 导入用户
$ ceph auth import -i ceph.client.user1.keyring

# 验证
$ ceph auth get client.user1
```

### 2、秘钥环文件多用户

>一个 keyring 文件中可以包含多个不同用户的认证文件

#### 1.将多用户导出至秘钥环

```bash
# 创建空的 keyring 文件
$ ceph-authtool --create-keyring ceph.client.user.keyring

# 把指定的 admin 用户的 keyring 文件内容导入到 user 用户的 keyring 文件
$ ceph-authtool ./ceph.client.user.keyring --import-keyring ./ceph.client.admin.keyring

# 验证 keyring 文件
$ ceph-authtool -l ./ceph.client.user.keyring

# 再导入一个其他用户的 keyring
$ ceph-authtool ./ceph.client.user.keyring --import-keyring ./ceph.client.user1.keyring

# 再次验证 keyring 文件是否包含多个用户的认证信息
$ ceph-authtool -l ./ceph.client.user.keyring
```
