# JuiceFS 支持的元数据存储引擎

通过阅读 [JuiceFS 的技术架构](architecture.md) 和 [JuiceFS 如何存储文件](how_juicefs_store_files.md)，你会了解到 JuiceFS 被设计成了一种将数据和元数据独立存储的架构，通常来说，数据被存储在以对象存储为主的云存储中，而数据所对应的元数据则被存储在独立的数据库中。

## 元数据存储引擎

元数据和数据同样至关重要，元数据中记录着每一个文件的详细信息，名称、大小、权限、位置等等。特别是这种数据与元数据分离存储的文件系统，元数据的读写性能直接决定了文件系统实际的性能表现。

JuiceFS 的元数据存储采用了多引擎设计。为了打造一个超高性能的云原生文件系统，JuiceFS 最先支持的是运行在内存上的键值数据库—— [Redis](https://redis.io)，这使得 JuiceFS 拥有十倍于 Amazon [EFS](https://aws.amazon.com/efs) 和 [S3FS](https://github.com/s3fs-fuse/s3fs-fuse) 的性能表现，[查看测试结果](benchmark.md)。

通过与社区用户积极互动，我们发现很多应用场景并不绝对依赖高性能，有时用户只是想临时找到一个方便的工具在云上可靠的迁移数据，或者只是想更简单的把对象存储挂载到本地小规模的使用。因此，JuiceFS 陆续开放了对 MySQL/MariaDB、SQLite 等更多数据库的支持。

**但需要特别注意的是**，在使用 JuiceFS 文件系统的过程中，不论你选择哪种数据库存储元数据，请 **务必确保元数据的安全**！元数据一旦损坏或丢失，将直接导致对应数据彻底损坏或丢失，严重的可能直接导致整个文件系统发生损毁。

## Redis

> [Redis](https://redis.io/) 是一款开源的（BSD许可）基于内存的键值存储系统，常被用作数据库、缓存和消息代理。

你可以很容易的在云计算平台购买到各种配置的云 Redis 数据库，但如果你只是想要快速评估 JuiceFS，可以使用 Docker 快速的在本地电脑上运行一个 Redis 数据库实例：

```shell
$ sudo docker run -d --name redis \
	-v redis-data:/data \
	-p 6379:6379 \
	--restart unless-stopped \
	redis redis-server --appendonly yes
```

> **注意**：以上命令将 Redis 的数据持久化在 Docker 的 redis-data 数据卷当中，你可以按需修改数据持久化的存储位置。

> **安全提示**：以上命令创建的 Redis 数据库实例没有启用身份认证，且暴露了主机的 `6379` 端口，如果你要通过互联网访问这个数据库实例，请参考 [Redis Security](https://redis.io/topics/security) 中的建议。

### 创建文件系统

使用 Redis 作为元数据存储引擎时，通常使用以下格式访问数据库：

```shell
redis://<IP or Domain name>:6379
```

例如，以下命令将创建一个名为 `pics` 的 JuiceFS 文件系统，使用 Redis 中的 `1` 号数据库存储元数据：

```shell
$ juicefs format \
	--storage minio \
	--bucket http://192.168.1.6:9000/jfs \
	--access-key minioadmin \
	--secret-key minioadmin \
	redis://192.168.1.6:6379/1 \
	pics
```

### 挂载文件系统

```shell
$ sudo juicefs mount -d redis://192.168.1.6:6379/1 /mnt/jfs
```

> **提示**：如果你计划在多台服务器上共享使用同一个 JuiceFS 文件系统，你必须确保 Redis 数据库可以被每一台要挂载文件系统的服务器访问到。

如果你有兴趣，可以查看 [Redis 最佳实践](redis_best_practices.md)。

## MySQL

> MySQL 是世界上最受欢迎的开源关系型数据库之一，常被作为 Web 应用程序的首选数据库。

你可以很容易的在云计算平台购买到各种配置的云 MySQL 数据库，但如果你只是想要快速评估 JuiceFS，可以使用 Docker 快速的在本地电脑上运行一个 MySQL 数据库实例：

```shell
$ sudo docker run -d --name mysql \
	-p 3306:3306 \
	-v mysql-data:/var/lib/mysql \
	-e MYSQL_ROOT_PASSWORD=password \
	-e MYSQL_DATABASE=juicefs \
	-e MYSQL_USER=user \
	-e MYSQL_PASSWORD=password \
	--restart unless-stopped \
	mysql
```

为了方便你快速开始测试，以上代码直接通过 `MYSQL_ROOT_PASSWORD`、`MYSQL_DATABASE`、`MYSQL_USER`、`MYSQL_PASSWORD` 环境变量分别设置了 root 用户的密码、名为 juicefs 的数据库以及用于管理该数据库的用户和密码，你可以根据实际需要调整上述环境变量对应的值，也可以 [点此查看](https://hub.docker.com/_/mysql)  Docker 中创建 MySQL 容器的更多内容。

> **注意**：以上命令将 MySQL 的数据持久化在了 Docker 的 mysql-data 数据卷当中，你可以按需修改数据持久化的存储位置。出于测试方便同时开放了 3306 端口，请勿将该实例用于关键数据的存储。

### 创建文件系统

使用 MySQL 作为元数据存储引擎时，通常使用以下格式访问数据库：

```shell
mysql://<username>:<password>@(<IP or Domain name>:3306)/<database-name>
```

例如：

```shell
$ juicefs format --storage minio \
	--bucket http://192.168.1.6:9000/jfs \
	--access-key minioadmin \
	--secret-key minioadmin \
	mysql://user:password@(192.168.1.6:3306)/juicefs \
	pics
```

更多 MySQL 数据库的地址格式示例，[点此查看](https://github.com/Go-SQL-Driver/MySQL/#examples)。

### 挂载文件系统

```shell
$ sudo juicefs mount -d mysql://user:password@(192.168.1.6:3306)/juicefs /mnt/jfs
```

## MariaDB

> MariaDB 也是最流行的关系型数据库之一，它是 MySQL 的一个开源分支，由 MySQL 原始开发者维护并保持开源。

因为 MariaDB 与 MySQL 高度兼容，因此在使用上也没有任何差别，比如在本地的 Docker 上创建一个实例，只是改换个名称和镜像而已，各项参数和设置完全一致：

```shell
$ sudo docker run -d --name mariadb \
	-p 3306:3306 \
	-v mysql-data:/var/lib/mysql \
	-e MYSQL_ROOT_PASSWORD=password \
	-e MYSQL_DATABASE=juicefs \
	-e MYSQL_USER=user \
	-e MYSQL_PASSWORD=password \
	--restart unless-stopped \
	mariadb
```

在创建和挂载文件系统时，则保持 MySQL 的语法，例如：

```shell
$ juicefs format --storage minio \
	--bucket http://192.168.1.6:9000/jfs \
	--access-key minioadmin \
	--secret-key minioadmin \
	mysql://user:password@(192.168.1.6:3306)/juicefs \
	pics
```

## SQLite

> [SQLite](https://sqlite.org) 是一款被广泛使用的小巧、快速、单文件、可靠、全功能的 SQL 数据库引擎。

SQLite 数据库只有一个文件，创建和使用都非常灵活，用它作为 JuiceFS 元数据存储引擎时无需提前创建数据库文件，可以直接创建文件系统：

```shell
$ juicefs format --storage minio \
	--bucket https://192.168.1.6:9000/jfs \
	--access-key minioadmin \
	--secret-key minioadmin \
	sqlite3://my-jfs.db \
	pics
```

执行以上命令会自动在当前目录中创建名为 `my-jfs.db` 的数据库文件，请 **务必妥善保管** 这个数据库文件！

挂载文件系统：

```shell
$ sudo juicefs mount -d sqlite3://my-jfs.db
```

如果数据库不在当前目录，则需要指定数据库文件的绝对路径，比如：

```shell
$ sudo juicefs mount -d sqlite3:///home/herald/my-jfs.db /mnt/jfs/
```

### 注意

由于 SQLite 是一款单文件数据库，在不做特殊共享设置的情况下，通常只有数据库所在的主机可以访问它。因此，SQLite 数据库更适合单机使用，对于多台服务器共享同一文件系统的情况，建议使用 Redis 或 MySQL 等数据库。

## TiDB

即将推出......

## TiKV

即将推出......

## PostgreSQL

即将推出......