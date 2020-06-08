## Minio分布式文件系统搭建



### 环境说明

用虚拟搭建

   SvrMinio01   192.168.1.58

   SvrMinio02   192.168.1.59

   SvrMinio03   192.168.1.60

CentOS Linux release 7.5



### 安装

三台虚拟机分别安装

下载minio二进制文件

```bash
cd /usr/local/bin
#server
wget https://dl.min.io/server/minio/release/linux-amd64/minio
#client
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x minio mc
```



# How can I extend the storage size after deployment? 

```
https://github.com/minio/minio/issues/4364s
```

# Minio-Server Scale UP and Down documentation

```
https://github.com/minio/minio/issues/5901
```

# Need for a document explaining scaling of minio #4366

```
https://github.com/minio/minio/issues/4366
```



# Modern Data Lake with Minio : Part 1

```
https://blog.minio.io/modern-data-lake-with-minio-part-1-716a49499533
```

