# Docker 基础要点

- 镜像（image）是只读模板，容器（container）是镜像的运行实例。
- `docker build -t name .` 根据 `Dockerfile` 构建镜像。
- `docker run -d -p 8080:80 name` 后台启动并映射端口。
- 数据用 volume 持久化，否则容器删除后数据丢失。
- 多服务用 `docker compose` 编排，配置文件是 `docker-compose.yml`。
