#### HDFS读数据

简述：

```

```





详细：

```
1. 　客户端调用FileSystem 实例的open 方法，获得这个文件对应的输入流InputStream。
2. 　通过RPC 远程调用NameNode ，获得NameNode 中此文件对应的数据块保存位置，包括这个文件的副本的保存位置( 主要是各DataNode的地址) 。
3. 　获得输入流之后，客户端调用read 方法读取数据。选择最近的DataNode 建立连接并读取数据。
4. 　如果客户端和其中一个DataNode 位于同一机器(比如MapReduce 过程中的mapper 和reducer)，那么就会直接从本地读取数据。
5. 　到达数据块末端，关闭与这个DataNode 的连接，然后重新查找下一个数据块。
6. 　不断执行第2 - 5 步直到数据全部读完。
7. 　客户端调用close ，关闭输入流DFS InputStream
```





#### HDFS写数据

简述：

```

```





详细:

```
1、使用 HDFS 提供的客户端 Client，向远程的 namenode 发起 RPC 请求；
2、namenode 会检查要创建的文件是否已经存在，创建者是否有权限进行操作，成功则会 为文件创建一个记录，否则会让客户端抛出异常；
3、当客户端开始写入文件的时候，客户端会将文件切分成多个 packets，并在内部以数据队列“data queue（数据队列）”的形式管理这些 packets，并向 namenode 申请 blocks，获 取用来存储 replicas 的合适的 datanode 列表，列表的大小根据 namenode 中 replication 的设定而定；
4、开始以 pipeline（管道）的形式将 packet 写入所有的 replicas 中。客户端把 packet 以流的 方式写入第一个 datanode，该 datanode 把该 packet 存储之后，再将其传递给在此 pipeline 中的下一个 datanode，直到最后一个 datanode，这种写数据的方式呈流水线的形式；
5、最后一个 datanode 成功存储之后会返回一个 ack packet（确认队列），在 pipeline 里传递 至客户端，在客户端的开发库内部维护着"ack queue"，成功收到 datanode 返回的 ack packet 后会从"data queue"移除相应的 packet；
6、如果传输过程中，有某个 datanode 出现了故障，那么当前的 pipeline 会被关闭，出现故 障的 datanode 会从当前的 pipeline 中移除，剩余的 block 会继续剩下的 datanode 中继续 以 pipeline 的形式传输，同时 namenode 会分配一个新的 datanode，保持 replicas 设定的 数量；
7、客户端完成数据的写入后，会对数据流调用 close()方法，关闭数据流。
```

