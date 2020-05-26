---
title: Harbor版本无缝升级
date: 2020-05-26 19:49:00
tags:
categories:
	- Kubernetes
---



### 备份


```

cd /data/soft/harbor

docker-compose down

cd /data/soft

mv harbor /path/to/backup/harbor_1.8.4

cp -r /data/database /path/to/backup/harbor_1.8.4/database

```

### 准备更新

```
docker pull goharbor/harbor-migrator:v1.8.6

# 获取相应的版本，这里更新到1.8.6

wget https://github.com/goharbor/harbor/releases/download/v1.8.6/harbor-offline-installer-v1.8.6.tgz

# 拉取相应版本的迁移工具

tar zxf harbor-offline-installer-v1.8.6.tgz

docker image load -i harbor/harbor.v1.8.6.tar.gz

# 获取同版本的迁移工具

docker pull goharbor/harbor-migrator:v1.8.6

# 更新配置文件

docker run -it --rm -v /path/to/backup/harbor_1.8.4/harbor.yml:/harbor-migration/harbor-cfg/harbor.yml goharbor/harbor-migrator:v1.8.6 --cfg up

# 或者直接copy配置文件,版本差距小，版本差距大时，还是建议使用以上操作

cp /path/to/backup/harbor_1.8.4/harbor.yml /data/soft/harbor/
```

### 更新

```
cd /data/soft/harbor

./install.sh
```


### 回滚(如果需要)

```
# 停止现行版本

cd /data/soft/harbor

docker-compose down

# 删除现行版本

cd /data/soft

rm -rf harbor

# 还原旧版本

cp -r /path/to/backup/harbor_older_version ./harbor

cd harbor

./install.sh
```
### Reference


https://www.jianshu.com/p/8c7bfb2f807e