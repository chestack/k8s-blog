## containerd 存储机制

containerd存储相关 主要有两个可配置的目录:
 - root, 默认使用 /var/lib/containerd/, 存储需要持久化的数据，包括: 容器镜像, containers的metadata数据，snapshot
 - state, 默认使用 /var/run/containerd, 利用 Linux /run 目录使用tmfs的文件系统的特性, 存储一些ephemeral data，包括: sockets, pids, container config.json这些状态数据

containerd 是如何管理 静态的镜像，运行中的容器的呢，其中关键一环是 snapshot. 这里主要记录下 对containerd snapshot的理解。

### Pull Image

镜像是分层方式组织且每一层是一个压缩包, ctr image pull的过程会把每一层的内容存储到 /var/lib/containerd/io.containerd.content.v1.content.

镜像的每一层，对应containerd里的一个content, 下面以一个nginx image为例
```
[root@node-2 ~]# ctr image ls
REF TYPE DIGEST SIZE PLATFORMS LABELS
[root@node-2 ~]# ctr content ls
DIGEST	SIZE	AGE	LABELS
[root@node-2 ~]# ctr snapshot ls
KEY PARENT KIND
[root@node-2 ~]#

[root@node-2 ~]# ctr i pull -k hub.easystack.io/production/nginx:stable
hub.easystack.io/production/nginx:stable:                                         resolved       |++++++++++++++++++++++++++++++++++++++|
index-sha256:f12c9983c4f114d981e0790c2664c370d113598f6ce7476bedbc0b6099fd8ee2:    done           |++++++++++++++++++++++++++++++++++++++|
manifest-sha256:aed492c4dc93d2d1f6fe9a49f00bc7e1863d950334a93facd8ca1317292bf6aa: done           |++++++++++++++++++++++++++++++++++++++|
config-sha256:d6c9558ba4456741fc4ee304e1a75a561e1c8d92f5107a715b6224bb7844f507:   done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:7976249980ef3e400b4e534a7dc808932173bef1d5c1c7a6b6ea79493692056a:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:8f66aa6726b2b2e2f14df11c13ea6d3ed62a9f228b7370b8fe6ab47430e57ed5:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:d86da7454448aa7f7de90e8ed24ff8264d37680599e72e9491a0f418d0521714:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:5eb5b503b37671af16371272f9c5313a3e82f1d0756e14506704489ad9900803:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:c004cabebe769668176089b938a4ee368fadb90fe0e01fc7a6aefa47850c1bb8:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:cdfeb356c0299c1c464d221a245d0c28967475dd5f9cf85bbe2e23470c4a74d8:    done           |++++++++++++++++++++++++++++++++++++++|
elapsed: 0.3 s                                                                    total:   0.0 B (0.0 B/s)
unpacking linux/amd64 sha256:f12c9983c4f114d981e0790c2664c370d113598f6ce7476bedbc0b6099fd8ee2...

DIGEST									SIZE	AGE		LABELS
sha256:5eb5b503b37671af16371272f9c5313a3e82f1d0756e14506704489ad9900803	31.37MB	37 minutes	containerd.io/distribution.source.hub.easystack.io=production/nginx,containerd.io/uncompressed=sha256:7d0ebbe3f5d26c1b5ec4d5dbb6fe3205d7061f9735080b0162d550530328abd6
sha256:7976249980ef3e400b4e534a7dc808932173bef1d5c1c7a6b6ea79493692056a	895B	37 minutes	containerd.io/distribution.source.hub.easystack.io=production/nginx,containerd.io/uncompressed=sha256:dc78c3d0e91713e0b45a209df1509fb96d4f6a4f3099f4dfbc9e9ae71b143e43
sha256:8f66aa6726b2b2e2f14df11c13ea6d3ed62a9f228b7370b8fe6ab47430e57ed5	667B	37 minutes	containerd.io/distribution.source.hub.easystack.io=production/nginx,containerd.io/uncompressed=sha256:8fa2ccbce0c21c1a0140ce0f2db6b92f92c16f9ff00e4c8a1daaea548799b698
sha256:aed492c4dc93d2d1f6fe9a49f00bc7e1863d950334a93facd8ca1317292bf6aa	1.57kB	37 minutes	containerd.io/distribution.source.hub.easystack.io=production/nginx,containerd.io/gc.ref.content.config=sha256:d6c9558ba4456741fc4ee304e1a75a561e1c8d92f5107a715b6224bb7844f507,containerd.io/gc.ref.content.l.0=sha256:5eb5b503b37671af16371272f9c5313a3e82f1d0756e14506704489ad9900803,containerd.io/gc.ref.content.l.1=sha256:cdfeb356c0299c1c464d221a245d0c28967475dd5f9cf85bbe2e23470c4a74d8,containerd.io/gc.ref.content.l.2=sha256:d86da7454448aa7f7de90e8ed24ff8264d37680599e72e9491a0f418d0521714,containerd.io/gc.ref.content.l.3=sha256:7976249980ef3e400b4e534a7dc808932173bef1d5c1c7a6b6ea79493692056a,containerd.io/gc.ref.content.l.4=sha256:8f66aa6726b2b2e2f14df11c13ea6d3ed62a9f228b7370b8fe6ab47430e57ed5,containerd.io/gc.ref.content.l.5=sha256:c004cabebe769668176089b938a4ee368fadb90fe0e01fc7a6aefa47850c1bb8
sha256:c004cabebe769668176089b938a4ee368fadb90fe0e01fc7a6aefa47850c1bb8	1.394kB	37 minutes	containerd.io/distribution.source.hub.easystack.io=production/nginx,containerd.io/uncompressed=sha256:b1073b41766d15d7835ea97e5eaf7f2f8a06b7fd1549ecbc2c9697813ede853a
sha256:cdfeb356c0299c1c464d221a245d0c28967475dd5f9cf85bbe2e23470c4a74d8	25.34MB	37 minutes	containerd.io/distribution.source.hub.easystack.io=production/nginx,containerd.io/uncompressed=sha256:f7d96e665ae1a81d81e18e2503b2341bec8491606ffa6e2b48480a8e741dfab7
sha256:d6c9558ba4456741fc4ee304e1a75a561e1c8d92f5107a715b6224bb7844f507	7.639kB	37 minutes	containerd.io/distribution.source.hub.easystack.io=production/nginx,containerd.io/gc.ref.snapshot.overlayfs=sha256:055624127108c9595ef76afb2fdcba8286c37684eeee175d30c59a63bf3cf90c
sha256:d86da7454448aa7f7de90e8ed24ff8264d37680599e72e9491a0f418d0521714	602B	37 minutes	containerd.io/distribution.source.hub.easystack.io=production/nginx,containerd.io/uncompressed=sha256:a64a30dea1c4b926a45abed0728327d079d2c5fa4a7b7e6aa2d06136783a27a2
sha256:f12c9983c4f114d981e0790c2664c370d113598f6ce7476bedbc0b6099fd8ee2	772B	37 minutes	containerd.io/distribution.source.hub.easystack.io=production/nginx,containerd.io/gc.ref.content.m.0=sha256:aed492c4dc93d2d1f6fe9a49f00bc7e1863d950334a93facd8ca1317292bf6aa,containerd.io/gc.ref.content.m.1=sha256:1cd8d6ec1857f28447dd6197ce5c704d8bdebbac2a1532829aa22843dc9e546c

[root@node-2 sha256]# ctr snapshot ls
KEY                                                                     PARENT                                                                  KIND
sha256:055624127108c9595ef76afb2fdcba8286c37684eeee175d30c59a63bf3cf90c sha256:51cc5e700f73e1d4ae88e0ca8d094743ac39ab9db881bd2d6f04c87a0f4341e6 Committed
sha256:51cc5e700f73e1d4ae88e0ca8d094743ac39ab9db881bd2d6f04c87a0f4341e6 sha256:82b7f14ee2763df71987ec4d5ce5ad846e91cd59204fdee6467a634619c263ad Committed
sha256:5c3da4d189e1edff6a1babc21a57332a6a1ee09e79bbfc30fd1bad0a8096ce89 sha256:7d0ebbe3f5d26c1b5ec4d5dbb6fe3205d7061f9735080b0162d550530328abd6 Committed
sha256:7d0ebbe3f5d26c1b5ec4d5dbb6fe3205d7061f9735080b0162d550530328abd6                                                                         Committed
sha256:82b7f14ee2763df71987ec4d5ce5ad846e91cd59204fdee6467a634619c263ad sha256:d1217a4a7e17bc7fefe318f17a84a3649ab716a7a370bc992bc8fe8d0ef04c9d Committed
sha256:d1217a4a7e17bc7fefe318f17a84a3649ab716a7a370bc992bc8fe8d0ef04c9d sha256:5c3da4d189e1edff6a1babc21a57332a6a1ee09e79bbfc30fd1bad0a8096ce89 Committed
```

