---
title: 当并发insert on duplicate key update遇见死锁--更新丢失
date: 2020-01-15
tags:
    - 数据库
    - 死锁
description: 数据库死锁问题，是一个老生常谈且很常见的问题，网上也有非常多对于各类死锁场景的解析和复现，但凡和死锁有关，无外乎不涉及数据库隔离等级、索引、以及innodb锁等相关原因。但是我这个案例我搜遍了全网没能找到比较相似情况。于是我想尽可能的复现出这种情况，找出死锁的原因，找出可能出现的隐患。
---
<p>数据库死锁问题，是一个老生常谈且很常见的问题，网上也有非常多对于各类死锁场景的解析和复现，但凡和死锁有关，无外乎不涉及数据库隔离等级、索引、以及innodb锁等相关原因。但是我这个案例我<strong>搜遍了全网</strong>也没能找到比较相似情况。于是我想尽可能的复现出这种情况，找出死锁的原因，找出可能出现的隐患。</p>
<h3>问题的背景：</h3>
<p>我们的数据库中的发生死锁的表是具有&rdquo;<em><strong><span style="color: #ff0000;">多列组合构建的唯一索引</span></strong></em>&ldquo;（不包含自增的主键），且数据库的隔离等级为<em><strong><span style="color: #ff0000;">Read Committed</span></strong></em>, 另外对于这个表来说是写入远大于读取的，由于业务的原因，经常会出现<span style="color: #ff0000;">同一数据反复插入</span>（<span style="color: #ff0000;">同一数据指唯一索引值相同的数据，但其他非索引字段可能不同</span>），所以为了简化代码，我们使用insert on duplicate key update来解决这种问题，当mysql检测到唯一键冲突时，仅更新特定（非索引）字段。但是问题就出现在大规模多worker并发插入的时候，会经常出现"<span style="color: #ff0000;">Deadlock found</span> when trying to get lock"。开始真的是百思不得其解，于是乎开始疯狂查阅mysql手册以及各类博文尝试找到问题所在。</p>
<h3>问题的现象：</h3>
<p>一般定位死锁原因第一步就是执行&rdquo;show engine innodb status&ldquo;, 查看innodb Standard monitor输出结果，这里面会有数据库最后一次的死锁记录。会记录出现死锁的两个事务，它们分别在等待什么锁，并且手里持有什么锁。mysql在检测到发生死锁的时候，会随机回滚其中的一个事务，从而解开死锁。下面的截图是发生死锁的时候innodb status截图(和业务相关的数据已脱敏，这里均用column_n和value_n表示)</p>
<p>Transaction1:</p>
<p><img src="https://ata2-img.cn-hangzhou.oss-pub.aliyun-inc.com/e883f65ef1ddb7a5571cadea33bb8a30.png" /></p>
<p>Transaction2:</p>
<p><img src="https://ata2-img.cn-hangzhou.oss-pub.aliyun-inc.com/ce9472b9ae1868f4e241e34f98ee1ed2.png" /></p>
<p>现象阐述：从上方两个截图可以发现，死锁均发生在insert on duplicate key update语句执行的时候，并且<strong><span style="color: #ff0000;">每个insert语句均为批量插入多个数据</span></strong>。对于事务一，可以看到事务一在<span style="color: #ff0000;">等待某个锁的获取<span style="color: #000000;">，且这个锁是</span></span>"lock_mode X locks gap before rec insert intention waiting"，直接翻译过来就是插入意向锁在等待排他gap锁的释放，也就是只有排他gap锁释放后插入意向锁才能获取到（关于这些锁的含义见下一节）。对于事务二，同样可以看到相同的一句话。并且两个事务的锁冲突均发生在&rdquo;<span style="color: #ff0000;">唯一索引</span>&ldquo;上。再进一步观察可以看到，事务二所持有（"Holds the Locks"下方展示的索引值）的<span style="color: #ff0000;">排它锁所在的索引<span style="color: #000000;">（锁均是加在索引上或者索引区间上的）</span></span>，与事务一<span style="color: #ff0000;">等待获取锁的索引</span>是一样的。进一步展示了的确，在同一个索引上出现了一个等待获取，一个已经获取的冲突现象。</p>
<h3>相关概念：</h3>
<p>在分析问题前，有必要概述一下（详细了解可以见附录），这里面涉及到的锁相关知识。具体可以详见<a href="https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html">Mysql手册</a></p>
<p>innodb级锁按照隔离能力，主要分为共享锁（S锁）和排他锁（X锁）。事务T1的某行上持有S锁，则另一事务T2可以在此行获取S锁，但是不能获取此行的X锁，而如果T1在某行上持有X锁，则另一事务T2，对此行既无法获取S锁，也无法获取X锁。（除了S和X锁外，还有表级锁，分别是意向共享IS锁和意向排他IX锁，这里不做深入）。</p>
<p>按照锁的种类：主要有四种。</p>
<p>        1. Record锁：这种锁会在索引上加锁，比如sql为select column_1 from table where column_1=1 for update，且column_1上有索引，则会把colunm_1为1的行都加排它锁，其他事务禁止对此行读和写。</p>
<p>        2. Gap锁（间隙锁）：这种锁作用在索引记录之间。目的只需要记住：他是为<span style="color: #ff0000;">防止其他事务插入间隙</span>（<span style="color: #ff0000;">包括防止insert方式插入新数据到间隙，以及update方式将其他行变更到此间隙</span>）。Gap锁可以有效的防止&rdquo;幻读&ldquo;（因为这些间隙都被上了锁，其他事务不可能再插入数据到这些间隙中，于是当前事务在连续进行&rdquo;当前读&ldquo;时，每次读到的都是相同的记录）。虽然Gap锁只作用在隔离级别为RR及以上的数据库上，但是不意味着隔离等级为RC级别的不会使用，在RC级别，在<span style="color: #ff0000;">进行外键约束检测和唯一键约束检测的时候，会使用到Gap锁</span>，而正是这个duplicate-key checking导致了上文出现的死锁发生。关于Gap锁到底是如何加锁的，<a href="https://www.cnblogs.com/crazylqy/p/7821481.html">可以参阅这篇文章</a>。</p>
<p><img src="https://ata2-img.cn-hangzhou.oss-pub.aliyun-inc.com/c4fde02f099e9c19a8dd9e317c5f24ff.png" /></p>
<p>        3. Next-Key锁：本质上就是Gap锁和Record锁的结合，锁住索引外还要锁住索引的间隙。再具体一些就是，一个record锁，加上，位于此索引记录前的第一个间隙处的间隙锁。举个简单的例子就是，如果现在有一个索引包含三个值1，3，5，则next-key lock锁，可能锁住的范围就有(-&infin;,1],(1,3],(3,5],(5,+&infin;]。同样在next-key lock<strong><em><span style="color: #ff0000;">一般</span></em></strong>作用在RR隔离等级的数据库，但是<span style="color: #ff0000;"><em><strong>当出现在insert时候，检测到唯一键冲突的时候，会在冲突所在唯一索引出和之前的间隙处加Next-key lock.</strong></em></span></p>
<p><img src="https://ata2-img.cn-hangzhou.oss-pub.aliyun-inc.com/b1ce2b1cc77b174690791b019438699d.png" /></p>
<p>mysq官方手册中，对Next-key lock在innodb monitor中的打印如下图所示：</p>
<p><img src="https://ata2-img.cn-hangzhou.oss-pub.aliyun-inc.com/04ede18d2555a33683012c837ca30bfe.png" /></p>
<p>可以发现和我们在&rdquo;问题的现象&ldquo;一节贴的日志中事务二&rdquo;Hold the Locks&ldquo;处非常相似。所以可以怀疑当时死锁发生的时候，出现了排他的next-key lock。</p>
<p>        4. Insert Intention锁（插入意向锁）：顾名思义，这个锁是在数据插入之前会加此锁。它是一种轻量的Gap锁，同事也是意向排他锁的一种。它的存在使得多个事务在写入不同数据到统一索引间隙的时候，不会发生锁等待。另外由于它是一种意向插入锁，所以<span style="color: #ff0000;">当排他锁已经处于间隙上的时候</span>，根据锁的兼容矩阵，可以知道，<em><strong><span style="color: #ff0000;">意向插入锁必须等待此间隙上的排它锁释放，才能获取。</span></strong></em></p>
<p>根据上面，对锁的种类说明，其实我们已经能猜到，大概是什么锁导致了死锁的出现。（这里我要再明确一点，我们的数据库隔离等级为Read Committed级别。）本质上就是两个事务同时<span style="color: #ff0000;">获取到了不同间隙的X Next-key锁</span>，而<span style="color: #ff0000;">这个两个事务又同时想要向对方已经获取了next-key锁的间隙内插入新的数据<span style="color: #000000;">，</span></span>于是乎死锁出现了。下面我们来完全复现一下。</p>
<h3>问题的复现：</h3>
<p>数据库准备：数据库中能够包含一个unique key： code</p>
<pre class="language-java"><code>CREATE TABLE `test2` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `code` int(11) NOT NULL,
  `other` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `code` (`code`)
) ENGINE=InnoDB </code></pre>
<p>初始数据：insert into test2 <span class="s1">(</span>code<span class="s1">,</span> other<span class="s1">)</span> values<span class="s1">(</span>1<span class="s1">,</span>1<span class="s1">),(</span>3<span class="s1">,</span>3<span class="s1">),(</span>5<span class="s1">,</span>5<span class="s1">)</span></p>
<p>复现场景：原始的<span class="s1">code字段</span>为<span class="s1">1,3, 5</span>，现在要在中间插入<span class="s1">code</span>为<span class="s1">2</span>，<span class="s1">3</span>，<span class="s1">4</span>，<span class="s1">5</span>的<span class="s1">row</span>, 如果碰到唯一键约束则更新<span class="s1">other</span>字段。</p>
<table style="border-collapse: collapse; width: 0%; height: 481px;" border="1">
<tbody>
<tr style="height: 34px;">
<td style="width: 14.6011%; text-align: center; height: 34px;"><strong>Time</strong></td>
<td style="width: 42.8063%; height: 34px; text-align: center;"><strong>Session1</strong></td>
<td style="width: 42.5926%; height: 34px; text-align: center;"><strong>Session2</strong></td>
</tr>
<tr style="height: 34px;">
<td style="width: 14.6011%; height: 76px; text-align: center;"><strong>T1</strong></td>
<td style="width: 42.8063%; height: 76px; text-align: left;">
<p><strong>start transaction<span class="s1">;</span></strong></p>
<p><strong>insert into </strong>test2<span class="s1">(</span><strong>code</strong><span class="s1">,</span> other<span class="s1">)</span> <strong>values </strong><span class="s1">(</span>3<span class="s1">,</span> 4<span class="s1">)</span> <strong>on duplicate key update </strong>other = <em>VALUES</em><span class="s1">(</span>other<span class="s1">);</span></p>
</td>
<td style="width: 42.5926%; height: 76px; text-align: center;"> </td>
</tr>
<tr style="height: 34px;">
<td style="width: 14.6011%; height: 92px; text-align: center;">T2</td>
<td style="width: 42.8063%; height: 92px;"> </td>
<td style="width: 42.5926%; height: 92px;">
<p class="p1"><strong>start transaction<span class="s1">;</span></strong></p>
<p class="p1"><strong>insert into </strong>test2<span class="s1">(</span><strong>code</strong><span class="s1">,</span> other<span class="s1">)</span> <strong>values </strong><span class="s1">(</span>5<span class="s1">,</span> 6<span class="s1">)</span> <strong>on duplicate key update </strong>other = <em>VALUES</em><span class="s1">(</span>other<span class="s1">);</span></p>
</td>
</tr>
<tr style="height: 34px;">
<td style="width: 14.6011%; height: 129px; text-align: center;">
<p class="p1"><strong>T3</strong></p>
</td>
<td style="width: 42.8063%; height: 129px;">
<p class="p1"><strong>insert into </strong>test2<span class="s1">(</span><strong>code</strong><span class="s1">,</span> other<span class="s1">)</span> <strong>values </strong><span class="s1">(</span>4<span class="s1">,</span> 4<span class="s1">)</span> <strong>on duplicate key update </strong>other = <em>VALUES</em><span class="s1">(</span>other<span class="s1">)</span></p>
<p class="p1"><span style="color: #ff0000;"><em><strong><span class="s1">现象：一直阻塞</span></strong></em></span></p>
</td>
<td style="width: 42.5926%; height: 129px;"> </td>
</tr>
<tr style="height: 150px;">
<td style="width: 14.6011%; height: 150px; text-align: center;">T4</td>
<td style="width: 42.8063%; height: 150px;"> </td>
<td style="width: 42.5926%; height: 150px;">
<p class="p1"><strong>insert into </strong>test2<span class="s1">(</span><strong>code</strong><span class="s1">,</span> other<span class="s1">)</span> <strong>values </strong><span class="s1">(</span>2<span class="s1">,</span> 2<span class="s1">)</span> <strong>on duplicate key update </strong>other = <em>VALUES</em><span class="s1">(</span>other<span class="s1">);</span></p>
<p class="p1"><em><strong><span style="font-size: 14px; color: #ff0000;"><span class="s1">现象：</span>出现死锁，事务回滚 (<span class="s1">2</span>,<span class="s1">2</span>插入失败， 且<span class="s1">code</span>为<span class="s1">5</span>时，更新<span class="s1">other</span>字段失败，更新和插入均丢失)</span></strong></em></p>
</td>
</tr>
</tbody>
</table>
<p>死锁出现后，我们查看innodb status中的死锁记录，如下：</p>
<p class="p1"><img src="https://ata2-img.cn-hangzhou.oss-pub.aliyun-inc.com/731d9f06a83d4c729103f65eb3a89966.png" /></p>
<p class="p1">可以发现，复现出来的结果，和上文中的案例几乎完全一致。下面我们对此结果进行分析。</p>
<h3 class="p1">问题的分析：</h3>
<ol>
<li class="p1"><span class="s1"><span class="s1"><span class="s1"><span style="color: #ff0000;">在T1时刻Session1执行完insert操作后<span style="color: #000000;">，由于插入的code=2已经存在于表中，发生了唯一键冲突，所以触发了duplicate-checking，<span style="color: #ff0000;">导致在(1,3]这个区间加上了next-key lock</span>。这里，我为了进一步证明确实只有(1,3]这个区间加了锁。在T1时刻执行完后，验证插入code=0/4/6的数据可以在Session2中执行成功。同时这个时候Session2中可以修改code=1的数据，如update test2 set other=0 where code=1可以执行成功（当然你不能update test2 set code=2 where code=1,因为这个操作是在向(1,3]的间隙内插入了数据，违反了gap锁的要求）。同时我们可以证明这时<span style="color: #ff0000;">code=3肯定是被排他锁锁住</span>的，由于当出现唯一键冲突时，就会执行on duplicate key update，更新other字段，所以code=3一定在更新结束后处于排它锁锁定状态（补充说明：可以证明如果是共享锁的话，session2在T2时刻执行<strong style="color: #24292e; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">insert into </strong><span style="color: #24292e; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">test2</span><span class="s1" style="color: #24292e; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">(</span><strong style="color: #24292e; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">code</strong><span class="s1" style="color: #24292e; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">,</span><span style="color: #24292e; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';"> other</span><span class="s1" style="color: #24292e; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">)</span> <strong style="color: #24292e; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">values </strong><span class="s1" style="color: #24292e; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">(</span><span style="color: #24292e; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">3</span><span class="s1" style="color: #24292e; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">,</span><span style="color: #24292e; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';"> 33</span><span class="s1" style="color: #24292e; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">)语句的话，一定会立刻包duplicate error而不会阻塞。但是事实上如果Session2在T2时刻执行这句sql，会一直阻塞，进一步说明code=3加的是排它锁。另外需要注意的是，其实我目前只能非常确定<span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">code = 3</span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">是有排它锁，但是(</span><span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">1</span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">,</span><span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">3</span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">)上面，到底是</span><span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">S gap lock </span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">还是</span><span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">X gap lock</span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">无法确定，不过无论是S还是X,不影响后续的解释。</span></span></span></span></span></span></span><span class="s1"><span style="color: #ff0000;"><span style="color: #000000;">）</span></span></span></li>
<li class="p1">
<p class="p1"><span class="s1">在</span>T2 <span class="s1">完成时，同理也会在<span style="color: #ff0000;">（</span></span><span style="color: #ff0000;">3<span class="s1">，</span>5<span class="s1">]这个区间上</span>X next-key lock</span> <span class="s1">(在上面的截图中也可以看到插入</span>code=5<span class="s1">后，正在插入</span>code=2<span class="s1">的时候，写着</span>HOLD the lock hex 80000005<span class="s1">)</span></p>
</li>
<li class="p1">
<p class="p1">当<span class="s1">T1和T2</span>执行完成之后，我们可以看到(<span class="s1">1</span>,<span class="s1">3</span>] 和（<span class="s1">3</span>,<span class="s1">5</span>]分别被<span class="s1">Session1</span>和<span class="s1">Session2</span>锁定，<span class="s1">T3</span>时候，<span style="color: #ff0000;"><span class="s1">Session1</span>尝试插入<span class="s1">code=4</span>, 由于在插入前会加插入意向锁</span>，（对于插入意向锁的锁的范围，我目前尚无法确认在3~5的区间内加锁的时候，左右临界的开合问题）但是很明显，插入意向锁一定和(<span class="s1"><strong>3</strong></span>,<span class="s1"><strong>5</strong></span>]区间的<span class="s1"><strong>next-key lock</strong></span>有重合，所以会出现在Session1执行<span class="s1"><strong>T3</strong></span>的时候，语句被阻塞了，它在等待<span class="s1"><strong>Session2</strong></span>释放(<span class="s1"><strong>3</strong></span>,<span class="s1"><strong>5</strong></span>]这个区间的X<span class="s1"><strong> next_key</strong> lock 。可以参考下图&mdash;&mdash;一个非常详细的锁兼容矩阵，理解阻塞原因（<a href="https://my.oschina.net/actiontechoss/blog/3068976">兼容矩阵图链接</a></span><span class="s1">）。</span><span class="s1"><img src="https://ata2-img.cn-hangzhou.oss-pub.aliyun-inc.com/b90769bc02aa178901669fa1ad437800.png" /></span></p>
</li>
<li class="p1">
<p class="p1">同理，在<span class="s1">T4时刻Session2执行插入语句</span>的时候，由于（<span class="s1">1</span>,<span class="s1">3</span>]被阻塞了，但是插入的时候又要请求<span class="s1">1~3</span>这个区间的插入意向锁，等待<span class="s1">Session1</span>释放X<span class="s1"> next-key lock</span>。于是乎死锁发生，Session2被回滚。</p>
</li>
</ol>
<p class="p1">至此：死锁的现象可以顺利的解释通。（当然，这里还有一个疑惑不是很明白，当出现唯一冲突的时候为什么要加Next-Key Lock。有知道原因的小伙伴可以告诉我）</p>
<h3>问题的拓展：</h3>
<ol>
<li>如果将insert on duplicate key update换成insert ignore语句，是否可以避免死锁的发生呢？答案是：否定的。其实原理都是一样的。如果我们将上述复现中的insert on duplicate key update换成insert ignore，同样会在T4时刻出现死锁。</li>
<li>同样，update和insert on duplicate key update组合也可以构造出死锁的出现。数据库中表结构不变，数据初始化为(<span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">1</span><span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">,</span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">1</span><span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">,</span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">1</span><span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">),(</span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">3</span><span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">,</span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">3</span><span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">,</span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">3</span><span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">),(</span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">5</span><span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">,</span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">5</span><span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">,</span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">5</span><span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">)</span> <span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">分别对应</span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">id</span><span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">,</span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';"> code</span><span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">,</span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">other</span><span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">,</span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';"> id</span><span class="s1" style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">是</span><span style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';">pk.</span></li>
</ol>
<table style="border-collapse: collapse; width: 0%; height: 481px;" border="1">
<tbody>
<tr style="height: 34px;">
<td style="width: 14.6011%; text-align: center; height: 34px;"><strong>Time</strong></td>
<td style="width: 42.8063%; height: 34px; text-align: center;"><strong>Session1</strong></td>
<td style="width: 42.5926%; height: 34px; text-align: center;"><strong>Session2</strong></td>
</tr>
<tr style="height: 34px;">
<td style="width: 14.6011%; height: 76px; text-align: center;"><strong>T1</strong></td>
<td style="width: 42.8063%; height: 76px; text-align: left;">
<p><strong>start transaction<span class="s1">;</span></strong></p>
<p class="p1"><strong>update </strong>test2 <strong>set </strong>other=1 <strong>where </strong>id=3<span class="s1">;</span></p>
</td>
<td style="width: 42.5926%; height: 76px; text-align: center;"> </td>
</tr>
<tr style="height: 34px;">
<td style="width: 14.6011%; height: 92px; text-align: center;">T2</td>
<td style="width: 42.8063%; height: 92px;"> </td>
<td style="width: 42.5926%; height: 92px;">
<p class="p1"><strong>start transaction<span class="s1">;</span></strong></p>
<p class="p1"><strong>insert into </strong>test2<span class="s1">(</span><strong>code</strong><span class="s1">,</span> other<span class="s1">)</span> <strong>values </strong><span class="s1">(</span>5<span class="s1">,</span> 55<span class="s1">)</span> <strong>on duplicate key update </strong>other = <em>VALUES</em><span class="s1">(</span>other<span class="s1">);</span></p>
</td>
</tr>
<tr style="height: 34px;">
<td style="width: 14.6011%; height: 129px; text-align: center;">
<p class="p1"><strong>T3</strong></p>
</td>
<td style="width: 42.8063%; height: 129px;">
<p class="p1"><strong>update </strong>test2 <strong>set </strong>other=1 <strong>where </strong>id=5<span class="s1">;</span></p>
<p class="p1"><em><strong><span class="s1" style="color: #ff0000;">现象：一直阻塞</span></strong></em></p>
</td>
<td style="width: 42.5926%; height: 129px;"> </td>
</tr>
<tr style="height: 150px;">
<td style="width: 14.6011%; height: 150px; text-align: center;">T4</td>
<td style="width: 42.8063%; height: 150px;"> </td>
<td style="width: 42.5926%; height: 150px;">
<p class="p1"><strong>insert into </strong>test2<span class="s1">(</span><strong>code</strong><span class="s1">,</span> other<span class="s1">)</span> <strong>values </strong><span class="s1">(</span>3<span class="s1">,</span> 33<span class="s1">)</span> <strong>on duplicate key update </strong>other = <em>VALUES</em><span class="s1">(</span>other<span class="s1">);</span></p>
<p class="p1"><em><strong><span style="color: #ff0000;">现象：出现死锁，回滚 (<span class="s1">2</span>,<span class="s1">2</span>插入失败)</span></strong></em></p>
</td>
</tr>
</tbody>
</table>
<h3>总结：</h3>
<p>    说了这么多，死锁的原因找到了，解决的办法其实比较简单。</p>
<ol>
<li>将批量insert on duplicate key update,拆分成多个语句。保证<span style="color: #ff0000;">一次事务中</span>不要插入过多值，将多个数据，变成多个sql，执行插入。可以有效的减少死锁命中的发生。</li>
<li>重试：死锁不可怕，当出现死锁发生时，多执行重试操作可以有效保证插入成功，更新不丢失。</li>
</ol>
<p>参考文章：</p>
<ol>
<li><span style="font-size: 10px;"><a href="https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-insert-intention-locks">https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-insert-intention-locks</a></span></li>
<li><span style="font-size: 10px;"><a href="https://dba.stackexchange.com/questions/237549/gap-locking-in-read-committed-isolation-level-in-mysql">https://dba.stackexchange.com/questions/237549/gap-locking-in-read-committed-isolation-level-in-mysql</a></span></li>
<li><span style="font-size: 10px;"><a href="https://www.cnblogs.com/crazylqy/p/7773492.html">https://www.cnblogs.com/crazylqy/p/7773492.html</a></span></li>
<li><span style="font-size: 10px;"><a href="https://www.jianshu.com/p/dca007208a58">https://www.jianshu.com/p/dca007208a58</a></span></li>
<li><span style="font-size: 10px;"><a href="https://my.oschina.net/actiontechoss/blog/3068976">https://my.oschina.net/actiontechoss/blog/3068976</a></span></li>
<li><span style="font-size: 10px;"><a href="https://www.aneasystone.com/archives/2017/12/solving-dead-locks-three.html">https://www.aneasystone.com/archives/2017/12/solving-dead-locks-three.html</a></span></li>
<li><span style="font-size: 10px;"><a href="https://www.cnblogs.com/zhoujinyi/p/3435982.html">https://www.cnblogs.com/zhoujinyi/p/3435982.html</a></span></li>
<li><span style="font-size: 10px;">http://thushw.blogspot.com/2010/11/mysql-deadlocks-with-concurrent-inserts.html</span></li>
</ol>
<p> </p>
<p> </p>
<p> </p>
<p> </p>
