# job和cronjob

>​    我们在工作中会遇到需要批量处理数据和分析的需求，也会有按时间来进行调度的工作，在k8s集群中，有job和cronjob两中资源对象来映带我们的这种需要。

> ​    job负责处理任务，仅执行一次的任务，他保证批处理任务的一个或多个pod成功结束。而cronjob则就是在job上加上了时间调度，相当于定时任务。

## 一、Job

>https://kubernetes.io/docs/concepts/workloads/controllers/job/

> 运行一个官方job示例，计算π到2000位，大约需要10秒钟：

```yaml
apiVersion: batch/v1
kind: Jobs
metadata:
  name: pi
spec:
  template:
  backoffLimit: 6 		#默认重试6次后才认为执行失败
  activeDeadlineSeconds: 100  #重试工作的持续时间，优先级别高于backoffLimit
    spec:
      restartPolicy: Never	#job只有两种重启策略，Never、OnFailure不支持Always,失败状态会陷入失败死循环
      containers:
      - name: pi
        image: perl
        command:
        - perl
        - Mbignum=bpi
        - wle
        - print bpip(2000)
#  请注意，作业的.spec.activeDeadlineSeconds优先于.spec.backoffLimit。因此，重试一个或多个失败Pod的Job一旦达到所指定的时间限制activeDeadlineSeconds，就不会部署其他Pod ，即使backoffLimit尚未达到。    
```

### 1、parallel(并行) Jobs

