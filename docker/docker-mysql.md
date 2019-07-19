### Docker MySQL



#### 外部导入数据到Docker容器

1. 从主机复制.sql文件到容器

   ```shell
   sudo docker cp host_path cotainerId:container_path
   ```

   从容器复制到主机

   ```shell
   sudo docker cp cotainerId:container_path host_path
   ```

2. 进入数据库

   ```shell
   docker exec -it containerId /bin/bash
   ```

3. 导入容器内部文件

   ```shell
   source /var/xx.sql
   ```

