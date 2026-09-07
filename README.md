# 合同服务 Compose 部署

本仓库统一构建和运行前端 Nginx、后端 FastAPI 与单节点 Elasticsearch。前后端源码仍由各自仓库维护；不包含 vLLM 模型服务，也不自动迁移原开发环境数据。

---

## 目录与源码

```text
contract-service-deploy/
  docker-compose.yml
  .env.example                 可提交的完整环境模板
  .env                         本机配置与模型密钥，不提交
  user.example.yaml            用户结构示例
  user.yaml                    实际用户配置，不提交
  elasticsearch/Dockerfile     带 SmartCN 的 ES 镜像
  contract-service-be/         可选：独立克隆的后端仓库
  contract-service-fe/         可选：独立克隆的前端仓库
```

示例配置按源码位于本目录下组织；将两个仓库克隆到对应目录即可。它们已被部署仓库忽略，不会重复提交。各仓库的版本需要自行记录和固定。

当前本机 `.env` 已改为引用同级现有前后端，没有移动源码：

```dotenv
BACKEND_SOURCE=../contract-service-be
FRONTEND_SOURCE=../contract-service-fe
```

`SOURCE` 相对于 Compose 文件。前后端使用各自项目根目录的 `Dockerfile` 和 `.dockerignore`，deploy 不再重复维护。前端 Nginx 模板也由前端项目维护，通过 `BACKEND_UPSTREAM=backend:20000` 接入后端。ES 没有独立业务源码项目，其镜像配置保留在 `elasticsearch/`。构建使用当前工作区内容，包括未提交修改；源码更新后须重新构建镜像。

前端使用 `package-lock.json` 与 `npm ci`；后端沿用源码的 `requirements.txt` 版本范围。当前基础镜像使用版本标签，尚未锁定镜像 digest 和全部 Python 间接依赖；正式发布时应固定并记录验证过的镜像版本，避免把重新构建视为完全相同的产物。

---

## 首次启动

要求 Docker Engine / Docker Desktop、支持长格式 bind 挂载的现代 Docker Compose，以及访问镜像仓库、PyPI、npm 和 Elastic 插件下载站的网络。ES、前后端构建和模型调用都需要足够内存。Linux 主机需按所用 ES 版本满足 `vm.max_map_count` 等启动检查；失败时查看 ES 日志，不关闭检查。

新机器第一次配置（已有文件时不要覆盖）：

```bash
cp .env.example .env
cp user.example.yaml user.yaml
chmod 600 .env user.yaml
```

编辑源码路径、模型地址和密钥，并替换用户示例密钥。当前本机已从后端复制 `.env` 和用户配置；保留原用户权限，没有新增管理员。

```bash
docker compose config --quiet
docker compose up -d --build
docker compose ps
curl --fail http://127.0.0.1:17080/contract/api/health
```

浏览器打开 `http://127.0.0.1:17080/contract/`。前端构建固定使用 `/contract/`；同源 API 使用 `/contract/api/`，支持 History 路由刷新、PDF.js 静态资源及不缓冲的 SSE。上传上限默认为 100 MB，可在 `.env` 设置 `CLIENT_MAX_BODY_SIZE`，重新创建前端容器后生效。

前后端共用宿主机监听地址 `BIND_ADDRESS`，默认 `127.0.0.1`。`FRONTEND_PORT=17080` 映射到前端容器 80，`BACKEND_PORT=17000` 映射到后端容器 20000，供映射服务器直接转发。代理位于其他机器时，将公共监听地址改为指定内网 IP 或 `0.0.0.0`，并限制访问来源；同机代理可保持回环地址。两个宿主机端口必须不同且未被占用。

映射服务器的上游设置为 `http://后端宿主机内网IP:BACKEND_PORT`，保留 `/contract/api/` 路径、Authorization 请求头和 SSE 流式响应，关闭响应缓冲并配置足够的读取超时。前端容器仍通过 `backend:20000` 访问后端，不受宿主机端口修改影响。变更端口后需重新创建后端容器，不能只执行 restart。

宿主机防火墙或安全组应仅允许映射服务器访问后端端口。公网入口需增加 HTTPS、访问控制及上传容量限制，不应直接暴露后端明文 HTTP。ES 不发布宿主机端口，当前关闭认证，仅适用于受信任 Docker 主机与专用网络；本配置不是完整生产安全方案。

模型服务在 Compose 外运行：容器内 `127.0.0.1` 指向容器自身。宿主机模型可使用 `host.docker.internal`，远程模型直接配置可达地址。Linux 已设置 host-gateway 映射，但模型仍需监听容器可达网卡并允许防火墙访问。模型健康不包含在后端 health 检查中，应额外通过真实合同任务验证。