从上面的例子看
- pull image 会为每一层layer 创建一个content, 通过sha256能清晰看到image 和 content之间 一一对应关系.
- 除去image中 manifest、index、config, 每一层镜像内容会创建一个snapshot. content 和 snapshot之间应该也是一对一关系, 从content 和 snapshot的命令结果看，只能看到最上面的snapshot的 sha256值==镜像一个layer的 uncompressed的值.

为什么image的每一个layer 都要再存储成一份 snapshot呢？
- image layer(content)是压缩格式的, 容器运行依赖 rootfs内容可以mount为文件系统来用，压缩格式的没法直接使用
- 容器rootfs是多层layer只读Union mount联合挂载, 且最上面加上一个可写的读写层，这样的层级结构以及COPY-ON-WRITE特性，保证容器镜像的immutable 以及 读写性能

以上功能需求, 都是需要文件系统支持的，containerd将其抽象为snapshot插件机制来实现，根据不同的文件系统对应着不同的snapshot插件，默认是overlayfs
```
[root@node-2 work]# ctr plugin ls | grep snapshot
io.containerd.snapshotter.v1           aufs                     linux/amd64    skip
io.containerd.snapshotter.v1           btrfs                    linux/amd64    skip
io.containerd.snapshotter.v1           blockfile                linux/amd64    skip
io.containerd.snapshotter.v1           native                   linux/amd64    ok
io.containerd.snapshotter.v1           overlayfs                linux/amd64    ok
io.containerd.snapshotter.v1           devmapper                linux/amd64    error
io.containerd.snapshotter.v1           zfs                      linux/amd64    skip
```

