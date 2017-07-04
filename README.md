polestar
========

JDBC是一种很常用的数据访问接口，Hive自带了Hive Server，可以接受Hive JDBC Driver的连接。实际 上，Hive JDBC Driver是将JDBC的请求转化为Thrift Call发给Hive Server，再由Hive Server将Job 启动起来。但Hive自带的Hive Server并不支持Security，默认会使用启动Hive Server的用户作为Job的owner提交到 Hadoop，造成安全漏洞。因此，我们自己开发了Hive Server的Security，解决了这个问题。

但在Hive Server的使用过程中，我们发现Hive Server并不稳定，而且存在内存泄漏。更严重的是由于Hive Server自身的设计缺陷，不能很好地应对并发访问的情况，所以我们现在并不推荐使用Hive JDBC的访问方式。

社区后来重新开发了Hive Server 2，解决了并发的问题，我们正在对Hive Server 2进行测试。

有一些同事，特别是BI的同事，不熟悉以CLI的方式使用Hive，希望Hive可以有个GUI界面。在上线Hive Server之后，我们调研了开源的SQL GUI Client——Squirrel，可惜使用Squirrel访问Hive存在一些问题。

办公网与线上环境是隔离的，在办公机器上运行的Squirrel无法连到线上环境的Hive Server。
Hive会返回大量的数据，特别是当用户对于Hive返回的数据量没有预估的情况下，Squirrel会吃掉大量的内存，然后Out of Memory挂掉。
Hive JDBC实现的JDBC不完整，导致Squirrel的GUI中只有一部分功能可用，用户体验非常差。
基于以上考虑，我们自己开发了Hive Web，让用户通过浏览器就可以使用Hive。Hive Web最初是作为大众点评第一届Hackathon的一个项目被开发出来的，技术上很简单，但获得了良好的反响。现在Hive Web已经发展成了一个RESTful的Service，称为Polestar（https://github.com/dianping/polestar）。
##![Polestar的结构](http://cms.csdnimg.cn/article/201312/18/52b14ba353bc8_middle.jpg?_=27667)

图3是Polestar的结构图。目前Hive Web只是一个GWT的前端，通过HAProxy将RESTfull Call分发到执行引擎Worker执行。Worker将自身的状态保存在MySQL，将数据保存在HDFS，并使用JSON返回数据或数据在HDFS的 路径。我们还将Shark与Hive Web集成到了一起，用户可以选择以Hive或者Shark执行Query。

一开始我们使用LZO作为存储格式，使大文件可以在MapReduce处理中被切分，提高并行度。但LZO的压缩比不够高，按照我们的测试，Lzo压缩的文件，压缩比基本只有Gz的一半。

经过调研，我们将默认存储格式替换成RCFile，在RCFile内部再使用Gz压缩，这样既可保持文件可切分的特性，同时又可获得Gz的高压缩比，而且因 为RCFile是一种列存储的格式，所以对于不需要的字段就不用从I/O读入，从而提高了性能。图4显示了将Nginx数据分别用Lzo、 RCFile+Gz、RCFfile+Lzo压缩，再不断增加Select的Column数，在Hive上消耗的CPU时间（越小越好）
