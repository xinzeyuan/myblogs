# URL
  - https://www.jianshu.com/p/2530d1185778
<h2>01  开局一张图</h2>
<p>这张图是<code>重点</code>！！！咱要先对MySQL有一个宏观的了解，知道他的执行流程。</p>
<p>一条SQL语句过来的流程是什么样的？那就follow me。哈哈哈哈，皮一下很开心。</p>
<ul>
<li><p>1.当客户端连接到MySQL服务器时，服务器对其进行认证。可以通过用户名与密码认证，也可以通过SSL证书进行认证。登录认证后，服务器还会验证客户端是否有执行某个查询的操作权限。</p></li>
<li><p>2.在正式查询之前，服务器会检查查询缓存，如果能找到对应的查询，则不必进行查询解析，优化，执行等过程，直接返回缓存中的结果集。</p></li>
<li><p>3.MySQL的解析器会根据查询语句，构造出一个解析树，主要用于根据语法规则来验证语句是否正确，比如SQL的关键字是否正确，关键字的顺序是否正确。</p></li>
</ul>
<p>而预处理器主要是进一步校验，比如表名，字段名是否正确等</p>
<ul>
<li><p>4.查询优化器将解析树转化为查询计划，一般情况下，一条查询可以有很多种执行方式，最终返回相同的结果，优化器就是根据<code>成本</code>找到这其中最优的执行计划</p></li>
<li><p>5.执行计划调用查询执行引擎，而查询引擎通过一系列API接口查询到数据</p></li>
<li><p>6.得到数据之后，在返回给客户端的同时，会将数据存在查询缓存中</p></li>
</ul>
![在这里插入图片描述](./18375227-1c8d30f7eebed5dd.jpg)

