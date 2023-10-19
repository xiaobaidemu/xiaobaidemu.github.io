---
title: 如何使用Redis让周期异步任务变得Fault-tolerant且Dynamic
date: 2020-02-19
tags:
    - python
    - 调度
    - 容错
    - celery
description: Python技术栈的同学一定都非常了解Celery——基于消息队列的分布式任务调度系统。(具体用法介绍不在此赘述)。通过Celery可以快速高效的将大规模的任务实时分发到众多的不同的机器上，让用户只关注每个单独任务的处理，而非调度分配任务本身。另外，Celery中有一个非常好用的周期任务发生器，通过此进程可以帮助我们快速开发出大规模的周期任务，也就是任务的生产者。例如我们当前的应用每天就会产生15万以上的周期任务，但是与之相伴随的就是一个非常重要的问题，原生Beat在Celery中是一个单点进程，不具备容错能力。所以如何让周期任务发生器变得具有容错性(Fault-tolerant)——即周期任务产生器进程Beat挂掉后(或者机器宕机)快速恢复进程，是必须要考虑的问题。
---
<p>        Python技术栈的同学一定都非常了解Celery——基于消息队列的分布式任务调度系统。(具体用法介绍不在此赘述)。通过Celery可以快速高效的将大规模的任务实时分发到众多的不同的机器上，让用户只关注每个单独任务的处理，而非调度分配任务本身。另外，Celery中有一个非常好用的<em><span style="color: #ff6600;">周期任务发生器Beat</span></em>，通过此进程可以帮助我们快速开发出大规模的周期任务，也就是<span style="color: #ff6600;"><em>任务的生产者</em></span>。例如我们当前的应用每天就会产生15万以上的周期任务，但是与之相伴随的就是一个非常重要的问题，原生Beat在Celery中是一个单点进程，不具备容错能力。所以如何让周期任务发生器变得具有<span style="color: #ff6600;">容错性(Fault-tolerant)</span>——即周期任务产生器进程Beat挂掉后(或者机器宕机)快速恢复进程，是必须要考虑的问题。Celery在某种程度上是具有容错性的，但高可用性也是由条件的：需要保证所使用的<span style="color: #ff6600;">消息中间件的高可用性</span>，并且只保证消息存储和消费过程的容错性。而另一个问题就是原生Beat不支持<span style="color: #ff6600;">动态添加/删除/修改(Dynamic)</span>周期任务的方法，当应用的周期任务越来越多，不能因为新增一个任务而重启整个Beat进程，导致其他任务周期产生影响。</p>
<p>        当然，开源对于这两类问题，已经有很多成熟的解决方法，往往大家都殊途同归，借助消息中间件的高可用性，来进行故障转移。这里我也是站在别人的肩上，看看<a href="https://github.com/sibson/redbeat">redbeat</a>如何借助Redis来做到周期任务的高可用性（Fault-tolerant），以及周期任务的弹性动态变化能力（Resilient）。</p>
<h3>一、原生Celery4.x Beat是如何产生周期任务的？</h3>
<p>        Celery4.x中默认使用的是PersistentScheduler作为默认调度器的持久化方式，即使用<span style="color: #ff0000;">写文件的方式（序列化）</span>记录所有周期任务的最后执行时间last_run_at以及周期调度的频率(interval/crontab)，Celery<span style="color: #ff0000;">支持用户自己实现持久化的方式</span>如将每次调度任务的时间信息，记录在mysql数据库，redis/mongodb等。用户可以通过重载PersistentScheduler，自己实现存储调度任务的信息的方式。如对于<span style="color: #ff0000;">django用户可以使用django-celery-beat</span>实现通过控制台进行动态的<span style="color: #ff0000;">创建，编辑，删除周期性任务</span>。但django-celery-beat暂不确定是否可以保证容错性的beat(即允许多个beat进程同时调度)。不过，我们可以确定，要实现Beat的Fault-tolerant&Dynamic，必须要将待调度的周期性任务的相关信息，存储到某种类型的后端数据库，无论是mysql还是nosql的redis等。</p>
<p>        下面我们用一张图来说明整个原生Beat的调度方式</p>
<p><img src="https://ata2-img.cn-hangzhou.oss-pub.aliyun-inc.com/f7c2dcc779e80a106f0f76d486af290b.png" alt=""></p>
<p>        由上图可以了解到，Celery4.x中使用了最小堆的方式来计算下一次需要触发调度的周期性任务(Celery3.x并未使用此数据结构)。同时会在<span style="color: #ff0000;">内存</span>和<span style="color: #ff0000;">后端存储（数据库/文件）</span>中记录每个任务最后一次的调度时间。但是从原生Beat中也可以看出，只有当重启beat进程时才能对周期性任务进行<span style="color: #ff0000;">增删改<span style="color: #000000;">操作，并无法进行动态修改，并且不支持容错行为。</span></span></p>
<h3>二、Redbeat如何重载原生Scheduler？</h3>
<p>        使beat做到Fault-tolerant特性，重要的一点是有多节点的beat进程冗余。而要想动态的增加/删出/改变周期性任务，则需将每个任务的调度相关信息，存储在某一中心化存储中，如mysql/redis。将调度信息在后端进行存储也进一步保证了beat调度的任务准确性和唯一性。 在实现上需要做到以下五点。</p>
<ul>
<li>重载Scheduler：初始化加载以及更新所有的周期性任务，将所有周期性任务的定时信息，参数等防止到redis中存储。</li>
<li>重载ScheduleEntry: 每一个周期性任务对应一个entry，记录任务的调度周期，参数以及最近最后一次调度时间，并序列化至redis，用key-value方式记录这些信息。</li>
<li>使用redis中的Sorted Set数据结构存储每个任务下一次调度的时间，代替最小堆。</li>
<li>需要保证所有的任务的定时信息如interval/crontab等参数，均获取自redis/任何中心化存储(如diamond)，保证任务调度信息可以动态变化</li>
<li><span style="color: #3366ff;">利用简易的redis分布式锁实现Fault-tolerant，同时保证一个时刻只能有一个beat进程真实进行调度</span></li>
</ul>
<p>      下图简单的表达整个Redbeat的调度流程。</p>
<p><img src="https://ata2-img.cn-hangzhou.oss-pub.aliyun-inc.com/24d88b8a0f18977d0f7fa459e209a425.png" alt=""></p>
<p>        可以看到beat进程的执行中，通过分布式锁保证同一时刻仅有一个beat进程进行调度，其他进程均会被阻塞，<span style="background-color: #ffffff; color: #3366ff;">直到调度进程挂掉，超时后释放锁<span style="color: #000000;">。另外通过redis中Sorted Set数据结构模拟原生beat中的最小堆判断本次应该调度的任务。整个过程相对精简直观。</span></span></p>
<p><span style="background-color: #ffffff; color: #3366ff;"><span style="color: #000000;">        对于动态改变Beat中周期任务的行为，可以分为新增/删除/修改定期任务执行周期三种。</span></span></p>
<ul>
<li>新增任务：1. 将新增任务名插入至statics中，2.并且任务定义调度等信息放入对应任务名的hash对象中。3.同时把任务及对应的当前时间插入至schedule的Redis对象中，触发下一轮调度。</li>
<li>删除任务：1. 把任务相关定义调度信息所对应的hash对象删除，从而使下一轮调度时无法获取对应信息，进而调度失败。2. 删除statics中改任务对应的名称</li>
<li>修改定期任务执行的周期：直接修改任务名在redis中对应的hash对象内的调度信息即可</li>
</ul>
<p>        至此，一个简单的具有容错和动态改变任务调度信息的周期性任务即完成。</p>
<h3>三、总结：</h3>
<p>      当然此实现也有一些小的缺陷。比如当beat挂掉后，最长的恢复时间取决于redis_lock的超时时间，并且保证高可用性的最为重要的问题，即要保证redis的高可用性。对于这个问题，本人暂未深入研究。 另外对于整个当前Redbeat实现的性能的极限也并没有进行测试，对于我们的应用来说，共有400+的周期性任务，并且平均一天内这些任务一共会执行15万次，此实现对于满足我们的需求已足够。</p>
<h4>参考文章源码：</h4>
<ul>
<li><a href="https://github.com/sibson/redbeat">https://github.com/sibson/redbeat</a> (celery 3.x & celery 4.x)</li>
<li><a href="https://github.com/MnogoByte/celery-redundant-scheduler">https://github.com/MnogoByte/celery-redundant-scheduler</a> (celery 3.x)</li>
<li><a href="https://redis.io/commands">https://redis.io/commands</a></li>
<li><a href="http://docs.celeryproject.org/en/latest/userguide/periodic-tasks.html">http://docs.celeryproject.org/en/latest/userguide/periodic-tasks.html</a></li>
</ul>