---

## 数据与初始化

| 卷 / 挂载 | 容器位置 | 用途 |
| --- | --- | --- |
| `backend-data` | `/workspace/data` | 整个 data：定义、PDF、SQLite（含 WAL/SHM）及其他数据子目录 |
| `elasticsearch-data` | `/usr/share/elasticsearch/data` | ES 索引数据 |
| `./user.yaml`（只读） | `/workspace/data/user/users.yaml` | 顶层审核用户配置 |

两个命名卷由 Docker 管理，实际名称以 `COMPOSE_PROJECT_NAME` 为前缀，并非 deploy 下的普通目录。后端整体持久化 `/workspace/data`，包括现有和未来新增的数据子目录。`user.yaml` 仍作为子路径只读挂载，覆盖容器中的用户配置视图；真实用户配置应单独备份，不会被写入数据卷。

首次挂载空的命名卷时，Docker 默认将镜像内该目录的内容复制到卷中，因此初始 `data/definition` 来自镜像。已有非空卷不会随镜像重建自动刷新定义；更新类别、字段或检索指南时，须先备份、停写，再显式更新卷内对应定义并重建后端容器。不要为更新定义清空整个卷，也不要覆盖合同和数据库。参见 [Docker 卷初始化规则](https://docs.docker.com/engine/storage/volumes/#mounting-a-volume-over-existing-data)。镜像仍不包含真实用户、`.env`、历史合同、SQLite 或 Git 内容；整体挂载不代表自动复制开发环境的全部 data。

旧方案的 `backend-contracts`、`backend-abstract` 卷不会自动合并到 `backend-data`。若曾使用旧方案入库，应先停写并备份，再迁移 PDF 和完整 SQLite 数据至新卷对应子目录；本次仅修改配置，不删除旧卷、不迁移数据。

ES 镜像构建时安装同版本 SmartCN 插件，不需要 MySQL 式初始化 SQL。后端等待 ES 健康后启动，自动创建或检查索引 mapping、初始化 SQLite 并同步类别目录；不会自动生成示例合同。前端等待后端健康后启动。

这是一套全新数据环境，不自动复用原 `contract-service-elasticsearch-data` 卷，也不复制开发 SQLite/PDF。已有合同需要成套迁移 ES、SQLite 与 PDF，并在停写及备份后执行；只复制其中一项会导致数据不一致。

```bash
docker compose stop
docker compose start
docker compose down
```

以上操作保留命名卷。**不要执行 `docker compose down --volumes`，除非确认要永久清空本部署数据。** 更改 Compose 项目名会切换卷，不代表原数据被删除。备份必须覆盖整个 `backend-data`、ES 快照和独立的 `user.yaml`；SQLite 与 PDF 应在停写状态下一致备份，不要在运行时单独拷贝数据库主文件。

---

## 顶层用户配置与更新

`user.yaml` 的结构与后端原 `data/user/users.yaml` 一致，包含 `users` 列表，每项为 `name`、`secret_key`、`permission_level`。等级 1 可新增/查看/删除，2 可新增/查看，3 仅查看。

挂载覆盖容器内配置视图，不会修改开发源码里的用户文件。缺少 `user.yaml` 时 Compose 报错，不创建同名目录。用户在启动期加载，不支持热更新；为兼容编辑器原子替换文件，修改后重新创建后端容器：

```bash
docker compose up -d --force-recreate backend
```

后端单 worker、不启用热重载。重建或重启会清空登录免登码、未入库任务及其内存 PDF；先完成或取消任务，再更新。已入库数据保存在卷中。

修改 `.env` 后需重新创建受影响的容器；`docker compose restart` 不重新加载环境变量。更新前后端源码或前端 Nginx 模板后执行 `docker compose up -d --build`。前端在启动时解析后端地址，单独重建后端导致 IP 改变时，应同时重新创建前端容器。

---

## 排查与提交

```bash
docker compose logs --tail=100 elasticsearch backend frontend
docker compose exec frontend nginx -t
docker compose exec elasticsearch curl -fsS 'http://localhost:9200/_cat/plugins?v'
docker compose config --quiet
```

启动卡在健康等待时先看 ES，再看后端配置和索引校验。SSE 若经过外层反向代理，该层也需关闭响应缓冲并配置足够超时。构建依赖下载失败属于网络或仓库问题，不能用已有服务健康代替新镜像验证。

提交部署仓库时只提交环境及用户模板、Compose、ES Dockerfile 和本文档；前后端 Dockerfile 分别随各自项目提交。不要提交 `.env`、`user.yaml`、源码副本或备份。不要将 `docker compose config` 完整输出粘贴到工单：它会展开模型密钥。检查语法使用 `--quiet`。