<h2>02  查询缓存</h2>
<p>我们先通过<code>show variables like '%query_cache%'</code>来看一下默认的数据库配置，此为本地数据库的配置。</p>
<div class="image-package">
<div class="image-container" style="max-width: 391px; max-height: 285px;">
<div class="image-container-fill" style="padding-bottom: 72.89%;"></div>
<div class="image-view" data-width="391" data-height="285"><img data-original-src="//upload-images.jianshu.io/upload_images/18375227-42c0d6317f9323e9.png" data-original-width="391" data-original-height="285" data-original-format="image/png" data-original-filesize="87827"></div>
</div>
<div class="image-caption">image.png</div>
</div>
<h3>2.1  概念</h3>
<p>have_query_cache:当前的MYSQL版本是否支持“查询缓存”功能。</p>
<p>query_cache_limit:MySQL能够缓存的最大查询结果，查询结果大于该值时不会被缓存。默认值是1048576(1MB)</p>
<p>query_cache_min_res_unit:查询缓存分配的最小块（字节）。默认值是4096（4KB）。当查询进行时，MySQL把查询结果保存在query cache，但是如果保存的结果比较大，超过了query_cache_min_res_unit的值，这时候MySQL将一边检索结果，一边进行保存结果。他保存结果也是按默认大小先分配一块空间，如果不够，又要申请新的空间给他。如果查询结果比较小，默认的query_cache_min_res_unit可能造成大量的内存碎片，如果查询结果比较大，默认的query_cache_min_res_unit又不够，导致一直分配块空间，所以可以根据实际需求，调节query_cache_min_res_unit的大小。</p>
<p><code>注：如果上面说的内容有点弯弯绕，那举个现实生活中的例子，比如咱现在要给运动员送水，默认的是500ml的瓶子，如果过来的是少年运动员，可能500ml太大了，他们喝不完，造成了浪费，那我们就可以选择300ml的瓶子，如果过来的是成年运动员，可能500ml不够，那他们一瓶喝完了，又开一瓶，直接不渴为止。那么那样开瓶子也要时间，我们就可以选择1000ml的瓶子。</code></p>
<p>query_cache_size:为缓存查询结果分配的总内存。</p>
<p>query_cache_type:默认为on，可以缓存除了以select sql_no_cache开头的所有查询结果。</p>
<p>query_cache_wlock_invalidate:如果该表被锁住，是否返回缓存中的数据，默认是关闭的。</p>
<h3>2.2  原理</h3>
<p>MYSQL的查询缓存实质上是缓存SQL的hash值和该SQL的查询结果，如果运行相同的SQL,服务器直接从缓存中去掉结果，而不再去解析，优化，寻找最低成本的执行计划等一系列操作，大大提升了查询速度。</p>
<p>但是万事有利也有弊。</p>
<ul>
<li>第一个弊端就是如果表的数据有一条发生变化，那么缓存好的结果将全部不再有效。这对于频繁更新的表，查询缓存是不适合的。</li>
</ul>
<p><code>比如一张表里面只有两个字段，分别是id和name，数据有一条为1，张三。我使用select * from 表名 where name=“张三”来进行查询，MySQL发现查询缓存中没有此数据，会进行一系列的解析，优化等操作进行数据的查询，查询结束之后将该SQL的hash和查询结果缓存起来，并将查询结果返回给客户端。但是这个时候我有新增了一条数据2，张三。如果我还用相同的SQL来执行，他会根据该SQL的hash值去查询缓存中，那么结果就错了。所以MySQL对于数据有变化的表来说，会直接清空关于该表的所有缓存。这样其实是效率是很差的。</code></p>
<ul>
<li>第二个弊端就是缓存机制是通过对SQL的hash，得出的值为key，查询结果为value来存放的，那么就意味着SQL必须完完全全一模一样，否则就命不中缓存。</li>
</ul>
<p><code>我们都知道hash值的规则，就算很小的查询，哈希出来的结果差距是很多的，所以select * from 表名 where name=“张三”和SELECT * FROM 表名 WHERE NAME=“张三”和select * from 表名 where name = “张三”，三个SQL哈希出来的值是不一样的，大小写和空格影响了他们，所以并不能命中缓存，但其实他们搜索结果是完全一样的。</code></p>
<h3>2.3  生产如何设置MySQL Query Cache</h3>
<p>先来看线上参数：</p>
<div class="image-package">
<div class="image-container" style="max-width: 352px; max-height: 240px;">
<div class="image-container-fill" style="padding-bottom: 68.17999999999999%;"></div>
<div class="image-view" data-width="352" data-height="240"><img data-original-src="//upload-images.jianshu.io/upload_images/18375227-ab994b6bfd4d1e61.png" data-original-width="352" data-original-height="240" data-original-format="image/png" data-original-filesize="83326"></div>
</div>
<div class="image-caption">image.png</div>
</div>
<p>我们发现将query_cache_type设置为OFF，其实网上资料和各大云厂商提供的云服务器都是将这个功能关闭的，从上面的原理来看，在一般情况下，<code>他的弊端大于优点</code>。</p>
<h2>03  索引</h2>
<h3>3.1  例子</h3>
<p>创建一个名为user的表，其包括id，name，age，sex等字段信息。此外，id为主键聚簇索引，idx_name为非聚簇索引。</p>
<pre><code>CREATE TABLE `user` (
  `id` varchar(10) NOT NULL DEFAULT '',
  `name` varchar(10) DEFAULT NULL,
  `age` int(11) DEFAULT NULL,
  `sex` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_name` (`name`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
</code></pre>
<p>我们将其设置10条数据，便于下面的索引的理解。</p>
<pre><code>INSERT INTO `user` VALUES ('1', 'andy', '20', '女');
INSERT INTO `user` VALUES ('10', 'baby', '12', '女');
INSERT INTO `user` VALUES ('2', 'kat', '12', '女');
INSERT INTO `user` VALUES ('3', 'lili', '20', '男');
INSERT INTO `user` VALUES ('4', 'lucy', '22', '女');
INSERT INTO `user` VALUES ('5', 'bill', '20', '男');
INSERT INTO `user` VALUES ('6', 'zoe', '20', '男');
INSERT INTO `user` VALUES ('7', 'hay', '20', '女');
INSERT INTO `user` VALUES ('8', 'tony', '20', '男');
INSERT INTO `user` VALUES ('9', 'rose', '21', '男');
</code></pre>
<h3>3.2  聚簇索引（主键索引）</h3>
<p>先来一张图镇楼，接下来就是看图说话。</p>
<div class="image-package">
<div class="image-container" style="max-width: 596px; max-height: 427px;">
<div class="image-container-fill" style="padding-bottom: 71.64%;"></div>
<div class="image-view" data-width="596" data-height="427"><img data-original-src="//upload-images.jianshu.io/upload_images/18375227-1c8d30f7eebed5dd.png" data-original-width="596" data-original-height="427" data-original-format="image/png" data-original-filesize="126958"></div>
</div>
<div class="image-caption">image.png</div>
</div>
<p>他包含两个特点：</p>
<p>1.使用记录主键值的大小来进行记录和页的排序。</p>
<p>页内的记录是按照主键的大小顺序排成一个单项链表。</p>
<p>各个存放用户记录的页也是根据页中用户记录的主键大小顺序排成一个双向链表。</p>
<p>2.叶子节点存储的是<code>完整的用户记录</code>。</p>
<pre><code>注：聚簇索引不需要我们显示的创建，他是由InnoDB存储引擎自动为我们创建的。如果没有主键，其也会默认创建一个。复制代码
</code></pre>
<h3>3.3  非聚簇索引（二级索引）</h3>
<p>上面的聚簇索引只能在搜索条件是主键时才能发挥作用，因为聚簇索引可以根据主键进行排序的。如果搜索条件是name，在刚才的聚簇索引上，我们可能遍历，挨个找到符合条件的记录，但是，这样真的是太蠢了，MySQL不会这样做的。</p>
<p>如果我们想让搜索条件是name的时候，也能使用索引，那可以多创建一个基于name的二叉树。如下图。</p>
<div class="image-package">
<div class="image-container" style="max-width: 619px; max-height: 407px;">
<div class="image-container-fill" style="padding-bottom: 65.75%;"></div>
<div class="image-view" data-width="619" data-height="407"><img data-original-src="//upload-images.jianshu.io/upload_images/18375227-c5414349d1c19c5c.png" data-original-width="619" data-original-height="407" data-original-format="image/png" data-original-filesize="81211"></div>
</div>
<div class="image-caption">image.png</div>
</div>
<p>他与聚簇索引的不同：</p>
<p>1.叶子节点内部使用name字段排序，叶子节点之间也是使用name字段排序。</p>
<p>2.叶子节点不再是完整的数据记录，而是name和主键值。</p>
<p><code>为什么不再是完整信息？</code></p>
<p>MySQL只让聚簇索引的叶子节点存放完整的记录信息，因为如果有好几个非聚簇索引，他们的叶子节点也存放完整的记录绩效，那就不浪费空间啦。</p>
<p><code>如果我搜索条件是基于name，需要查询所有字段的信息，那查询过程是啥？</code></p>
<p>1.根据查询条件，采用name的非聚簇索引，先定位到该非聚簇索引某些记录行。</p>
<p>2.根据记录行找到相应的id，再根据id到聚簇索引中找到相关记录。这个过程叫做<code>回``表</code>。</p>
<h3>3.4  联合索引</h3>
<p>图就不画了，简单来说，如果name和age组成一个联合索引，那么先按name排序，如果name一样，就按age排序。</p>
<h3>3.5  一些原则</h3>
<p>1.最左前缀原则。一个联合索引（a,b,c）,如果有一个查询条件有a，有b，那么他则走索引，如果有一个查询条件没有a，那么他则不走索引。</p>
<p>2.使用唯一索引。具有多个重复值的列，其索引效果最差。例如，存放姓名的列具有不同值，很容易区分每行。而用来记录性别的列，只含有“男”，“女”，不管搜索哪个值，都会得出大约一半的行，这样的索引对性能的提升不够高。</p>
<p>3.不要过度索引。每个额外的索引都要占用额外的磁盘空间，并降低写操作的性能。在修改表的内容时，索引必须进行更新，有时可能需要重构，因此，索引越多，所花的时间越长。</p>
<p>4、索引列不能参与计算，保持列“干净”，比如from_unixtime(create_time) = ’2014-05-29’就不能使用到索引，原因很简单，b+树中存的都是数据表中的字段值，但进行检索时，需要把所有元素都应用函数才能比较，显然成本太大。所以语句应该写成create_time = unix_timestamp(’2014-05-29’);</p>
<p>5.一定要设置一个主键。前面聚簇索引说到如果不指定主键，InnoDB会自动为其指定主键，这个我们是看不见的。反正都要生成一个主键的，还不如我们设置，以后在某些搜索条件时还能用到主键的聚簇索引。</p>
<p>6.主键推荐用自增id，而不是uuid。上面的聚簇索引说到每页数据都是排序的，并且页之间也是排序的，如果是uuid，那么其肯定是随机的，其可能从中间插入，导致页的分裂，产生很多表碎片。如果是自增的，那么其有从小到大自增的，有顺序，那么在插入的时候就添加到当前索引的后续位置。当一页写满，就会自动开辟一个新的页。</p>
<pre><code>注：如果自增id用完了，那将字段类型改为bigint，就算每秒1万条数据，跑100年，也没达到bigint的最大值。复制代码
</code></pre>
<h3>3.6  万年面试题（为什么索引用B+树）</h3>
<p>1、 B+树的磁盘读写代价更低：B+树的内部节点并没有指向关键字具体信息的指针，因此其内部节点相对B树更小，如果把所有同一内部节点的关键字存放在同一盘块中，那么盘块所能容纳的关键字数量也越多，一次性读入内存的需要查找的关键字也就越多，相对<code>IO读写次数就降低</code>了。</p>
<p>2、由于B+树的数据都存储在叶子结点中，分支结点均为索引，方便扫库，只需要扫一遍叶子结点即可，但是B树因为其分支结点同样存储着数据，我们要找到具体的数据，需要进行一次中序遍历按序来扫，所以B+树更加适合在<code>区间查询</code>的情况，所以通常B+树用于数据库索引。</p>
<h2>04  优化器</h2>
<p>在开篇的图里面，我们知道了SQL语句从客户端经由网络协议到查询缓存，如果没有命中缓存，再经过解析工作，得到准确的SQL，现在就来到了我们这模块说的优化器。</p>
<p>首先，我们知道每一条SQL都有不同的执行方法，要不通过索引，要不通过全表扫描的方式。</p>
<p>那么问题就来了，MySQL是如何选择时间最短，占用内存最小的执行方法呢？</p>
<h3>4.1  什么是成本？</h3>
<p>1.I/O成本。数据存储在硬盘上，我们想要进行某个操作需要将其加载到内存中，这个过程的时间被称为I/O成本。默认是1。</p>
<p>2.CPU成本。在内存对结果集进行排序的时间被称为CPU成本。默认是0.2。</p>
<h3>4.2  单表查询的成本</h3>
<p>先来建一个用户表dev_user，里面包括主键id，用户名username，密码password，外键user_info_id，状态status，外键main_station_id，是否外网访问visit，这七个字段。索引有两个，一个是主键的聚簇索引，另一个是显式添加的以username为字段的唯一索引uname_unique。</p>
<div class="image-package">
<div class="image-container" style="max-width: 700px; max-height: 80px;">
<div class="image-container-fill" style="padding-bottom: 10.879999999999999%;"></div>
<div class="image-view" data-width="735" data-height="80"><img data-original-src="//upload-images.jianshu.io/upload_images/18375227-91e6da824076e362.png" data-original-width="735" data-original-height="80" data-original-format="image/png" data-original-filesize="39454"></div>
</div>
<div class="image-caption">image.png</div>
</div>
<p>如果搜索条件是select * from dev_user where username='XXX'，那么MySQL是如何选择相关索引呢？</p>
<p>1.使用所有可能用到的索引</p>
<p>我们可以看到搜索条件username，所以可能走uname_unique索引。也可以做聚簇索引，也就是全表扫描。</p>
<p>2.计算全表扫描代价</p>
<p>我们通过<code>show table status like ‘dev_user’</code>命令知道<code>rows</code>和<code>data_length</code>字段，如下图。</p>
<div class="image-package">
<div class="image-container" style="max-width: 700px; max-height: 155px;">
<div class="image-container-fill" style="padding-bottom: 12.11%;"></div>
<div class="image-view" data-width="1280" data-height="155"><img data-original-src="//upload-images.jianshu.io/upload_images/18375227-ad55c097cb502d93.png" data-original-width="1280" data-original-height="155" data-original-format="image/png" data-original-filesize="111939"></div>
</div>
<div class="image-caption">image.png</div>
</div>
<p>rows：表示表中的记录条数，但是这个数据不准确，是个估计值。</p>
<p>data_length:表示表占用的存储空间字节数。</p>
<p><code>data_length=聚簇索引的页面数量X每个页面的大小</code></p>
<p>反推出页面数量=1589248÷16÷1024=97</p>
<p>I/O成本：97X1=97</p>
<p>CPU成本：6141X0.2=1228</p>
<p>总成本：97+1228=1325</p>
<p>3.计算使用不同索引执行查询的代价</p>
<p>因为要查询出满足条件的所有字段信息，所以要考虑回表成本。</p>
<p>I/O成本=1+1X1=2(范围区间的数量+预计二级记录索引条数)</p>
<p>CPU成本=1X0.2+1X0.2=0.4(读取二级索引的成本+回表聚簇索引的成本)</p>
<p>总成本=I/O成本+CPU成本=2.4</p>
<p>4.对比各种执行方案的代价，找出成本最低的那个</p>
<p>上面两个数字一对比，成本是采用uname_unique索引成本最低。</p>
<h3>4.3  多表查询的成本</h3>
<p>对于两表连接查询来说，他的查询成本由下面两个部分构成：</p>
<ul>
<li>单次查询驱动表的成本</li>
<li>多次查询被驱动表的成本（具体查询多次取决于对驱动表查询的结果集有多少个记录）</li>
</ul>
<h3>4.4  index dive</h3>
<p>如果前面的搜索条件不是等值，而是区间，如<code>select * from dev_user where username&gt;'admin' and username&lt;'test'</code>这个时候我们是无法看出需要回表的数量。</p>
<p>步骤1：先根据username&gt;'admin'这个条件找到第一条记录，称为<code>区间最左记录</code>。</p>
<p>步骤2：再根据username&lt;'test'这个条件找到最后一条记录，称为<code>区间最右记录</code>。</p>
<p>步骤3：如果区间最左记录和区间最右记录相差不是很远，可以准确统计出需要回表的数量。如果相差很远，就先计算10页有多少条记录，再乘以页面数量，最终模糊统计出来。</p>
<h2>05  Explain</h2>
<h3>5.1  产品来索命</h3>
<p>产品：为什么这个页面出来这么慢？</p>
<p>开发：因为你查的数据多呗，他就是这么慢</p>
<p>产品：我不管，我要这个页面快点，你这样，客户怎么用啊</p>
<p>开发：。。。。。。。你行你来</p>
<div class="image-package">
<div class="image-container" style="max-width: 500px; max-height: 449px;">
<div class="image-container-fill" style="padding-bottom: 89.8%;"></div>
<div class="image-view" data-width="500" data-height="449"><img data-original-src="//upload-images.jianshu.io/upload_images/18375227-7f6e414bed8e9694.png" data-original-width="500" data-original-height="449" data-original-format="image/png" data-original-filesize="306785"></div>
</div>
<div class="image-caption">image.png</div>
</div>
<p>哈哈哈哈，不瞎BB啦，如果有些SQL贼慢，我们需要知道他有没有走索引，走了哪个索引，这个时候我就需要通过explain关键字来深入了解MySQL内部是如何执行的。</p>
<div class="image-package">
<div class="image-container" style="max-width: 700px; max-height: 175px;">
<div class="image-container-fill" style="padding-bottom: 22.18%;"></div>
<div class="image-view" data-width="789" data-height="175"><img data-original-src="//upload-images.jianshu.io/upload_images/18375227-12dc862c5053cd89.png" data-original-width="789" data-original-height="175" data-original-format="image/png" data-original-filesize="84358"></div>
</div>
<div class="image-caption">image.png</div>
</div>
<h3>5.2  id</h3>
<p>一般来说一个select一个唯一id，如果是子查询，就有两个select，id是不一样的，但是凡事有例外，有些子查询的，他们id是一样的。</p>
<br>
<div class="image-package">
<div class="image-container" style="max-width: 700px; max-height: 153px;">
<div class="image-container-fill" style="padding-bottom: 20.51%;"></div>
<div class="image-view" data-width="746" data-height="153"><img data-original-src="//upload-images.jianshu.io/upload_images/18375227-65d384dbb5af5c09.png" data-original-width="746" data-original-height="153" data-original-format="image/png" data-original-filesize="93904"></div>
</div>
<div class="image-caption">image.png</div>
</div>
<p>这是为什么呢？</p>
<p>那是因为MySQL在进行优化的时候已经将子查询改成了连接查询，而连接查询的id是一样的。</p>
<h3>5.3  select_type</h3>
<ul>
<li>simple：不包括union和子查询的查询都算simple类型。</li>
<li>primary：包括union，union all，其中最左边的查询即为primary。</li>
<li>union：包括union，union all，除了最左边的查询，其他的查询类型都为union。</li>
</ul>
<h3>5.4 table</h3>
<p>显示这一行是关于哪张表的。</p>
<h3>5.5 type：访问方法</h3>
<ul>
<li>ref：普通二级索引与常量进行等值匹配</li>
<li>ref_or_null：普通二级索引与常量进行等值匹配，该索引可能是null</li>
<li>const：主键或唯一二级索引列与常量进行等值匹配</li>
<li>range：范围区间的查询</li>
<li>all：全表扫描</li>
</ul>
<h3>5.6 possible_keys</h3>
<p>对某表进行单表查询时可能用到的索引</p>
<h3>5.7 key</h3>
<p>经过查询优化器计算不同索引的成本，最终选择成本最低的索引</p>
<h3>5.8 rows</h3>
<ul>
<li>如果使用全表扫描，那么rows就代表需要扫描的行数</li>
<li>如果使用索引，那么rows就代表预计扫描的行数</li>
</ul>
<h3>5.9 filtered</h3>
<ul>
<li>如果全表扫描，那么filtered就代表满足搜索条件的记录的满分比</li>
<li>如果是索引，那么filtered就代表除去索引对应的搜索，其他搜索条件的百分比</li>
</ul>
<h2>06 redo日志（物理日志）</h2>
<p>InnoDB存储引擎是以页为单位来管理存储空间的，我们进行的增删改查操作都是将页的数据加载到内存中，然后进行操作，再将数据刷回到硬盘上。</p>
<p>那么问题就来了，如果我要给张三转账100块钱，事务已经提交了，这个时候InnoDB把数据加载到内存中，这个时候还没来得及刷入硬盘，突然停电了，数据库崩了。重启之后，发现我的钱没有转成功，这不是尴尬了吗？</p>
<p>解决方法很明显，我们在硬盘加载到内存之后，进行一系列操作，一顿操作猛如虎，还未刷新到硬盘之前，先记录下，在XXX位置我的记录中金额减100，在XXX位置张三的记录中金额加100，然后再进行增删改查操作，最后刷入硬盘。如果未刷入硬盘，在重启之后，先加载之前的记录，那么数据就回来了。</p>
<p>这个记录就叫做重做日志，即redo日志。他的目的是想让已经提交的事务对数据的修改是永久的，就算他重启，数据也能恢复出来。</p>
<h3>6.1  log buffer（日志缓冲区）</h3>
<p>为了解决磁盘速度过慢的问题，redo日志不能直接写入磁盘，咱先整一大片连续的内存空间给他放数据。这一大片内存就叫做日志缓冲区，即log buffer。到了合适的时候，再刷入硬盘。至于什么时候是合适的，这个下一章节说。</p>
<p>我们可以通过<code>show VARIABLES like 'innodb_log_buffer_size'</code>命令来查看当前的日志缓存大小，下图为线上的大小。</p>
<div class="image-package">
<div class="image-container" style="max-width: 407px; max-height: 135px;">
<div class="image-container-fill" style="padding-bottom: 33.17%;"></div>
<div class="image-view" data-width="407" data-height="135"><img data-original-src="//upload-images.jianshu.io/upload_images/18375227-61ec519d9b02b091.png" data-original-width="407" data-original-height="135" data-original-format="image/png" data-original-filesize="44556"></div>
</div>
<div class="image-caption">image.png</div>
</div>
<h3>6.2 redo日志刷盘时机</h3>
<p>由于redo日志一直都是增长的，且内存空间有限，数据也不能一直待在缓存中， 我们需要将其刷新至硬盘上。</p>
<p>那什么时候刷新到硬盘呢？</p>
<ul>
<li>log buffer空间不足。上面有指定缓冲区的内存大小，MySQL认为日志量已经占了 总容量的一半左右，就需要将这些日志刷新到磁盘上。</li>
<li>事务提交时。我们使用redo日志的目的就是将他未刷新到磁盘的记录保存起来，防止 丢失，如果数据提交了，我们是可以不把数据提交到磁盘的，但为了保证持久性，必须 把修改这些页面的redo日志刷新到磁盘。</li>
<li>后台线程不同的刷新 后台有一个线程，大概每秒都会将log buffer里面的redo日志刷新到硬盘上。</li>
<li>checkpoint 下下小节讲</li>
</ul>
<h3>6.3 redo日志文件组</h3>
<p>我们可以通过<code>show variables like 'datadir'</code>命令找到相关目录，底下有两个文件， 分别是ib_logfile0和ib_logfile1,如下图所示。</p>
<div class="image-package">
<div class="image-container" style="max-width: 483px; max-height: 153px;">
<div class="image-container-fill" style="padding-bottom: 31.680000000000003%;"></div>
<div class="image-view" data-width="483" data-height="153"><img data-original-src="//upload-images.jianshu.io/upload_images/18375227-31dd34338d350428.png" data-original-width="483" data-original-height="153" data-original-format="image/png" data-original-filesize="44921"></div>
</div>
<div class="image-caption">image.png</div>
</div>
<div class="image-package">
<div class="image-container" style="max-width: 623px; max-height: 431px;">
<div class="image-container-fill" style="padding-bottom: 69.17999999999999%;"></div>
<div class="image-view" data-width="623" data-height="431"><img data-original-src="//upload-images.jianshu.io/upload_images/18375227-8116696aedd60988.png" data-original-width="623" data-original-height="431" data-original-format="image/png" data-original-filesize="260850"></div>
</div>
<div class="image-caption">image.png</div>
</div>
<p>我们将缓冲区log buffer里面的redo日志刷新到这个两个文件里面，他们写入的方式 是循环写入的，先写ib_logfile0,再写ib_logfile1,等ib_logfile1写满了，再写ib_logfile0。 那这样就会存在一个问题，如果ib_logfile1写满了，再写ib_logfile0，之前ib_logfile0的内容 不就被覆盖而丢失了吗？ 这就是checkpoint的工作啦。</p>
<h3>6.4  checkpoint</h3>
<p>redo日志是为了系统崩溃后恢复脏页用的，如果这个脏页可以被刷新到磁盘上，那么 他就可以功成身退，被覆盖也就没事啦。</p>
<p>冲突补习</p>
<p>从系统运行开始，就不断的修改页面，会不断的生成redo日志。redo日志是不断 递增的，MySQL为其取了一个名字日志序列号Log Sequence Number，简称lsn。 他的初始化的值为8704，用来记录当前一共生成了多少redo日志。</p>
<p>redo日志是先写入log buffer，之后才会被刷新到磁盘的redo日志文件。MySQL为其 取了一个名字flush_to_disk_lsn。用来说明缓存区中有多少的脏页数据被刷新到磁盘上啦。 他的初始值和lsn一样，后面的差距就有了。</p>
<p>做一次checkpoint分为两步</p>
<ul>
<li>计算当前系统可以被覆盖的redo日志对应的lsn最大值是多少。redo日志可以被覆盖， 意味着他对应的脏页被刷新到磁盘上，只要我们计算出当前系统中最早被修改的oldest_modification, 只要系统中lsn小于该节点的oldest_modification值磁盘的redo日志都是可以被覆盖的。</li>
<li>将lsn过程中的一些数据统计。</li>
</ul>
<h2>07 undo日志（这部分不是很明白，所以大概说了）</h2>
<h3>7.1 基本概念</h3>
<p>undo log有两个作用：提供回滚和多个行版本控制(<code>MVCC</code>)。</p>
<p>undo log和redo log记录物理日志不一样，它是逻辑日志。可以认为当delete一条记录时，undo log中会记录一条对应的insert记录，反之亦然，当update一条记录时，它记录一条对应相反的update记录。</p>
<p>举个例子:</p>
<p>insert into a(id) values(1);(redo)<br>
这条记录是需要回滚的。<br>
回滚的语句是delete from a where id = 1;(undo)</p>
<p>试想想看。如果没有做insert into a(id) values(1);(redo)<br>
那么delete from a where id = 1;(undo)这句话就没有意义了。</p>
<p>现在看下正确的恢复:<br>
先insert into a(id) values(1);(redo)<br>
然后delete from a where id = 1;(undo)<br>
系统就回到了原先的状态，没有这条记录了</p>
<h3>7.2  存储方式</h3>
<p>是存在段之中。</p>
<h2>08  事务</h2>
<h3>8.1  引言</h3>
<p>事务中有一个隔离性特征，理论上在某个事务对某个数据进行访问时，其他事务应该排序，当该事务提交之后，其他事务才能继续访问这个数据。</p>
<p>但是这样子对性能影响太大，我们既想保持事务的隔离性，又想让服务器在出来多个事务时性能尽量高些，所以只能舍弃一部分隔离性而去性能。</p>
<h3>8.2  事务并发执行的问题</h3>
<ul>
<li>脏写（这个太严重了，任何隔离级别都不允许发生）</li>
</ul>
<p>sessionA：修改了一条数据，回滚掉</p>
<p>sessionB：修改了同一条数据，提交掉</p>
<p>对于sessionB来说，明明数据更新了也提交了事务，不能说自己啥都没干</p>
<ul>
<li>脏读：一个事务读到另一个未提交事务修改的数据</li>
</ul>
<p>session A：查询，得到某条数据</p>
<p>session B：修改某条数据，但是最后回滚掉啦</p>
<p>session A：在sessionB修改某条数据之后，在回滚之前，读取了该条记录</p>
<p>对于session A来说，读到了session回滚之前的脏数据</p>
<ul>
<li>不可重复读：前后多次读取，同一个数据内容不一样</li>
</ul>
<p>session A：查询某条记录<br>
session B : 修改该条记录，并提交事务<br>
session A : 再次查询该条记录，发现前后查询不一致</p>
<ul>
<li>幻读：前后多次读取，数据总量不一致</li>
</ul>
<p>session A：查询表内所有记录<br>
session B : 新增一条记录，并查询表内所有记录<br>
session A : 再次查询该条记录，发现前后查询不一致</p>
<h3>8.3  四种隔离级别</h3>
<p>数据库都有的四种隔离级别，MySQL事务默认的隔离级别是可重复读，而且MySQL可以解决了幻读的问题。</p>
<ul>
<li>未提交读：脏读，不可重复读，幻读都有可能发生</li>
<li>已提交读：不可重复读，幻读可能发生</li>
<li>可重复读：幻读可能发生</li>
<li>可串行化：都不可能发生</li>
</ul>
<p>但凡事没有百分百，emmmm，其实MySQL并没有百分之百解决幻读的问题。</p>
<div class="image-package">
<div class="image-container" style="max-width: 140px; max-height: 150px;">
<div class="image-container-fill" style="padding-bottom: 107.13999999999999%;"></div>
<div class="image-view" data-width="140" data-height="150"><img data-original-src="//upload-images.jianshu.io/upload_images/18375227-4ca72034844a348f" data-original-width="140" data-original-height="150" data-original-format="image/gif" data-original-filesize="140151"></div>
</div>
<div class="image-caption">image</div>
</div>
<p>举个例子：</p>
<p>session A：查询某条不存在的记录。</p>
<p>session B：新增该条不存在的记录，并提交事务。</p>
<p>session A：再次查询该条不存在的记录，是查询不出来的，但是如果我尝试修改该条记录，并提交，其实他是可以修改成功的。</p>
<h3>8.4  MVCC</h3>
<p>版本链：对于该记录的每次更新，都会将值放在一条undo日志中，算是该记录的一个旧版本，随着更新次数的增多，所有版本都会被roll_pointer属性连接成一个链表，即为版本链。</p>
<p>readview：</p>
<ul>
<li>未提交读：因为可以读到未提交事务修改的记录，所以可以直接读取记录的最新版本就行</li>
<li>已提交读：每次读取之前都生成一个readview</li>
<li>可重复读：只有在第一次读取的时候才生成readview</li>
<li>可串行化：InnoDB涉及了加锁的方式来访问记录</li>
</ul>
<blockquote>
<p>作者：学习Java的小姐姐<br>
原文链接：<a href="https://links.jianshu.com/go?to=https%3A%2F%2Fjuejin.im%2Fpost%2F5dfc846051882512327a63b6" target="_blank">https://juejin.im/post/5dfc846051882512327a63b6</a></p>
</blockquote>
</article><div></div><div class="_19DgIp" style="margin-top:24px;margin-bottom:24px"></div></section><section class="sFiE8U" aria-label="google-ad"><ins class="adsbygoogle" style="display:inline-block;width:730px;height:114px" data-ad-client="ca-pub-3077285224019295" data-ad-slot="2979144022"></ins></section><div id="note-page-comment"><div class="lazyload-placeholder"></div></div><section class="ouvJEz"></section></div><aside class="_2OwGUo"><div><div class=""><section class="sFiE8U" aria-label="google-ad"><ins class="adsbygoogle" style="display:inline-block;width:260px;height:173px" data-ad-client="ca-pub-3077285224019295" data-ad-slot="1970871612"></ins></section></div></div></aside></div></div><footer style="width:100%"><div class="_2xr8G8"><div class="_1Jdfvb"><div class="TDvCqd"><textarea class="W2TSX_" placeholder="写下你的评论..."></textarea></div><div class="-pXE92"><div class="_3nj4GN" role="button" tabindex="0" aria-label="添加评论"><i aria-label="ic-reply" class="anticon"><svg width="1em" height="1em" fill="currentColor" aria-hidden="true" focusable="false" class=""><use xlink:href="#ic-reply"></use></svg></i><span>评论<!-- -->4</span></div><div class="_3nj4GN" role="button" tabindex="0" aria-label="给文章点赞"><i aria-label="ic-like" class="anticon"><svg width="1em" height="1em" fill="currentColor" aria-hidden="true" focusable="false" class=""><use xlink:href="#ic-like"></use></svg></i><span>赞<!-- -->112</span></div><div class="_3nj4GN ant-dropdown-trigger" role="button" tabindex="0" aria-label="更多操作"><i aria-label="ic-others" class="anticon"><svg width="1em" height="1em" fill="currentColor" aria-hidden="true" focusable="false" class=""><use xlink:href="#ic-others"></use></svg></i></div></div></div></div><div class="_1LI0En" style="height:0"></div></footer><div class="_3Pnjry"><div class="_1pUUKr"><div class="_2VdqdF" role="button" tabindex="-1" aria-label="给文章点赞"><i aria-label="ic-like" class="anticon"><svg width="1em" height="1em" fill="currentColor" aria-hidden="true" focusable="false" class=""><use xlink:href="#ic-like"></use></svg></i></div><div class="P63n6G"><div class="_2LKTFF"><span class="_1GPnWJ" role="button" tabindex="-1" aria-label="查看点赞列表">112<!-- -->赞</span><span class="_1GPnWJ">113<!-- -->赞</span></div></div></div><div class="_1pUUKr"><div class="_2VdqdF" role="button" tabindex="-1" aria-label="赞赏作者"><i aria-label="ic-shang" class="anticon"><svg width="1em" height="1em" fill="currentColor" aria-hidden="true" focusable="false" class=""><use xlink:href="#ic-shang"></use></svg></i></div><div class="P63n6G" role="button" tabindex="-1" aria-label="查看赞赏列表">赞赏</div></div></div></div><script id="__NEXT_DATA__" type="application/json">{"dataManager":"[]","props":{"isServer":true,"initialState":{"global":{"fontType":"black","locale":"zh-CN","readMode":"day","seoList":[],"done":false,"$ua":{"value":"Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.149 Safari/537.36","isIE11":false,"earlyIE":null,"chrome":"80.0","firefox":null,"safari":null,"isMac":false},"$diamondRate":{"displayable":false,"rate":0},"$modal":{"ContributeModal":false,"RewardListModal":false,"PayModal":false,"CollectionModal":false,"LikeListModal":false,"ReportModal":false,"QRCodeShareModal":false,"BookCatalogModal":false,"RewardModal":false}},"note":{"data":{"is_author":false,"last_updated_at":1578387112,"public_title":"MySQL的万字总结（缓存，索引，Explain，事务，redo日志等）","purchased":false,"liked_note":false,"comments_count":4,"free_content":"\u003cul\u003e\n\u003cli\u003e\u003cstrong\u003e\u003ca href=\"https://www.jianshu.com/p/9ba9a460ed75\" target=\"_blank\"\u003e微服务架构之春招总结：SpringCloud、Docker、Dubbo与SpringBoot\u003c/a\u003e\u003c/strong\u003e\u003c/li\u003e\n\u003cli\u003e\u003cstrong\u003e欢迎大家关注我的专栏：【\u003ca href=\"https://www.jianshu.com/c/8d050bf89c61\" target=\"_blank\"\u003eJava架构技术进阶\u003c/a\u003e】，里面有大量batj面试题集锦，还有各种技术分享，如有好文章也欢迎投稿哦~\u003c/strong\u003e\u003c/li\u003e\n\u003c/ul\u003e\n\u003ch2\u003e01  开局一张图\u003c/h2\u003e\n\u003cp\u003e这张图是\u003ccode\u003e重点\u003c/code\u003e！！！咱要先对MySQL有一个宏观的了解，知道他的执行流程。\u003c/p\u003e\n\u003cp\u003e一条SQL语句过来的流程是什么样的？那就follow me。哈哈哈哈，皮一下很开心。\u003c/p\u003e\n\u003cul\u003e\n\u003cli\u003e\u003cp\u003e1.当客户端连接到MySQL服务器时，服务器对其进行认证。可以通过用户名与密码认证，也可以通过SSL证书进行认证。登录认证后，服务器还会验证客户端是否有执行某个查询的操作权限。\u003c/p\u003e\u003c/li\u003e\n\u003cli\u003e\u003cp\u003e2.在正式查询之前，服务器会检查查询缓存，如果能找到对应的查询，则不必进行查询解析，优化，执行等过程，直接返回缓存中的结果集。\u003c/p\u003e\u003c/li\u003e\n\u003cli\u003e\u003cp\u003e3.MySQL的解析器会根据查询语句，构造出一个解析树，主要用于根据语法规则来验证语句是否正确，比如SQL的关键字是否正确，关键字的顺序是否正确。\u003c/p\u003e\u003c/li\u003e\n\u003c/ul\u003e\n\u003cp\u003e而预处理器主要是进一步校验，比如表名，字段名是否正确等\u003c/p\u003e\n\u003cul\u003e\n\u003cli\u003e\u003cp\u003e4.查询优化器将解析树转化为查询计划，一般情况下，一条查询可以有很多种执行方式，最终返回相同的结果，优化器就是根据\u003ccode\u003e成本\u003c/code\u003e找到这其中最优的执行计划\u003c/p\u003e\u003c/li\u003e\n\u003cli\u003e\u003cp\u003e5.执行计划调用查询执行引擎，而查询引擎通过一系列API接口查询到数据\u003c/p\u003e\u003c/li\u003e\n\u003cli\u003e\u003cp\u003e6.得到数据之后，在返回给客户端的同时，会将数据存在查询缓存中\u003c/p\u003e\u003c/li\u003e\n\u003c/ul\u003e\n\u003cdiv class=\"image-package\"\u003e\n\u003cdiv class=\"image-container\" style=\"max-width: 700px; max-height: 635px;\"\u003e\n\u003cdiv class=\"image-container-fill\" style=\"padding-bottom: 57.940000000000005%;\"\u003e\u003c/div\u003e\n\u003cdiv class=\"image-view\" data-width=\"1096\" data-height=\"635\"\u003e\u003cimg data-original-src=\"//upload-images.jianshu.io/upload_images/18375227-8e1f04724ca6881f.png\" data-original-width=\"1096\" data-original-height=\"635\" data-original-format=\"image/png\" data-original-filesize=\"311938\"\u003e\u003c/div\u003e\n\u003c/div\u003e\n\u003cdiv class=\"image-caption\"\u003eimage.png\u003c/div\u003e\n\u003c/div\u003e\n\u003ch2\u003e02  查询缓存\u003c/h2\u003e\n\u003cp\u003e我们先通过\u003ccode\u003eshow variables like '%query_cache%'\u003c/code\u003e来看一下默认的数据库配置，此为本地数据库的配置。\u003c/p\u003e\n\u003cdiv class=\"image-package\"\u003e\n\u003cdiv class=\"image-container\" style=\"max-width: 391px; max-height: 285px;\"\u003e\n\u003cdiv class=\"image-container-fill\" style=\"padding-bottom: 72.89%;\"\u003e\u003c/div\u003e\n\u003cdiv class=\"image-view\" data-width=\"391\" data-height=\"285\"\u003e\u003cimg data-original-src=\"//upload-images.jianshu.io/upload_images/18375227-42c0d6317f9323e9.png\" data-original-width=\"391\" data-original-height=\"285\" data-original-format=\"image/png\" data-original-filesize=\"87827\"\u003e\u003c/div\u003e\n\u003c/div\u003e\n\u003cdiv class=\"image-caption\"\u003eimage.png\u003c/div\u003e\n\u003c/div\u003e\n\u003ch3\u003e2.1  概念\u003c/h3\u003e\n\u003cp\u003ehave_query_cache:当前的MYSQL版本是否支持“查询缓存”功能。\u003c/p\u003e\n\u003cp\u003equery_cache_limit:MySQL能够缓存的最大查询结果，查询结果大于该值时不会被缓存。默认值是1048576(1MB)\u003c/p\u003e\n\u003cp\u003equery_cache_min_res_unit:查询缓存分配的最小块（字节）。默认值是4096（4KB）。当查询进行时，MySQL把查询结果保存在query cache，但是如果保存的结果比较大，超过了query_cache_min_res_unit的值，这时候MySQL将一边检索结果，一边进行保存结果。他保存结果也是按默认大小先分配一块空间，如果不够，又要申请新的空间给他。如果查询结果比较小，默认的query_cache_min_res_unit可能造成大量的内存碎片，如果查询结果比较大，默认的query_cache_min_res_unit又不够，导致一直分配块空间，所以可以根据实际需求，调节query_cache_min_res_unit的大小。\u003c/p\u003e\n\u003cp\u003e\u003ccode\u003e注：如果上面说的内容有点弯弯绕，那举个现实生活中的例子，比如咱现在要给运动员送水，默认的是500ml的瓶子，如果过来的是少年运动员，可能500ml太大了，他们喝不完，造成了浪费，那我们就可以选择300ml的瓶子，如果过来的是成年运动员，可能500ml不够，那他们一瓶喝完了，又开一瓶，直接不渴为止。那么那样开瓶子也要时间，我们就可以选择1000ml的瓶子。\u003c/code\u003e\u003c/p\u003e\n\u003cp\u003equery_cache_size:为缓存查询结果分配的总内存。\u003c/p\u003e\n\u003cp\u003equery_cache_type:默认为on，可以缓存除了以select sql_no_cache开头的所有查询结果。\u003c/p\u003e\n\u003cp\u003equery_cache_wlock_invalidate:如果该表被锁住，是否返回缓存中的数据，默认是关闭的。\u003c/p\u003e\n\u003ch3\u003e2.2  原理\u003c/h3\u003e\n\u003cp\u003eMYSQL的查询缓存实质上是缓存SQL的hash值和该SQL的查询结果，如果运行相同的SQL,服务器直接从缓存中去掉结果，而不再去解析，优化，寻找最低成本的执行计划等一系列操作，大大提升了查询速度。\u003c/p\u003e\n\u003cp\u003e但是万事有利也有弊。\u003c/p\u003e\n\u003cul\u003e\n\u003cli\u003e第一个弊端就是如果表的数据有一条发生变化，那么缓存好的结果将全部不再有效。这对于频繁更新的表，查询缓存是不适合的。\u003c/li\u003e\n\u003c/ul\u003e\n\u003cp\u003e\u003ccode\u003e比如一张表里面只有两个字段，分别是id和name，数据有一条为1，张三。我使用select * from 表名 where name=“张三”来进行查询，MySQL发现查询缓存中没有此数据，会进行一系列的解析，优化等操作进行数据的查询，查询结束之后将该SQL的hash和查询结果缓存起来，并将查询结果返回给客户端。但是这个时候我有新增了一条数据2，张三。如果我还用相同的SQL来执行，他会根据该SQL的hash值去查询缓存中，那么结果就错了。所以MySQL对于数据有变化的表来说，会直接清空关于该表的所有缓存。这样其实是效率是很差的。\u003c/code\u003e\u003c/p\u003e\n\u003cul\u003e\n\u003cli\u003e第二个弊端就是缓存机制是通过对SQL的hash，得出的值为key，查询结果为value来存放的，那么就意味着SQL必须完完全全一模一样，否则就命不中缓存。\u003c/li\u003e\n\u003c/ul\u003e\n\u003cp\u003e\u003ccode\u003e我们都知道hash值的规则，就算很小的查询，哈希出来的结果差距是很多的，所以select * from 表名 where name=“张三”和SELECT * FROM 表名 WHERE NAME=“张三”和select * from 表名 where name = “张三”，三个SQL哈希出来的值是不一样的，大小写和空格影响了他们，所以并不能命中缓存，但其实他们搜索结果是完全一样的。\u003c/code\u003e\u003c/p\u003e\n\u003ch3\u003e2.3  生产如何设置MySQL Query Cache\u003c/h3\u003e\n\u003cp\u003e先来看线上参数：\u003c/p\u003e\n\u003cdiv class=\"image-package\"\u003e\n\u003cdiv class=\"image-container\" style=\"max-width: 352px; max-height: 240px;\"\u003e\n\u003cdiv class=\"image-container-fill\" style=\"padding-bottom: 68.17999999999999%;\"\u003e\u003c/div\u003e\n\u003cdiv class=\"image-view\" data-width=\"352\" data-height=\"240\"\u003e\u003cimg data-original-src=\"//upload-images.jianshu.io/upload_images/18375227-ab994b6bfd4d1e61.png\" data-original-width=\"352\" data-original-height=\"240\" data-original-format=\"image/png\" data-original-filesize=\"83326\"\u003e\u003c/div\u003e\n\u003c/div\u003e\n\u003cdiv class=\"image-caption\"\u003eimage.png\u003c/div\u003e\n\u003c/div\u003e\n\u003cp\u003e我们发现将query_cache_type设置为OFF，其实网上资料和各大云厂商提供的云服务器都是将这个功能关闭的，从上面的原理来看，在一般情况下，\u003ccode\u003e他的弊端大于优点\u003c/code\u003e。\u003c/p\u003e\n\u003ch2\u003e03  索引\u003c/h2\u003e\n\u003ch3\u003e3.1  例子\u003c/h3\u003e\n\u003cp\u003e创建一个名为user的表，其包括id，name，age，sex等字段信息。此外，id为主键聚簇索引，idx_name为非聚簇索引。\u003c/p\u003e\n\u003cpre\u003e\u003ccode\u003eCREATE TABLE `user` (\n  `id` varchar(10) NOT NULL DEFAULT '',\n  `name` varchar(10) DEFAULT NULL,\n  `age` int(11) DEFAULT NULL,\n  `sex` varchar(10) DEFAULT NULL,\n  PRIMARY KEY (`id`),\n  KEY `idx_name` (`name`) USING BTREE\n) ENGINE=InnoDB DEFAULT CHARSET=utf8;\n\u003c/code\u003e\u003c/pre\u003e\n\u003cp\u003e我们将其设置10条数据，便于下面的索引的理解。\u003c/p\u003e\n\u003cpre\u003e\u003ccode\u003eINSERT INTO `user` VALUES ('1', 'andy', '20', '女');\nINSERT INTO `user` VALUES ('10', 'baby', '12', '女');\nINSERT INTO `user` VALUES ('2', 'kat', '12', '女');\nINSERT INTO `user` VALUES ('3', 'lili', '20', '男');\nINSERT INTO `user` VALUES ('4', 'lucy', '22', '女');\nINSERT INTO `user` VALUES ('5', 'bill', '20', '男');\nINSERT INTO `user` VALUES ('6', 'zoe', '20', '男');\nINSERT INTO `user` VALUES ('7', 'hay', '20', '女');\nINSERT INTO `user` VALUES ('8', 'tony', '20', '男');\nINSERT INTO `user` VALUES ('9', 'rose', '21', '男');\n\u003c/code\u003e\u003c/pre\u003e\n\u003ch3\u003e3.2  聚簇索引（主键索引）\u003c/h3\u003e\n\u003cp\u003e先来一张图镇楼，接下来就是看图说话。\u003c/p\u003e\n\u003cdiv class=\"image-package\"\u003e\n\u003cdiv class=\"image-container\" style=\"max-width: 596px; max-height: 427px;\"\u003e\n\u003cdiv class=\"image-container-fill\" style=\"padding-bottom: 71.64%;\"\u003e\u003c/div\u003e\n\u003cdiv class=\"image-view\" data-width=\"596\" data-height=\"427\"\u003e\u003cimg data-original-src=\"//upload-images.jianshu.io/upload_images/18375227-1c8d30f7eebed5dd.png\" data-original-width=\"596\" data-original-height=\"427\" data-original-format=\"image/png\" data-original-filesize=\"126958\"\u003e\u003c/div\u003e\n\u003c/div\u003e\n\u003cdiv class=\"image-caption\"\u003eimage.png\u003c/div\u003e\n\u003c/div\u003e\n\u003cp\u003e他包含两个特点：\u003c/p\u003e\n\u003cp\u003e1.使用记录主键值的大小来进行记录和页的排序。\u003c/p\u003e\n\u003cp\u003e页内的记录是按照主键的大小顺序排成一个单项链表。\u003c/p\u003e\n\u003cp\u003e各个存放用户记录的页也是根据页中用户记录的主键大小顺序排成一个双向链表。\u003c/p\u003e\n\u003cp\u003e2.叶子节点存储的是\u003ccode\u003e完整的用户记录\u003c/code\u003e。\u003c/p\u003e\n\u003cpre\u003e\u003ccode\u003e注：聚簇索引不需要我们显示的创建，他是由InnoDB存储引擎自动为我们创建的。如果没有主键，其也会默认创建一个。复制代码\n\u003c/code\u003e\u003c/pre\u003e\n\u003ch3\u003e3.3  非聚簇索引（二级索引）\u003c/h3\u003e\n\u003cp\u003e上面的聚簇索引只能在搜索条件是主键时才能发挥作用，因为聚簇索引可以根据主键进行排序的。如果搜索条件是name，在刚才的聚簇索引上，我们可能遍历，挨个找到符合条件的记录，但是，这样真的是太蠢了，MySQL不会这样做的。\u003c/p\u003e\n\u003cp\u003e如果我们想让搜索条件是name的时候，也能使用索引，那可以多创建一个基于name的二叉树。如下图。\u003c/p\u003e\n\u003cdiv class=\"image-package\"\u003e\n\u003cdiv class=\"image-container\" style=\"max-width: 619px; max-height: 407px;\"\u003e\n\u003cdiv class=\"image-container-fill\" style=\"padding-bottom: 65.75%;\"\u003e\u003c/div\u003e\n\u003cdiv class=\"image-view\" data-width=\"619\" data-height=\"407\"\u003e\u003cimg data-original-src=\"//upload-images.jianshu.io/upload_images/18375227-c5414349d1c19c5c.png\" data-original-width=\"619\" data-original-height=\"407\" data-original-format=\"image/png\" data-original-filesize=\"81211\"\u003e\u003c/div\u003e\n\u003c/div\u003e\n\u003cdiv class=\"image-caption\"\u003eimage.png\u003c/div\u003e\n\u003c/div\u003e\n\u003cp\u003e他与聚簇索引的不同：\u003c/p\u003e\n\u003cp\u003e1.叶子节点内部使用name字段排序，叶子节点之间也是使用name字段排序。\u003c/p\u003e\n\u003cp\u003e2.叶子节点不再是完整的数据记录，而是name和主键值。\u003c/p\u003e\n\u003cp\u003e\u003ccode\u003e为什么不再是完整信息？\u003c/code\u003e\u003c/p\u003e\n\u003cp\u003eMySQL只让聚簇索引的叶子节点存放完整的记录信息，因为如果有好几个非聚簇索引，他们的叶子节点也存放完整的记录绩效，那就不浪费空间啦。\u003c/p\u003e\n\u003cp\u003e\u003ccode\u003e如果我搜索条件是基于name，需要查询所有字段的信息，那查询过程是啥？\u003c/code\u003e\u003c/p\u003e\n\u003cp\u003e1.根据查询条件，采用name的非聚簇索引，先定位到该非聚簇索引某些记录行。\u003c/p\u003e\n\u003cp\u003e2.根据记录行找到相应的id，再根据id到聚簇索引中找到相关记录。这个过程叫做\u003ccode\u003e回``表\u003c/code\u003e。\u003c/p\u003e\n\u003ch3\u003e3.4  联合索引\u003c/h3\u003e\n\u003cp\u003e图就不画了，简单来说，如果name和age组成一个联合索引，那么先按name排序，如果name一样，就按age排序。\u003c/p\u003e\n\u003ch3\u003e3.5  一些原则\u003c/h3\u003e\n\u003cp\u003e1.最左前缀原则。一个联合索引（a,b,c）,如果有一个查询条件有a，有b，那么他则走索引，如果有一个查询条件没有a，那么他则不走索引。\u003c/p\u003e\n\u003cp\u003e2.使用唯一索引。具有多个重复值的列，其索引效果最差。例如，存放姓名的列具有不同值，很容易区分每行。而用来记录性别的列，只含有“男”，“女”，不管搜索哪个值，都会得出大约一半的行，这样的索引对性能的提升不够高。\u003c/p\u003e\n\u003cp\u003e3.不要过度索引。每个额外的索引都要占用额外的磁盘空间，并降低写操作的性能。在修改表的内容时，索引必须进行更新，有时可能需要重构，因此，索引越多，所花的时间越长。\u003c/p\u003e\n\u003cp\u003e4、索引列不能参与计算，保持列“干净”，比如from_unixtime(create_time) = ’2014-05-29’就不能使用到索引，原因很简单，b+树中存的都是数据表中的字段值，但进行检索时，需要把所有元素都应用函数才能比较，显然成本太大。所以语句应该写成create_time = unix_timestamp(’2014-05-29’);\u003c/p\u003e\n\u003cp\u003e5.一定要设置一个主键。前面聚簇索引说到如果不指定主键，InnoDB会自动为其指定主键，这个我们是看不见的。反正都要生成一个主键的，还不如我们设置，以后在某些搜索条件时还能用到主键的聚簇索引。\u003c/p\u003e\n\u003cp\u003e6.主键推荐用自增id，而不是uuid。上面的聚簇索引说到每页数据都是排序的，并且页之间也是排序的，如果是uuid，那么其肯定是随机的，其可能从中间插入，导致页的分裂，产生很多表碎片。如果是自增的，那么其有从小到大自增的，有顺序，那么在插入的时候就添加到当前索引的后续位置。当一页写满，就会自动开辟一个新的页。\u003c/p\u003e\n\u003cpre\u003e\u003ccode\u003e注：如果自增id用完了，那将字段类型改为bigint，就算每秒1万条数据，跑100年，也没达到bigint的最大值。复制代码\n\u003c/code\u003e\u003c/pre\u003e\n\u003ch3\u003e3.6  万年面试题（为什么索引用B+树）\u003c/h3\u003e\n\u003cp\u003e1、 B+树的磁盘读写代价更低：B+树的内部节点并没有指向关键字具体信息的指针，因此其内部节点相对B树更小，如果把所有同一内部节点的关键字存放在同一盘块中，那么盘块所能容纳的关键字数量也越多，一次性读入内存的需要查找的关键字也就越多，相对\u003ccode\u003eIO读写次数就降低\u003c/code\u003e了。\u003c/p\u003e\n\u003cp\u003e2、由于B+树的数据都存储在叶子结点中，分支结点均为索引，方便扫库，只需要扫一遍叶子结点即可，但是B树因为其分支结点同样存储着数据，我们要找到具体的数据，需要进行一次中序遍历按序来扫，所以B+树更加适合在\u003ccode\u003e区间查询\u003c/code\u003e的情况，所以通常B+树用于数据库索引。\u003c/p\u003e\n\u003ch2\u003e04  优化器\u003c/h2\u003e\n\u003cp\u003e在开篇的图里面，我们知道了SQL语句从客户端经由网络协议到查询缓存，如果没有命中缓存，再经过解析工作，得到准确的SQL，现在就来到了我们这模块说的优化器。\u003c/p\u003e\n\u003cp\u003e首先，我们知道每一条SQL都有不同的执行方法，要不通过索引，要不通过全表扫描的方式。\u003c/p\u003e\n\u003cp\u003e那么问题就来了，MySQL是如何选择时间最短，占用内存最小的执行方法呢？\u003c/p\u003e\n\u003ch3\u003e4.1  什么是成本？\u003c/h3\u003e\n\u003cp\u003e1.I/O成本。数据存储在硬盘上，我们想要进行某个操作需要将其加载到内存中，这个过程的时间被称为I/O成本。默认是1。\u003c/p\u003e\n\u003cp\u003e2.CPU成本。在内存对结果集进行排序的时间被称为CPU成本。默认是0.2。\u003c/p\u003e\n\u003ch3\u003e4.2  单表查询的成本\u003c/h3\u003e\n\u003cp\u003e先来建一个用户表dev_user，里面包括主键id，用户名username，密码password，外键user_info_id，状态status，外键main_station_id，是否外网访问visit，这七个字段。索引有两个，一个是主键的聚簇索引，另一个是显式添加的以username为字段的唯一索引uname_unique。\u003c/p\u003e\n\u003cdiv class=\"image-package\"\u003e\n\u003cdiv class=\"image-container\" style=\"max-width: 700px; max-height: 80px;\"\u003e\n\u003cdiv class=\"image-container-fill\" style=\"padding-bottom: 10.879999999999999%;\"\u003e\u003c/div\u003e\n\u003cdiv class=\"image-view\" data-width=\"735\" data-height=\"80\"\u003e\u003cimg data-original-src=\"//upload-images.jianshu.io/upload_images/18375227-91e6da824076e362.png\" data-original-width=\"735\" data-original-height=\"80\" data-original-format=\"image/png\" data-original-filesize=\"39454\"\u003e\u003c/div\u003e\n\u003c/div\u003e\n\u003cdiv class=\"image-caption\"\u003eimage.png\u003c/div\u003e\n\u003c/div\u003e\n\u003cp\u003e如果搜索条件是select * from dev_user where username='XXX'，那么MySQL是如何选择相关索引呢？\u003c/p\u003e\n\u003cp\u003e1.使用所有可能用到的索引\u003c/p\u003e\n\u003cp\u003e我们可以看到搜索条件username，所以可能走uname_unique索引。也可以做聚簇索引，也就是全表扫描。\u003c/p\u003e\n\u003cp\u003e2.计算全表扫描代价\u003c/p\u003e\n\u003cp\u003e我们通过\u003ccode\u003eshow table status like ‘dev_user’\u003c/code\u003e命令知道\u003ccode\u003erows\u003c/code\u003e和\u003ccode\u003edata_length\u003c/code\u003e字段，如下图。\u003c/p\u003e\n\u003cdiv class=\"image-package\"\u003e\n\u003cdiv class=\"image-container\" style=\"max-width: 700px; max-height: 155px;\"\u003e\n\u003cdiv class=\"image-container-fill\" style=\"padding-bottom: 12.11%;\"\u003e\u003c/div\u003e\n\u003cdiv class=\"image-view\" data-width=\"1280\" data-height=\"155\"\u003e\u003cimg data-original-src=\"//upload-images.jianshu.io/upload_images/18375227-ad55c097cb502d93.png\" data-original-width=\"1280\" data-original-height=\"155\" data-original-format=\"image/png\" data-original-filesize=\"111939\"\u003e\u003c/div\u003e\n\u003c/div\u003e\n\u003cdiv class=\"image-caption\"\u003eimage.png\u003c/div\u003e\n\u003c/div\u003e\n\u003cp\u003erows：表示表中的记录条数，但是这个数据不准确，是个估计值。\u003c/p\u003e\n\u003cp\u003edata_length:表示表占用的存储空间字节数。\u003c/p\u003e\n\u003cp\u003e\u003ccode\u003edata_length=聚簇索引的页面数量X每个页面的大小\u003c/code\u003e\u003c/p\u003e\n\u003cp\u003e反推出页面数量=1589248÷16÷1024=97\u003c/p\u003e\n\u003cp\u003eI/O成本：97X1=97\u003c/p\u003e\n\u003cp\u003eCPU成本：6141X0.2=1228\u003c/p\u003e\n\u003cp\u003e总成本：97+1228=1325\u003c/p\u003e\n\u003cp\u003e3.计算使用不同索引执行查询的代价\u003c/p\u003e\n\u003cp\u003e因为要查询出满足条件的所有字段信息，所以要考虑回表成本。\u003c/p\u003e\n\u003cp\u003eI/O成本=1+1X1=2(范围区间的数量+预计二级记录索引条数)\u003c/p\u003e\n\u003cp\u003eCPU成本=1X0.2+1X0.2=0.4(读取二级索引的成本+回表聚簇索引的成本)\u003c/p\u003e\n\u003cp\u003e总成本=I/O成本+CPU成本=2.4\u003c/p\u003e\n\u003cp\u003e4.对比各种执行方案的代价，找出成本最低的那个\u003c/p\u003e\n\u003cp\u003e上面两个数字一对比，成本是采用uname_unique索引成本最低。\u003c/p\u003e\n\u003ch3\u003e4.3  多表查询的成本\u003c/h3\u003e\n\u003cp\u003e对于两表连接查询来说，他的查询成本由下面两个部分构成：\u003c/p\u003e\n\u003cul\u003e\n\u003cli\u003e单次查询驱动表的成本\u003c/li\u003e\n\u003cli\u003e多次查询被驱动表的成本（具体查询多次取决于对驱动表查询的结果集有多少个记录）\u003c/li\u003e\n\u003c/ul\u003e\n\u003ch3\u003e4.4  index dive\u003c/h3\u003e\n\u003cp\u003e如果前面的搜索条件不是等值，而是区间，如\u003ccode\u003eselect * from dev_user where username\u0026gt;'admin' and username\u0026lt;'test'\u003c/code\u003e这个时候我们是无法看出需要回表的数量。\u003c/p\u003e\n\u003cp\u003e步骤1：先根据username\u0026gt;'admin'这个条件找到第一条记录，称为\u003ccode\u003e区间最左记录\u003c/code\u003e。\u003c/p\u003e\n\u003cp\u003e步骤2：再根据username\u0026lt;'test'这个条件找到最后一条记录，称为\u003ccode\u003e区间最右记录\u003c/code\u003e。\u003c/p\u003e\n\u003cp\u003e步骤3：如果区间最左记录和区间最右记录相差不是很远，可以准确统计出需要回表的数量。如果相差很远，就先计算10页有多少条记录，再乘以页面数量，最终模糊统计出来。\u003c/p\u003e\n\u003ch2\u003e05  Explain\u003c/h2\u003e\n\u003ch3\u003e5.1  产品来索命\u003c/h3\u003e\n\u003cp\u003e产品：为什么这个页面出来这么慢？\u003c/p\u003e\n\u003cp\u003e开发：因为你查的数据多呗，他就是这么慢\u003c/p\u003e\n\u003cp\u003e产品：我不管，我要这个页面快点，你这样，客户怎么用啊\u003c/p\u003e\n\u003cp\u003e开发：。。。。。。。你行你来\u003c/p\u003e\n\u003cdiv class=\"image-package\"\u003e\n\u003cdiv class=\"image-container\" style=\"max-width: 500px; max-height: 449px;\"\u003e\n\u003cdiv class=\"image-container-fill\" style=\"padding-bottom: 89.8%;\"\u003e\u003c/div\u003e\n\u003cdiv class=\"image-view\" data-width=\"500\" data-height=\"449\"\u003e\u003cimg data-original-src=\"//upload-images.jianshu.io/upload_images/18375227-7f6e414bed8e9694.png\" data-original-width=\"500\" data-original-height=\"449\" data-original-format=\"image/png\" data-original-filesize=\"306785\"\u003e\u003c/div\u003e\n\u003c/div\u003e\n\u003cdiv class=\"image-caption\"\u003eimage.png\u003c/div\u003e\n\u003c/div\u003e\n\u003cp\u003e哈哈哈哈，不瞎BB啦，如果有些SQL贼慢，我们需要知道他有没有走索引，走了哪个索引，这个时候我就需要通过explain关键字来深入了解MySQL内部是如何执行的。\u003c/p\u003e\n\u003cdiv class=\"image-package\"\u003e\n\u003cdiv class=\"image-container\" style=\"max-width: 700px; max-height: 175px;\"\u003e\n\u003cdiv class=\"image-container-fill\" style=\"padding-bottom: 22.18%;\"\u003e\u003c/div\u003e\n\u003cdiv class=\"image-view\" data-width=\"789\" data-height=\"175\"\u003e\u003cimg data-original-src=\"//upload-images.jianshu.io/upload_images/18375227-12dc862c5053cd89.png\" data-original-width=\"789\" data-original-height=\"175\" data-original-format=\"image/png\" data-original-filesize=\"84358\"\u003e\u003c/div\u003e\n\u003c/div\u003e\n\u003cdiv class=\"image-caption\"\u003eimage.png\u003c/div\u003e\n\u003c/div\u003e\n\u003ch3\u003e5.2  id\u003c/h3\u003e\n\u003cp\u003e一般来说一个select一个唯一id，如果是子查询，就有两个select，id是不一样的，但是凡事有例外，有些子查询的，他们id是一样的。\u003c/p\u003e\n\u003cbr\u003e\n\u003cdiv class=\"image-package\"\u003e\n\u003cdiv class=\"image-container\" style=\"max-width: 700px; max-height: 153px;\"\u003e\n\u003cdiv class=\"image-container-fill\" style=\"padding-bottom: 20.51%;\"\u003e\u003c/div\u003e\n\u003cdiv class=\"image-view\" data-width=\"746\" data-height=\"153\"\u003e\u003cimg data-original-src=\"//upload-images.jianshu.io/upload_images/18375227-65d384dbb5af5c09.png\" data-original-width=\"746\" data-original-height=\"153\" data-original-format=\"image/png\" data-original-filesize=\"93904\"\u003e\u003c/div\u003e\n\u003c/div\u003e\n\u003cdiv class=\"image-caption\"\u003eimage.png\u003c/div\u003e\n\u003c/div\u003e\n\u003cp\u003e这是为什么呢？\u003c/p\u003e\n\u003cp\u003e那是因为MySQL在进行优化的时候已经将子查询改成了连接查询，而连接查询的id是一样的。\u003c/p\u003e\n\u003ch3\u003e5.3  select_type\u003c/h3\u003e\n\u003cul\u003e\n\u003cli\u003esimple：不包括union和子查询的查询都算simple类型。\u003c/li\u003e\n\u003cli\u003eprimary：包括union，union all，其中最左边的查询即为primary。\u003c/li\u003e\n\u003cli\u003eunion：包括union，union all，除了最左边的查询，其他的查询类型都为union。\u003c/li\u003e\n\u003c/ul\u003e\n\u003ch3\u003e5.4 table\u003c/h3\u003e\n\u003cp\u003e显示这一行是关于哪张表的。\u003c/p\u003e\n\u003ch3\u003e5.5 type：访问方法\u003c/h3\u003e\n\u003cul\u003e\n\u003cli\u003eref：普通二级索引与常量进行等值匹配\u003c/li\u003e\n\u003cli\u003eref_or_null：普通二级索引与常量进行等值匹配，该索引可能是null\u003c/li\u003e\n\u003cli\u003econst：主键或唯一二级索引列与常量进行等值匹配\u003c/li\u003e\n\u003cli\u003erange：范围区间的查询\u003c/li\u003e\n\u003cli\u003eall：全表扫描\u003c/li\u003e\n\u003c/ul\u003e\n\u003ch3\u003e5.6 possible_keys\u003c/h3\u003e\n\u003cp\u003e对某表进行单表查询时可能用到的索引\u003c/p\u003e\n\u003ch3\u003e5.7 key\u003c/h3\u003e\n\u003cp\u003e经过查询优化器计算不同索引的成本，最终选择成本最低的索引\u003c/p\u003e\n\u003ch3\u003e5.8 rows\u003c/h3\u003e\n\u003cul\u003e\n\u003cli\u003e如果使用全表扫描，那么rows就代表需要扫描的行数\u003c/li\u003e\n\u003cli\u003e如果使用索引，那么rows就代表预计扫描的行数\u003c/li\u003e\n\u003c/ul\u003e\n\u003ch3\u003e5.9 filtered\u003c/h3\u003e\n\u003cul\u003e\n\u003cli\u003e如果全表扫描，那么filtered就代表满足搜索条件的记录的满分比\u003c/li\u003e\n\u003cli\u003e如果是索引，那么filtered就代表除去索引对应的搜索，其他搜索条件的百分比\u003c/li\u003e\n\u003c/ul\u003e\n\u003ch2\u003e06 redo日志（物理日志）\u003c/h2\u003e\n\u003cp\u003eInnoDB存储引擎是以页为单位来管理存储空间的，我们进行的增删改查操作都是将页的数据加载到内存中，然后进行操作，再将数据刷回到硬盘上。\u003c/p\u003e\n\u003cp\u003e那么问题就来了，如果我要给张三转账100块钱，事务已经提交了，这个时候InnoDB把数据加载到内存中，这个时候还没来得及刷入硬盘，突然停电了，数据库崩了。重启之后，发现我的钱没有转成功，这不是尴尬了吗？\u003c/p\u003e\n\u003cp\u003e解决方法很明显，我们在硬盘加载到内存之后，进行一系列操作，一顿操作猛如虎，还未刷新到硬盘之前，先记录下，在XXX位置我的记录中金额减100，在XXX位置张三的记录中金额加100，然后再进行增删改查操作，最后刷入硬盘。如果未刷入硬盘，在重启之后，先加载之前的记录，那么数据就回来了。\u003c/p\u003e\n\u003cp\u003e这个记录就叫做重做日志，即redo日志。他的目的是想让已经提交的事务对数据的修改是永久的，就算他重启，数据也能恢复出来。\u003c/p\u003e\n\u003ch3\u003e6.1  log buffer（日志缓冲区）\u003c/h3\u003e\n\u003cp\u003e为了解决磁盘速度过慢的问题，redo日志不能直接写入磁盘，咱先整一大片连续的内存空间给他放数据。这一大片内存就叫做日志缓冲区，即log buffer。到了合适的时候，再刷入硬盘。至于什么时候是合适的，这个下一章节说。\u003c/p\u003e\n\u003cp\u003e我们可以通过\u003ccode\u003eshow VARIABLES like 'innodb_log_buffer_size'\u003c/code\u003e命令来查看当前的日志缓存大小，下图为线上的大小。\u003c/p\u003e\n\u003cdiv class=\"image-package\"\u003e\n\u003cdiv class=\"image-container\" style=\"max-width: 407px; max-height: 135px;\"\u003e\n\u003cdiv class=\"image-container-fill\" style=\"padding-bottom: 33.17%;\"\u003e\u003c/div\u003e\n\u003cdiv class=\"image-view\" data-width=\"407\" data-height=\"135\"\u003e\u003cimg data-original-src=\"//upload-images.jianshu.io/upload_images/18375227-61ec519d9b02b091.png\" data-original-width=\"407\" data-original-height=\"135\" data-original-format=\"image/png\" data-original-filesize=\"44556\"\u003e\u003c/div\u003e\n\u003c/div\u003e\n\u003cdiv class=\"image-caption\"\u003eimage.png\u003c/div\u003e\n\u003c/div\u003e\n\u003ch3\u003e6.2 redo日志刷盘时机\u003c/h3\u003e\n\u003cp\u003e由于redo日志一直都是增长的，且内存空间有限，数据也不能一直待在缓存中， 我们需要将其刷新至硬盘上。\u003c/p\u003e\n\u003cp\u003e那什么时候刷新到硬盘呢？\u003c/p\u003e\n\u003cul\u003e\n\u003cli\u003elog buffer空间不足。上面有指定缓冲区的内存大小，MySQL认为日志量已经占了 总容量的一半左右，就需要将这些日志刷新到磁盘上。\u003c/li\u003e\n\u003cli\u003e事务提交时。我们使用redo日志的目的就是将他未刷新到磁盘的记录保存起来，防止 丢失，如果数据提交了，我们是可以不把数据提交到磁盘的，但为了保证持久性，必须 把修改这些页面的redo日志刷新到磁盘。\u003c/li\u003e\n\u003cli\u003e后台线程不同的刷新 后台有一个线程，大概每秒都会将log buffer里面的redo日志刷新到硬盘上。\u003c/li\u003e\n\u003cli\u003echeckpoint 下下小节讲\u003c/li\u003e\n\u003c/ul\u003e\n\u003ch3\u003e6.3 redo日志文件组\u003c/h3\u003e\n\u003cp\u003e我们可以通过\u003ccode\u003eshow variables like 'datadir'\u003c/code\u003e命令找到相关目录，底下有两个文件， 分别是ib_logfile0和ib_logfile1,如下图所示。\u003c/p\u003e\n\u003cdiv class=\"image-package\"\u003e\n\u003cdiv class=\"image-container\" style=\"max-width: 483px; max-height: 153px;\"\u003e\n\u003cdiv class=\"image-container-fill\" style=\"padding-bottom: 31.680000000000003%;\"\u003e\u003c/div\u003e\n\u003cdiv class=\"image-view\" data-width=\"483\" data-height=\"153\"\u003e\u003cimg data-original-src=\"//upload-images.jianshu.io/upload_images/18375227-31dd34338d350428.png\" data-original-width=\"483\" data-original-height=\"153\" data-original-format=\"image/png\" data-original-filesize=\"44921\"\u003e\u003c/div\u003e\n\u003c/div\u003e\n\u003cdiv class=\"image-caption\"\u003eimage.png\u003c/div\u003e\n\u003c/div\u003e\n\u003cdiv class=\"image-package\"\u003e\n\u003cdiv class=\"image-container\" style=\"max-width: 623px; max-height: 431px;\"\u003e\n\u003cdiv class=\"image-container-fill\" style=\"padding-bottom: 69.17999999999999%;\"\u003e\u003c/div\u003e\n\u003cdiv class=\"image-view\" data-width=\"623\" data-height=\"431\"\u003e\u003cimg data-original-src=\"//upload-images.jianshu.io/upload_images/18375227-8116696aedd60988.png\" data-original-width=\"623\" data-original-height=\"431\" data-original-format=\"image/png\" data-original-filesize=\"260850\"\u003e\u003c/div\u003e\n\u003c/div\u003e\n\u003cdiv class=\"image-caption\"\u003eimage.png\u003c/div\u003e\n\u003c/div\u003e\n\u003cp\u003e我们将缓冲区log buffer里面的redo日志刷新到这个两个文件里面，他们写入的方式 是循环写入的，先写ib_logfile0,再写ib_logfile1,等ib_logfile1写满了，再写ib_logfile0。 那这样就会存在一个问题，如果ib_logfile1写满了，再写ib_logfile0，之前ib_logfile0的内容 不就被覆盖而丢失了吗？ 这就是checkpoint的工作啦。\u003c/p\u003e\n\u003ch3\u003e6.4  checkpoint\u003c/h3\u003e\n\u003cp\u003eredo日志是为了系统崩溃后恢复脏页用的，如果这个脏页可以被刷新到磁盘上，那么 他就可以功成身退，被覆盖也就没事啦。\u003c/p\u003e\n\u003cp\u003e冲突补习\u003c/p\u003e\n\u003cp\u003e从系统运行开始，就不断的修改页面，会不断的生成redo日志。redo日志是不断 递增的，MySQL为其取了一个名字日志序列号Log Sequence Number，简称lsn。 他的初始化的值为8704，用来记录当前一共生成了多少redo日志。\u003c/p\u003e\n\u003cp\u003eredo日志是先写入log buffer，之后才会被刷新到磁盘的redo日志文件。MySQL为其 取了一个名字flush_to_disk_lsn。用来说明缓存区中有多少的脏页数据被刷新到磁盘上啦。 他的初始值和lsn一样，后面的差距就有了。\u003c/p\u003e\n\u003cp\u003e做一次checkpoint分为两步\u003c/p\u003e\n\u003cul\u003e\n\u003cli\u003e计算当前系统可以被覆盖的redo日志对应的lsn最大值是多少。redo日志可以被覆盖， 意味着他对应的脏页被刷新到磁盘上，只要我们计算出当前系统中最早被修改的oldest_modification, 只要系统中lsn小于该节点的oldest_modification值磁盘的redo日志都是可以被覆盖的。\u003c/li\u003e\n\u003cli\u003e将lsn过程中的一些数据统计。\u003c/li\u003e\n\u003c/ul\u003e\n\u003ch2\u003e07 undo日志（这部分不是很明白，所以大概说了）\u003c/h2\u003e\n\u003ch3\u003e7.1 基本概念\u003c/h3\u003e\n\u003cp\u003eundo log有两个作用：提供回滚和多个行版本控制(\u003ccode\u003eMVCC\u003c/code\u003e)。\u003c/p\u003e\n\u003cp\u003eundo log和redo log记录物理日志不一样，它是逻辑日志。可以认为当delete一条记录时，undo log中会记录一条对应的insert记录，反之亦然，当update一条记录时，它记录一条对应相反的update记录。\u003c/p\u003e\n\u003cp\u003e举个例子:\u003c/p\u003e\n\u003cp\u003einsert into a(id) values(1);(redo)\u003cbr\u003e\n这条记录是需要回滚的。\u003cbr\u003e\n回滚的语句是delete from a where id = 1;(undo)\u003c/p\u003e\n\u003cp\u003e试想想看。如果没有做insert into a(id) values(1);(redo)\u003cbr\u003e\n那么delete from a where id = 1;(undo)这句话就没有意义了。\u003c/p\u003e\n\u003cp\u003e现在看下正确的恢复:\u003cbr\u003e\n先insert into a(id) values(1);(redo)\u003cbr\u003e\n然后delete from a where id = 1;(undo)\u003cbr\u003e\n系统就回到了原先的状态，没有这条记录了\u003c/p\u003e\n\u003ch3\u003e7.2  存储方式\u003c/h3\u003e\n\u003cp\u003e是存在段之中。\u003c/p\u003e\n\u003ch2\u003e08  事务\u003c/h2\u003e\n\u003ch3\u003e8.1  引言\u003c/h3\u003e\n\u003cp\u003e事务中有一个隔离性特征，理论上在某个事务对某个数据进行访问时，其他事务应该排序，当该事务提交之后，其他事务才能继续访问这个数据。\u003c/p\u003e\n\u003cp\u003e但是这样子对性能影响太大，我们既想保持事务的隔离性，又想让服务器在出来多个事务时性能尽量高些，所以只能舍弃一部分隔离性而去性能。\u003c/p\u003e\n\u003ch3\u003e8.2  事务并发执行的问题\u003c/h3\u003e\n\u003cul\u003e\n\u003cli\u003e脏写（这个太严重了，任何隔离级别都不允许发生）\u003c/li\u003e\n\u003c/ul\u003e\n\u003cp\u003esessionA：修改了一条数据，回滚掉\u003c/p\u003e\n\u003cp\u003esessionB：修改了同一条数据，提交掉\u003c/p\u003e\n\u003cp\u003e对于sessionB来说，明明数据更新了也提交了事务，不能说自己啥都没干\u003c/p\u003e\n\u003cul\u003e\n\u003cli\u003e脏读：一个事务读到另一个未提交事务修改的数据\u003c/li\u003e\n\u003c/ul\u003e\n\u003cp\u003esession A：查询，得到某条数据\u003c/p\u003e\n\u003cp\u003esession B：修改某条数据，但是最后回滚掉啦\u003c/p\u003e\n\u003cp\u003esession A：在sessionB修改某条数据之后，在回滚之前，读取了该条记录\u003c/p\u003e\n\u003cp\u003e对于session A来说，读到了session回滚之前的脏数据\u003c/p\u003e\n\u003cul\u003e\n\u003cli\u003e不可重复读：前后多次读取，同一个数据内容不一样\u003c/li\u003e\n\u003c/ul\u003e\n\u003cp\u003esession A：查询某条记录\u003cbr\u003e\nsession B : 修改该条记录，并提交事务\u003cbr\u003e\nsession A : 再次查询该条记录，发现前后查询不一致\u003c/p\u003e\n\u003cul\u003e\n\u003cli\u003e幻读：前后多次读取，数据总量不一致\u003c/li\u003e\n\u003c/ul\u003e\n\u003cp\u003esession A：查询表内所有记录\u003cbr\u003e\nsession B : 新增一条记录，并查询表内所有记录\u003cbr\u003e\nsession A : 再次查询该条记录，发现前后查询不一致\u003c/p\u003e\n\u003ch3\u003e8.3  四种隔离级别\u003c/h3\u003e\n\u003cp\u003e数据库都有的四种隔离级别，MySQL事务默认的隔离级别是可重复读，而且MySQL可以解决了幻读的问题。\u003c/p\u003e\n\u003cul\u003e\n\u003cli\u003e未提交读：脏读，不可重复读，幻读都有可能发生\u003c/li\u003e\n\u003cli\u003e已提交读：不可重复读，幻读可能发生\u003c/li\u003e\n\u003cli\u003e可重复读：幻读可能发生\u003c/li\u003e\n\u003cli\u003e可串行化：都不可能发生\u003c/li\u003e\n\u003c/ul\u003e\n\u003cp\u003e但凡事没有百分百，emmmm，其实MySQL并没有百分之百解决幻读的问题。\u003c/p\u003e\n\u003cdiv class=\"image-package\"\u003e\n\u003cdiv class=\"image-container\" style=\"max-width: 140px; max-height: 150px;\"\u003e\n\u003cdiv class=\"image-container-fill\" style=\"padding-bottom: 107.13999999999999%;\"\u003e\u003c/div\u003e\n\u003cdiv class=\"image-view\" data-width=\"140\" data-height=\"150\"\u003e\u003cimg data-original-src=\"//upload-images.jianshu.io/upload_images/18375227-4ca72034844a348f\" data-original-width=\"140\" data-original-height=\"150\" data-original-format=\"image/gif\" data-original-filesize=\"140151\"\u003e\u003c/div\u003e\n\u003c/div\u003e\n\u003cdiv class=\"image-caption\"\u003eimage\u003c/div\u003e\n\u003c/div\u003e\n\u003cp\u003e举个例子：\u003c/p\u003e\n\u003cp\u003esession A：查询某条不存在的记录。\u003c/p\u003e\n\u003cp\u003esession B：新增该条不存在的记录，并提交事务。\u003c/p\u003e\n\u003cp\u003esession A：再次查询该条不存在的记录，是查询不出来的，但是如果我尝试修改该条记录，并提交，其实他是可以修改成功的。\u003c/p\u003e\n\u003ch3\u003e8.4  MVCC\u003c/h3\u003e\n\u003cp\u003e版本链：对于该记录的每次更新，都会将值放在一条undo日志中，算是该记录的一个旧版本，随着更新次数的增多，所有版本都会被roll_pointer属性连接成一个链表，即为版本链。\u003c/p\u003e\n\u003cp\u003ereadview：\u003c/p\u003e\n\u003cul\u003e\n\u003cli\u003e未提交读：因为可以读到未提交事务修改的记录，所以可以直接读取记录的最新版本就行\u003c/li\u003e\n\u003cli\u003e已提交读：每次读取之前都生成一个readview\u003c/li\u003e\n\u003cli\u003e可重复读：只有在第一次读取的时候才生成readview\u003c/li\u003e\n\u003cli\u003e可串行化：InnoDB涉及了加锁的方式来访问记录\u003c/li\u003e\n\u003c/ul\u003e\n\u003cblockquote\u003e\n\u003cp\u003e作者：学习Java的小姐姐\u003cbr\u003e\n原文链接：\u003ca href=\"https://links.jianshu.com/go?to=https%3A%2F%2Fjuejin.im%2Fpost%2F5dfc846051882512327a63b6\" target=\"_blank\"\u003ehttps://juejin.im/post/5dfc846051882512327a63b6\u003c/a\u003e\u003c/p\u003e\n\u003c/blockquote\u003e\n","voted_down":false,"rewardable":true,"show_paid_comment_tips":false,"share_image_url":"https://upload-images.jianshu.io/upload_images/18375227-8e1f04724ca6881f.png","slug":"2530d1185778","user":{"liked_by_user":false,"following_count":9407,"gender":0,"slug":"5872afeb0215","intro":"现在加群号：578486082 即可获取Java工程化、高性能及分布式、高性能、高架构。性能调...","likes_count":3825,"nickname":"java菲","badges":[],"total_fp_amount":"437666058468173681019","wordage":710906,"avatar":"https://upload.jianshu.io/users/upload_avatars/18375227/eb7d73d5-2b66-4fde-af00-2d3b2076ce8e.png","id":18375227,"liked_user":false},"likes_count":112,"paid_type":"free","show_ads":true,"paid_content_accessible":false,"total_fp_amount":"15304000000000000000","trial_open":false,"reprintable":true,"bookmarked":false,"wordage":7178,"featured_comments_count":0,"downvotes_count":0,"wangxin_trial_open":false,"commentable":true,"total_rewards_count":0,"id":58651933,"notebook":{"name":""},"description":"微服务架构之春招总结：SpringCloud、Docker、Dubbo与SpringBoot 欢迎大家关注我的专栏：【Java架构技术进阶】，里面有大量batj面试题集锦，还...","first_shared_at":1578386913,"views_count":5141,"notebook_id":37930225},"baseList":{"likeList":[],"rewardList":[]},"status":"success","statusCode":0},"user":{"isLogin":false,"userInfo":{}},"comments":{"list":[],"featuredList":[]}},"initialProps":{"pageProps":{"query":{"slug":"2530d1185778"}},"localeData":{"common":{"jianshu":"简书","diamond":"简书钻","totalAssets":"总资产{num}","diamondValue":" (约{num}元)","login":"登录","logout":"注销","register":"注册","on":"开","off":"关","follow":"关注","followBook":"关注连载","following":"已关注","cancelFollow":"取消关注","publish":"发布","wordage":"字数","audio":"音频","read":"阅读","reward":"赞赏","zan":"赞","comment":"评论","expand":"展开","prevPage":"上一页","nextPage":"下一页","floor":"楼","confirm":"确定","delete":"删除","report":"举报","fontSong":"宋体","fontBlack":"黑体","chs":"简体","cht":"繁体","jianChat":"简信","postRequest":"投稿请求","likeAndZan":"喜欢和赞","rewardAndPay":"赞赏和付费","home":"我的主页","markedNotes":"收藏的文章","likedNotes":"喜欢的文章","paidThings":"已购内容","wallet":"我的钱包","setting":"设置","feedback":"帮助与反馈","loading":"加载中...","needLogin":"请登录后进行操作","trialing":"文章正在审核中...","reprintTip":"禁止转载，如需转载请通过简信或评论联系作者。"},"error":{"rewardSelf":"无法打赏自己的文章哟~"},"message":{"paidNoteTip":"付费购买后才可以参与评论哦","CommentDisableTip":"作者关闭了评论功能","contentCanNotEmptyTip":"回复内容不能为空","addComment":"评论发布成功","deleteComment":"评论删除成功","likeComment":"评论点赞成功","setReadMode":"阅读模式设置成功","setFontType":"字体设置成功","setLocale":"显示语言设置成功","follow":"关注成功","cancelFollow":"取消关注成功","copySuccess":"复制代码成功"},"header":{"homePage":"首页","download":"下载APP","discover":"发现","message":"消息","reward":"赞赏支持","editNote":"编辑文章","writeNote":"写文章"},"note":{},"noteMeta":{"lastModified":"最后编辑于 ","wordage":"字数 {num}","viewsCount":"阅读 {num}"},"divider":{"selfText":"以下内容为付费内容，定价 ¥{price}","paidText":"已付费，可查看以下内容","notPaidText":"还有 {percent} 的精彩内容","modify":"点击修改"},"paidPanel":{"buyNote":"支付 ¥{price} 继续阅读","buyBook":"立即拿下 ¥{price}","freeTitle":"该作品为付费连载","freeText":"购买即可永久获取连载内的所有内容，包括将来更新的内容","paidTitle":"还没看够？拿下整部连载！","paidText":"永久获得连载内的所有内容, 包括将来更新的内容"},"book":{"last":"已是最后","lookCatalog":"查看连载目录","header":"文章来自以下连载"},"action":{"like":"{num}人点赞","collection":"收入专题","report":"举报文章"},"comment":{"allComments":"全部评论","featuredComments":"精彩评论","closed":"评论已关闭","close":"关闭评论","open":"打开评论","desc":"按时间倒序","asc":"按时间正序","disableText1":"用户已关闭评论，","disableText2":"与Ta简信交流","placeholder":"写下你的评论...","publish":"发表","create":" 添加新评论","reply":" 回复","restComments":"还有{num}条评论，","expandImage":"展开剩余{num}张图","deleteText":"确定要删除评论么？"},"collection":{"title":"被以下专题收入，发现更多相似内容","putToMyCollection":"收入我的专题"},"seoList":{"title":"推荐阅读","more":"更多精彩内容"},"sideList":{"title":"推荐阅读"},"wxShareModal":{"desc":"打开微信“扫一扫”，打开网页后点击屏幕右上角分享按钮"},"bookChapterModal":{"try":"试读","toggle":"切换顺序"},"collectionModal":{"title":"收入到我管理的专题","search":"搜索我管理的专题","newCollection":"新建专题","create":"创建","nothingFound":"未找到相关专题","loadMore":"展开查看更多"},"contributeModal":{"search":"搜索专题投稿","newCollection":"新建专题","addNewOne":"去新建一个","nothingFound":"未找到相关专题","loadMore":"展开查看更多","managed":"我管理的专题","recommend":"推荐专题"},"QRCodeShow":{"payTitle":"微信扫码支付","payText":"支付金额"},"rewardModal":{"title":"给作者送糖","custom":"自定义","placeholder":"给Ta留言...","choose":"选择支付方式","balance":"简书余额","tooltip":"网站该功能暂时下线，如需使用，请到简书App操作","confirm":"确认支付","success":"赞赏成功"},"payModal":{"payBook":"购买连载","payNote":"购买文章","promotion":"优惠券","promotionFetching":"优惠券获取中...","noPromotion":"无可用优惠券","promotionNum":"{num}张可用","noUsePromotion":"不使用优惠券","validPromotion":"可用优惠券","invalidPromotion":"不可用优惠券","total":"支付总额","tip1":"· 你将购买的商品为虚拟内容服务，购买后不支持退订、转让、退换，请斟酌确认。","tip2":"· 购买后可在“已购内容”中查看和使用。","success":"购买成功"},"reportModal":{"ad":"广告及垃圾信息","plagiarism":"抄袭或未授权转载","placeholder":"写下举报的详情情况（选填）","success":"举报成功"}},"currentLocale":"zh-CN","asPath":"/p/2530d1185778"}},"page":"/p/[slug]","query":{"slug":"2530d1185778"},"buildId":"QiPwfDmytgqS_hVmIsbVI","assetPrefix":"https://cdn2.jianshu.io/shakespeare"}</script><script nomodule="" src="https://cdn2.jianshu.io/shakespeare/_next/static/runtime/polyfills-83c9f0eea3aa0edfd89e.js"></script><script async="" data-next-page="/p/[slug]" src="https://cdn2.jianshu.io/shakespeare/_next/static/QiPwfDmytgqS_hVmIsbVI/pages/p/%5Bslug%5D.js"></script><script async="" data-next-page="/_app" src="https://cdn2.jianshu.io/shakespeare/_next/static/QiPwfDmytgqS_hVmIsbVI/pages/_app.js"></script><script src="https://cdn2.jianshu.io/shakespeare/_next/static/runtime/webpack-ad03007e443a25fa4d15.js" async=""></script><script src="https://cdn2.jianshu.io/shakespeare/_next/static/chunks/commons.6f517ba4cf1108d0202d.js" async=""></script><script src="https://cdn2.jianshu.io/shakespeare/_next/static/chunks/styles.4eb5917714db060a5a06.js" async=""></script><script src="https://cdn2.jianshu.io/shakespeare/_next/static/runtime/main-7d5f5d63b1c080a846ad.js" async=""></script></body></html>