### Run container

基于上面 image --> content --> snapshot, containerd还需要做哪些工作才能启动一个container呢？

```
[root@node-2 work]# ctr run -d -t hub.easystack.io/production/nginx:stable nginx0
[root@node-2 work]# ctr snapshot ls
KEY                                                                     PARENT                                                                  KIND
nginx0                                                                  sha256:055624127108c9595ef76afb2fdcba8286c37684eeee175d30c59a63bf3cf90c Active
sha256:055624127108c9595ef76afb2fdcba8286c37684eeee175d30c59a63bf3cf90c sha256:51cc5e700f73e1d4ae88e0ca8d094743ac39ab9db881bd2d6f04c87a0f4341e6 Committed
sha256:51cc5e700f73e1d4ae88e0ca8d094743ac39ab9db881bd2d6f04c87a0f4341e6 sha256:82b7f14ee2763df71987ec4d5ce5ad846e91cd59204fdee6467a634619c263ad Committed
sha256:5c3da4d189e1edff6a1babc21a57332a6a1ee09e79bbfc30fd1bad0a8096ce89 sha256:7d0ebbe3f5d26c1b5ec4d5dbb6fe3205d7061f9735080b0162d550530328abd6 Committed
sha256:7d0ebbe3f5d26c1b5ec4d5dbb6fe3205d7061f9735080b0162d550530328abd6                                                                         Committed
sha256:82b7f14ee2763df71987ec4d5ce5ad846e91cd59204fdee6467a634619c263ad sha256:d1217a4a7e17bc7fefe318f17a84a3649ab716a7a370bc992bc8fe8d0ef04c9d Committed
sha256:d1217a4a7e17bc7fefe318f17a84a3649ab716a7a370bc992bc8fe8d0ef04c9d sha256:5c3da4d189e1edff6a1babc21a57332a6a1ee09e79bbfc30fd1bad0a8096ce89 Committed
```

能看到, 新增了一个和container名字一样的 snapshot ‘nginx0’, 且parent是镜像最后一层的snapshot.

另外KIND 一列, ‘nginx0’的状态是 Active, Active 和 Commited的关系可以参照 引用部分的第一个链接，这里可以简单和git的commit状态做类比，Active是读写中的状态，Committed是修改保存后的状态。

```
[root@node-2 work]# mount -l | grep nginx0
overlay on /run/containerd/io.containerd.runtime.v2.task/default/nginx0/rootfs type overlay (rw,relatime,lowerdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/4690/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/4689/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/4688/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/4687/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/4686/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/4685/fs,upperdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/4692/fs,workdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/4692/work)
```
从 nginx0的mount信息中，可以看出
- lowerdir, 这是只读的底层文件系统，lowerdir可以指定多个, 其间是用冒号分隔开, 每一个对应image的每一层的snapshot.
- upperdir, 这是可写的层，位于rootfs之上，容器的的所有写操作都会被记录在这层snapshot.

lowerdir、upperdir、workdir 一起组成容器的rootfs.


- [Take Control of your Filesystems with containerd’s Snapshotters](https://www.youtube.com/watch?v=ebynv9XxrLk)
- [Containerd Content-flow](https://github.com/containerd/containerd/blob/main/docs/content-flow.md)
- [What is containerd snapshotter](https://dev.to/napicella/what-is-a-containerd-snapshotters-3eo2)