>适应job运行的三种模式：
>
>1. Non-parallel(非并行) Jobs
>   - 通常情况，除非pod发生故障，否则仅启动一个pod
>   - pod运行成功后，job成complate状态
>2. 固定parallel(并行)Jobs一个完成次数
>
>.spec.completion指定一个非零的正值
>
>job是整体任务，在1到范围内的每个值都有一个成功的Pod时完成 .spec.completions
>
>3. 具有工作队列的parallel(并行)Jobs
>   - 不指定 .spec.completions ,默认为 .spec.parallelism
>   - pod必须在彼此之间或外部服务之间进行协调，以确定每个pod应该如何处理。例如一个pod可以从工作队列中最多获取N批的批处理
>   - 每个pod都可以独立地确定其所有对等方是否都已完成，从而确定整个pod状态
>   - 当jobs中任何pod成功终止时，不会创建新的pod
>   - 所有pod成功终止，则jobs完成
>   - pod成功推出后，其他pod不应该为此任务再做任何工作或编写任何输出。他们都应该退出
>
>对于*非并行*jobs，您可以同时保留`.spec.completions`和不`.spec.parallelism`设置。两者均未设置时，均默认为1。
>
>对于*固定的完成计数*jobs，您应该设置`.spec.completions`为所需的完成数量。您可以设置`.spec.parallelism`，或不设置它，默认为1。
>
>对于*工作队列* Job，您必须保持未`.spec.completions`设置状态，并将其设置`.spec.parallelism`为非负整数。
>
>有关如何利用不同类型的jobs的更多信息，请参见[jobs模式](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/#job-patterns)部分。

### 2、处理Pod和容器故障

>​    Pod中的容器可能由于多种原因而失败，例如，由于该容器中的进程以非零退出代码退出，或者该容器因超出内存限制而被杀死等。如果发生这种情况，请使用`.spec.template.spec.restartPolicy = "OnFailure"`，然后Pod停留在节点上，但是容器重新运行。因此，您的程序需要在本地重新启动时处理该情况，或者指定`.spec.template.spec.restartPolicy = "Never"`。有关的更多信息，请参见[pod生命周期](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#example-states)`restartPolicy`

### 3、Pod后退失败策略

>​    在某些情况下，由于配置中的逻辑错误等原因，您需要在重试一定次数后使jobs失败。为此，设置`.spec.backoffLimit`为指定重试次数，然后再将jobs视为失败。默认情况下，将退避限制设置为6。与jobs相关联的失败Pod由Job控制器重新创建，并且其指数退避延迟（10s，20s，40s…）限制为六分钟。如果在jobs的下一个状态检查之前未出现新的失败Pod，则会重置退避计数。
>
>> **注意：**版本1.12之前的Kubernetes版本仍然存在问题[＃54870](https://github.com/kubernetes/kubernetes/issues/54870)
>
>> **注意：**如果您的jobs具有`restartPolicy = "OnFailure"`，请记住，一旦达到jobs退避限制，运行该jobs的容器将被终止。这会使调试jobs的可执行文件更加困难。我们建议`restartPolicy = "Never"`在调试jobs或使用日志记录系统时进行设置 ，以确保不会因疏忽而丢失失败的jobs的输出。
>
>​    请注意，作业的`.spec.activeDeadlineSeconds`优先于`.spec.backoffLimit`。因此，重试一个或多个失败Pod的Job一旦达到所指定的时间限制`activeDeadlineSeconds`，就不会部署其他Pod ，即使`backoffLimit`尚未达到。

## 二、CronJob

>https://kubernetes.io/docs/concepts/workloads/controllers/job/

>​    `CronJob`其实就是在`Job`的基础上加上了时间调度，我们可以：在给定的时间点运行一个任务，也可以周期性地在给定时间点运行。这个实际上和我们`Linux`中的`crontab`就非常类似了。
>
>​    一个`CronJob`对象其实就对应中`crontab`文件中的一行，它根据配置的时间格式周期性地运行一个`Job`，格式和`crontab`也是一样的。
>
>`crontab`的格式如下：
>
>> **分 时 日 月 星期 要运行的命令** 第1列分钟0～59 第2列小时0～23） 第3列日1～31 第4列月1～12 第5列星期0～7（0和7表示星期天） 第6列要运行的命令
>
>现在，我们用`CronJob`来管理我们上面的`Job`任务，

```yaml
apiVersion: batch/v2alpha1
kind: CronJob
metadata:
  name: cronjob-demo
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: hello
            image: busybox
            args:
            - "bin/sh"
            - "-c"
            - "for i in 9 8 7 6 5 4 3 2 1; do echo $i; done"
```

>   我们这里的`Kind`是`CronJob`了，要注意的是`.spec.schedule`字段是必须填写的，用来指定任务运行的周期，格式就和`crontab`一样，另外一个字段是`.spec.jobTemplate`, 用来指定需要运行的任务，格式当然和`Job`是一致的。还有一些值得我们关注的字段`.spec.successfulJobsHistoryLimit`和`.spec.failedJobsHistoryLimit`，表示历史限制，是可选的字段。它们指定了可以保留多少完成和失败的`Job`，默认没有限制，所有成功和失败的`Job`都会被保留。然而，当运行一个`Cron Job`时，`Job`可以很快就堆积很多，所以一般推荐设置这两个字段的值。如果设置限制的值为 0，那么相关类型的`Job`完成后将不会被保留。

## 三、使用范例

### 1、job实现数据库初始化

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-mysql-init
  namespace: linux66
spec:
  template:
    spec:
      containers:
      - name: job-mysql-init-container
        image: centos:7.9.2009  # 换成数据库客户端镜像
        command: ["/bin/sh"]
        args: ["-c", "echo data init job at `date +%Y-%m-%d_%H-%M-%S` >> /cache/data.log"] # 执行数据库初始化语句
        volumeMounts:
        - mountPath: /cache
          name: cache-volume
      volumes:
      - name: cache-volume
        hostPath:
          path: /tmp/jobdata
      restartPolicy: Never
```

### 2、cronjob实现数据库备份

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjob-mysql-databackup
spec:
  #schedule: "30 2 * * *"
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cronjob-mysql-databackup-pod
            image: centos:7.9.2009 # 换成数据库客户端镜像
            #imagePullPolicy: IfNotPresent
            command: ["/bin/sh"]
            args: ["-c", "echo mysql databackup cronjob at `date +%Y-%m-%d_%H-%M-%S` >> /cache/data.log"] # 执行数据库备份语句
            volumeMounts: 
            - mountPath: /cache
              name: cache-volume
          volumes:
          - name: cache-volume
            hostPath:
              path: /tmp/cronjobdata
          restartPolicy: OnFailure
```



