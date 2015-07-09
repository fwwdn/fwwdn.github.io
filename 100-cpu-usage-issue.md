title: '一次线上环境CPU过高问题解决过程'
date: 2015-06-20 23:40:09
tags: java
---
今天早晨，发现海外版的一个线上服务响应很慢，接口要刷新半天。ssh上去看了内存、磁盘都正常;用top查看cpu时发现Java进程cpu占用超过100%。tail -f 看了日志，发现几个error和warn但都感觉和cpu占用高关系不大。
所以初步怀疑是某些线程有问题：

- 使用top查找出最耗cpu的进程号（PID）
![](http://img2001.static.suishenyun.net/25ece6711a70a7938cb1105810f59bd3/acb9edd3e1ad94ea83e38a8ff956653e.jpg!w480.jpg)

发现最占CPU的进程是 1414

- 使用jstack dump对应PID的堆栈信息保存备查

jstack是JDK内置的性能调优工具，对于分析线程堆栈非常有用。直接  jstack 1414 > jstack.out

![](http://img2001.static.suishenyun.net/25ece6711a70a7938cb1105810f59bd3/bcf29f1958e24af7e7bbe2172a5b658c.jpg!w480.jpg)
 
- 再次使用top查出对应PID中最耗cpu的子进程（java线程）

top -p 1414 -H

![](http://img2001.static.suishenyun.net/25ece6711a70a7938cb1105810f59bd3/c8a8b0a93c57992b636427f96e6607ee.jpg!w480.jpg)

找到1462  1482 2个线程。

- 从dump下来的堆栈信息中查找耗费性能的代码块

从上面开到，我们用top找出来的pid是十进制的。而dump下来的nid是十六进制的。利用printf命令格式化来转换一下进制（当然，也可以用其他方式）：

![](http://img2001.static.suishenyun.net/25ece6711a70a7938cb1105810f59bd3/e9199514d71fbe272723db9b1a17c97d.jpg!w480.jpg)
 
通过vim查找之前保存的线程堆栈信息，就能找到问题代码所在了。 

![](http://img2001.static.suishenyun.net/25ece6711a70a7938cb1105810f59bd3/4ab6e1bbe2e147681deec8d615ad2721.jpg!w480.jpg)
 
发现是之前的用RabbitMQ来检查业务系统更新的线程有问题， 跟了下代码，这块逻辑之前是国内版用的，海外版有了新的接口，所以直接停掉这个后台任务，部署， cpu降下来了；
 ​
![](http://img2001.static.suishenyun.net/25ece6711a70a7938cb1105810f59bd3/b60ab6ae60752f709d0c35e7f859787f.jpg!w480.jpg)