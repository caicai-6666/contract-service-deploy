# 现象合同智能管理平台 · 部署

本仓库统一构建和运行前端 Nginx、后端 FastAPI、单节点 Elasticsearch 与 Neo4j Community。前后端源码仍由各自仓库维护；不包含 vLLM、中间件、DeepSeek 或博查服务，也不自动迁移原开发环境数据。本文配置对应 `main` 的多平台审核流程；独立应用部署使用各仓库相匹配的 `local-deploy` 分支，不混用两套流程。

## 阅读入口

- [部署范围与版本](#部署范围与版本)
- [目录与源码](#目录与源码)
- [首次启动](#首次启动)
- [访问地址与网络](#访问地址与网络)
- [数据与初始化](#数据与初始化)
- [中间件与合同审核](#中间件与合同审核)
- [模型与工具配置](#模型与工具配置)
- [顶层用户配置与更新](#顶层用户配置与更新)
- [排查与提交](#排查与提交)
- [Neo4j 部署与本地开发](#neo4j-部署与本地开发)

部署的机器配置以 [docker-compose.yml](docker-compose.yml) 和 [.env.example](.env.example) 为准；业务接口、模型工作流及数据契约由对应版本的后端 README 和 `description/` 维护。

---

## 部署范围与版本

| 服务 | 本仓库负责 | 对外依赖 |
| --- | --- | --- |
| 前端 | 从前端源码构建 Nginx 镜像，提供网页、同源 API 转发和 SSE。 | 后端 API。 |
| 后端 | 从后端源码构建 Python 镜像，自动编译 Lindera，装配单 worker 服务。 | ES、Neo4j、MLLM、Embedding；审核流程使用消息中间件。 |
| Elasticsearch | 构建包含 SmartCN 的单节点镜像，健康检查和数据卷。 | 镜像与插件下载源。 |
| Neo4j | Community 镜像、无认证连接、健康检查和数据卷。 | 镜像下载源。 |

MLLM 与 Embedding 服务需另行部署；消息中间件及其 RabbitMQ 由中间件项目管理。DeepSeek 和博查为独立外部 API，未填写对应 Key 时相关问答工具不能调用。Compose 不代替这些服务的启动或配置。

| 分支 | 使用方式 |
| --- | --- |
| `main` | 多平台审核版。入库先保存本地审核快照，经中间件和外部审核后才正式存储。 |
| `local-deploy` | 独立应用部署存档。使用该分支自己的配置、接口和 README，不按 main 的审核流程部署。 |

前端、后端和 deploy 选择同一套兼容版本，并记录各自提交号；本仓库不会自动拉取、切换或更新源码分支。

---

## 目录与源码

```text
contract-service-deploy/
  docker-compose.yml
  .env.example                 可提交的完整环境模板
  .env                         本机配置与模型密钥，不提交
  user.yaml.example            用户结构示例
  user.yaml                    实际用户配置，不提交
  elasticsearch/Dockerfile     带 SmartCN 的 ES 镜像
  neo4j/Dockerfile             Neo4j Community 镜像
  contract-service-be/         可选：独立克隆的后端仓库
  contract-service-fe/         可选：独立克隆的前端仓库
```

示例配置按源码位于本目录下组织；将两个仓库克隆到对应目录即可。它们已被部署仓库忽略，不会重复提交。各仓库的版本需要自行记录和固定。

若使用同级现有前后端，在 `.env` 中配置：

```dotenv
BACKEND_SOURCE=../contract-service-be
FRONTEND_SOURCE=../contract-service-fe
```

`SOURCE` 相对于 Compose 文件。前后端使用各自项目根目录的 `Dockerfile` 和 `.dockerignore`，deploy 不再重复维护。前端 Nginx 模板也由前端项目维护，通过 `BACKEND_UPSTREAM=backend:20000` 接入后端。ES 和 Neo4j 的镜像配置分别保留在 `elasticsearch/`、`neo4j/`。构建使用当前工作区内容，包括未提交修改；源码更新后须重新构建镜像。

前端使用 `package-lock.json` 与 `npm ci`；后端沿用源码的 `requirements.txt` 版本范围。当前基础镜像使用版本标签，尚未锁定镜像 digest 和全部 Python 间接依赖；正式发布时应固定并记录验证过的镜像版本，避免把重新构建视为完全相同的产物。

---

## 首次启动

要求 Docker Engine / Docker Desktop、支持长格式 bind 挂载的现代 Docker Compose，以及访问镜像仓库、PyPI、npm 和 Elastic 插件下载站的网络。ES、前后端构建和模型调用都需要足够内存。Linux 主机需按所用 ES 版本满足 `vm.max_map_count` 等启动检查；失败时查看 ES 日志，不关闭检查。

新机器第一次配置（已有文件时不要覆盖）：

```bash
test -f .env || cp .env.example .env
test -f user.yaml || cp user.yaml.example user.yaml
chmod 600 .env user.yaml
```

编辑源码路径、模型地址、中间件地址及凭证，并替换用户示例密钥。外部专家和网页搜索分别使用 DeepSeek、博查 Key，未填写时相应工具不能调用。仅运行 Neo4j 时，只需配置 Neo4j 环境变量；完整部署前须另行补齐模型、源码和用户配置。

首次完整部署至少核对以下项目：

| 配置 | 需要确认的内容 |
| --- | --- |
| `BACKEND_SOURCE` / `FRONTEND_SOURCE` | 路径对应真实仓库，根目录包含 Dockerfile；分支及接口契约匹配。 |
| `user.yaml` | 至少一个用户，具有唯一的 name 和 secret_key；替换示例密钥。 |
| `VLLM_MLLM_*` | 模型名称与服务一致，具备多模态、函数调用及可用的 `/tokenize` 接口；模板、推理和预算与部署模型对应。 |
| `VLLM_EMBEDDING_*` | 服务可达，向量维度与 `ELASTICSEARCH_VECTOR_DIMENSIONS` 一致。 |
| `MIDDLEWARE_*` | main 的审核流程需填写地址、platform_code、secret，并为平台配置对应消息权限。 |
| 端口与数据卷 | 监听地址可达、端口没有冲突、Compose 项目名与预期的数据环境一致。 |

配置模板里的模型地址与名称是起点，应填写实际部署值；不要直接复制后端的本机 `.env` 覆盖 deploy 环境。

```bash
docker compose config --quiet
docker compose up -d --build
docker compose ps
curl --fail http://127.0.0.1:17080/contract/api/health
```

上述地址使用默认端口；自定义监听地址或端口后按实际配置访问。首次镜像构建会下载 Python、npm、SmartCN、Rust/Cargo 和词典依赖，构建完成不代表模型及审核链路已验证。应通过一次实际 PDF 提取、一次问答，以及 main 的申请送审和反馈入库检查完整链路。

后端镜像在构建时自动编译 Lindera 中文分词扩展，并验证中文 BM25 检索；无需在宿主机安装 Rust 或运行本地 scripts。原生扩展保存在 `/opt/contract-service/extensions/`，由 Compose 固定加载路径，不受 `/workspace/data` 卷覆盖。

---

## 访问地址与网络

以下均为默认映射；后端源码直接运行的端口是 20000，Compose 的宿主机后端端口是 17000。

| 入口 | 宿主机地址 | 容器内地址 |
| --- | --- | --- |
| 前端网页 | `http://127.0.0.1:17080/contract/` | `frontend:80` |
| 同源业务 API | `http://127.0.0.1:17080/contract/api/` | Nginx 转发至 `backend:20000`。 |
| 后端 API | `http://127.0.0.1:17000/contract/api/` | `backend:20000` |
| Swagger / OpenAPI | `http://127.0.0.1:17000/docs` / `/openapi.json` | 后端提供，不位于 `/contract/api` 下。 |
| Neo4j Browser / Bolt | `http://127.0.0.1:7474` / `bolt://127.0.0.1:7687` | `neo4j:7474` / `neo4j:7687` |
| Elasticsearch | 不发布宿主机端口。 | `http://elasticsearch:9200` |

前端构建固定使用 `/contract/`；同源 API 使用 `/contract/api/`，支持 History 路由刷新、PDF.js 静态资源及不缓冲的 SSE。上传上限默认为 100 MB，可在 `.env` 设置 `CLIENT_MAX_BODY_SIZE`，重新创建前端容器后生效。

除健康检查和登录外，业务 API 使用用户登录取得的 `login_code`，以 `Authorization: Bearer <login_code>` 传入；中间件平台 access_token 由后端内部维护，不能替代本平台的用户免登码。

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
| `neo4j-data` | `/data` | 图数据库、认证数据及事务日志 |
| `neo4j-logs` | `/logs` | Neo4j 运行日志 |
| `./user.yaml`（只读） | `/workspace/data/user/users.yaml` | 顶层审核用户配置 |

命名卷由 Docker 管理，实际名称以 `COMPOSE_PROJECT_NAME` 为前缀，并非 deploy 下的普通目录。后端整体持久化 `/workspace/data`，包括现有和未来新增的数据子目录。`user.yaml` 仍作为子路径只读挂载，覆盖容器中的用户配置视图；真实用户配置应单独备份，不会被写入数据卷。

后端 Compose 固定以下数据位置，避免沿用宿主机 `.env` 中的绝对路径导致写入未持久化的容器目录：

| 内容 | 数据卷内位置 |
| --- | --- |
| 正式合同元数据、摘要、注意事项及检索索引 | `/workspace/data/abstract/contracts.db` |
| 正式合同 PDF | `/workspace/data/contract/` |
| 会话数据库及附件 | `/workspace/data/communication/` |
| 入库审核 SQLite、快照及临时 PDF | `/workspace/data/ingestion-review/` |
| 删除审核 SQLite、备注及处理记录 | `/workspace/data/deletion-review/` |

后端 Dockerfile 已创建审核目录，应用启动时初始化 SQLite。删除审核记录引用正式 PDF，审核批准前不删除原合同。审核数据与正式数据分别管理，均随 `backend-data` 保留。

首次挂载空的命名卷时，Docker 默认将镜像内该目录的内容复制到卷中，因此初始 `data/definition` 来自镜像。已有非空卷不会随镜像重建自动刷新定义；更新类别、字段或检索指南时，须先备份、停写，再显式更新卷内对应定义并重建后端容器。不要为更新定义清空整个卷，也不要覆盖合同和数据库。参见 [Docker 卷初始化规则](https://docs.docker.com/engine/storage/volumes/#mounting-a-volume-over-existing-data)。镜像仍不包含真实用户、`.env`、历史合同、SQLite 或 Git 内容；整体挂载不代表自动复制开发环境的全部 data。

旧方案的 `backend-contracts`、`backend-abstract` 卷不会自动合并到 `backend-data`。若曾使用旧方案入库，应先停写并备份，再迁移 PDF 和完整 SQLite 数据至新卷对应子目录；迁移需单独执行，修改配置不会自动合并或删除旧卷。

更早版本的数据卷若包含 `data/pending-review` 或 `data/pending-deletion`，升级前停写并备份，将旧目录分别迁移为 `data/ingestion-review` 和 `data/deletion-review`。应用只在新目录内兼容迁移旧表名，不自动搜索旧目录；新旧目录同时有记录时须核对并迁移，不能直接覆盖。

ES 镜像构建时安装同版本 SmartCN 插件，不需要 MySQL 式初始化 SQL。后端等待 ES 和 Neo4j 健康后启动，自动创建或检查索引 mapping、初始化 SQLite 并同步类别目录；不会自动生成示例合同。前端等待后端健康后启动。

这是一套全新数据环境，不自动复用原 `contract-service-elasticsearch-data` 卷，也不复制开发 SQLite/PDF。已有合同需要成套迁移 ES、SQLite 与 PDF，并在停写及备份后执行；只复制其中一项会导致数据不一致。

```bash
docker compose stop
docker compose start
docker compose down
```

以上操作保留命名卷。**不要执行 `docker compose down --volumes`，除非确认要永久清空本部署数据。** 更改 Compose 项目名会切换卷，不代表原数据被删除。备份必须覆盖整个 `backend-data`、ES、Neo4j 和独立的 `user.yaml`；SQLite 与 PDF 应在停写状态下一致备份，不要在运行时单独拷贝数据库主文件。清空本平台的数据卷不会同步清除中间件已经发布的申请与反馈消息。

---

## 中间件与合同审核

在 deploy 的私有 `.env` 中填写三项；编码和 secret 必须与中间件登记一致：

```dotenv
MIDDLEWARE_BASE_URL=http://host.docker.internal:22000
MIDDLEWARE_PLATFORM_CODE=contract-pheno
MIDDLEWARE_SECRET=replace-with-platform-secret
```

这是宿主机中间件的地址示例；跨机器部署时换成容器可达的实际 HTTP/HTTPS 地址。`127.0.0.1` 在容器内指向后端自身；`host.docker.internal` 不会让仅监听宿主机回环地址的服务自动变为可达，需调整中间件监听与网络规则。中间件独立部署，本 Compose 不启动 RabbitMQ 或中间件，也不把中间件健康作为后端启动依赖。

后端后台登录取得令牌，按返回间隔心跳；失败按 `MIDDLEWARE_RETRY_INTERVAL_SECONDS` 重试，不阻塞应用启动。三项缺失时不登录，入库和删除申请仍可保存到本地审核区，但不能完成外部送审；后端 `/health` 成功不代表中间件已连接。同一 platform_code 只能保持一个活跃会话，部署后端单 worker，避免开发进程与容器同时使用同一平台凭证。

| 环节 | 配置项 | 默认值及行为 |
| --- | --- | --- |
| 登录与心跳 | `MIDDLEWARE_RETRY_INTERVAL_SECONDS` / `MIDDLEWARE_REQUEST_TIMEOUT_SECONDS` | 失败间隔30秒，单次请求总超时10秒。正常心跳采用服务端建议。 |
| 入库送审 | `INGESTION_REVIEW_SCAN_INTERVAL_SECONDS` / `INGESTION_REVIEW_PUBLISH_RETRY_SECONDS` / `INGESTION_REVIEW_PUBLISH_TIMEOUT_SECONDS` / `INGESTION_REVIEW_PUBLISH_BATCH_SIZE` | 扫描30秒，重试等待60秒，上传超时120秒，每轮最多10条。 |
| 入库审核结果 | `INGESTION_REVIEW_RESULT_POLL_INTERVAL_SECONDS` / `INGESTION_REVIEW_RESULT_BATCH_SIZE` | 轮询30秒，每轮最多10条；批准后正式入库，处理成功再 ack。 |
| 删除送审 | `DELETION_REVIEW_PUBLISH_SCAN_INTERVAL_SECONDS` / `DELETION_REVIEW_PUBLISH_RETRY_SECONDS` / `DELETION_REVIEW_PUBLISH_TIMEOUT_SECONDS` / `DELETION_REVIEW_PUBLISH_BATCH_SIZE` | 扫描30秒，重试等待60秒，上传超时120秒，每轮最多10条。 |
| 批准后正式删除 | `DELETION_REVIEW_SCAN_INTERVAL_SECONDS` / `DELETION_REVIEW_RETRY_SECONDS` / `DELETION_REVIEW_BATCH_SIZE` | 扫描30秒，失败等待60秒，每轮最多10条；仅处理已批准申请。 |
| 审核区清理 | `INGESTION_REVIEW_*` / `DELETION_REVIEW_*` 的 `RETENTION_SECONDS`、`CLEANUP_INTERVAL_SECONDS`、`CLEANUP_BATCH_SIZE` | 本地处理完成后保留604800秒（7天），每3600秒清理，批次上限100。未处理完成申请不按提交时间清除。 |

前端确认入库先保存待审快照；审核通过后才写正式 SQLite、PDF、ES 和 Neo4j。删除申请先锁定 `can_delete` 并保存备注，再后台送审；审批通过才执行正式删除，拒绝则恢复可申请状态。删除审核反馈拉取和 ack 尚未接入，本配置不宣称删除审核全链路已经自动闭环。具体接口由对应后端版本的 API 文档定义。

入库确认和删除申请都要求前端提交申请人 `note`。目前入库审核记录允许所有已登录用户查询，删除审核记录按申请人或原合同上传人过滤；前端应与所部署后端的请求体和查询契约同步。

发送以原 `document_id` 作为 source_id；中间件在各申请流内按平台编码和 source_id 去重。明确暂时失败及发布结果不确定均延迟重试，合法409重复响应关联原 message_id；403、422、本地文件损坏或缺少通行证等进入 blocked，不自动重试。更新配置不会自动解除已有 blocked 记录。审核区 SQLite、正式 PDF 和远端消息不是跨系统原子事务，部署升级及备份须保持这些数据的一致性。

修改 `.env` 后使用 `docker compose up -d --force-recreate backend frontend` 应用环境变量；只执行 restart 不会更新配置。入库反馈接口使用 `/middleware-service/api/ingestion-results/`；中间件需支持当前带 source_id 的 multipart 送审和去重契约。

---

## 模型与工具配置

`.env.example` 已包含主模型、合同提取和会话门禁独立的推理强度、tool-tag 注入开关、思考 FIFO、页面保留轮数、中途反馈提醒，以及文件、通行证、网页搜索和网页内容池配置。配置值需与所部署模型能力及上下文窗口对应；模型、工具协议文件名和 API 地址不从本地开发 `.env` 自动复制。

`VLLM_MLLM_TOOL_TAG_ENABLED=false` 时不在消息上下文注入工具格式及相关纠错说明，模型部署侧仍须配置合适的工具解析和模板。Lindera 扩展路径由 Compose 固定为 `/opt/contract-service/extensions/liblindera_sqlite.so`，不能使用 macOS 原生库。私有执行链路调试默认 `COMMUNICATION_TRACE_ENABLED=false`，仅开发环境使用。

---

## 顶层用户配置与更新

`user.yaml` 的结构与后端原 `data/user/users.yaml` 一致，包含 `users` 列表，每项仅包含 `name` 和 `secret_key`。所有已登录用户均可执行共享合同的查看、新增、删除等操作；个人会话和运行任务仍按所有者隔离。

挂载覆盖容器内配置视图，不会修改开发源码里的用户文件。缺少 `user.yaml` 时 Compose 报错，不创建同名目录。用户在启动期加载，不支持热更新；为兼容编辑器原子替换文件，修改后重新创建后端容器：

```bash
docker compose up -d --force-recreate backend
```

后端单 worker、不启用热重载。重建或重启会清空登录免登码、未入库任务及其内存 PDF；先完成或取消任务，再更新。已入库数据保存在卷中。

修改 `.env` 后需重新创建受影响的容器；`docker compose restart` 不重新加载环境变量。更新前后端源码或前端 Nginx 模板后执行 `docker compose up -d --build`。前端在启动时解析后端地址，单独重建后端导致 IP 改变时，应同时重新创建前端容器。

---

## 排查与提交

```bash
docker compose logs --tail=100 elasticsearch neo4j backend frontend
docker compose exec frontend nginx -t
docker compose exec elasticsearch curl -fsS 'http://localhost:9200/_cat/plugins?v'
docker compose config --quiet
```

| 现象 | 优先检查 |
| --- | --- |
| 构建找不到源码或用户挂载失败 | SOURCE 路径、源码 Dockerfile、`user.yaml` 是否存在且为文件。 |
| 后端没有启动完成 | ES/Neo4j 健康、用户和业务定义、Lindera 扩展、索引 mapping 及启动对账日志。 |
| health 成功但提取/问答失败 | 模型地址与名称、`/tokenize`、Embedding 维度、生成预算及工具解析。health 只表示 API 存活。 |
| 申请一直待发送 | 中间件地址、平台身份和消息权限；确认已有有效平台会话。 |
| 申请显示发送受阻 | 检查是否403/422，或本地缺少 passport、文件损坏、身份不匹配；blocked 不会自动重试。 |
| 修改 `.env` 后没有生效 | 重新创建受影响容器，而非只 restart；固定的数据库地址和路径仍由 Compose environment 覆盖。 |
| 前端没有收到即时输出 | 外层代理的 SSE 缓冲、读取超时和后端事件日志。 |

构建依赖下载失败属于网络或仓库问题，不能用已有服务健康代替新镜像验证。

提交部署仓库时只提交环境及用户模板、Compose、ES/Neo4j Dockerfile 和本文档；前后端 Dockerfile 分别随各自项目提交。不要提交 `.env`、`user.yaml`、源码副本或备份。不要将 `docker compose config` 完整输出粘贴到工单：它会展开模型密钥。检查语法使用 `--quiet`。


---

## Neo4j 部署与本地开发

Neo4j 与 ES 一样由主 Compose 编排，镜像构建入口为 `neo4j/Dockerfile`。默认版本固定为 `5.26.30-community`，可通过 `NEO4J_VERSION` 调整。主配置发布图数据库端口，默认仅绑定宿主机回环地址；Compose 内服务使用 `bolt://neo4j:7687`。后端已接入共享异步客户端、合同节点入库/删除同步和启动恢复，并等待 Neo4j 健康；关系创建、独立删除及一跳查询接口已实现。

与 ES 一致关闭认证：`NEO4J_AUTH: "none"`，无需配置用户名和密码。宿主机映射由 `NEO4J_BIND_ADDRESS`（默认 `127.0.0.1`）、`NEO4J_HTTP_PORT`（默认 `7474`）和 `NEO4J_BOLT_PORT`（默认 `7687`）控制，在 `.env` 中配置。数据库绑定地址独立于前后端的 `BIND_ADDRESS`，不会随前后端地址改变。

仅启动 Neo4j：

```bash
docker compose config --quiet
docker compose up -d --build --wait neo4j
```

宿主机开发和整套容器部署共用主 Compose，无需额外覆盖文件。按默认配置，浏览器访问 `http://127.0.0.1:7474`，本机后端连接 `bolt://127.0.0.1:7687`，认证方式选择 No authentication。容器后端仍连接 `bolt://neo4j:7687`，不受宿主机映射端口变化影响。

上述命令只选择 `neo4j`，不会启动或重建 ES、前后端。修改监听地址或端口后，重新执行 `docker compose up -d neo4j` 应用配置；无需删除数据卷。

默认堆初始 256 MiB、最大 512 MiB、页面缓存 256 MiB，容器内存上限 2 GiB；这是小型开发环境的起点，非生产容量结论。对应配置为 `NEO4J_HEAP_INITIAL_SIZE`、`NEO4J_HEAP_MAX_SIZE`、`NEO4J_PAGECACHE_SIZE`、`NEO4J_MEMORY_LIMIT`，堆和页面缓存之外需保留原生内存余量。

健康检查通过无认证 Bolt `RETURN 1` 验证数据库，每 30 秒检查一次。排查与停启：

```bash
docker compose logs --tail=100 neo4j
docker compose stop neo4j
docker compose start neo4j
```

命名卷随 Compose 项目名隔离；停止、重建容器保留数据。禁止为普通停启使用 `down --volumes`。图关系投入使用后需将 Neo4j 纳入备份，Community 可停库后使用 `neo4j-admin database dump`；备份与 SQLite/PDF 的一致性策略仍需完善。后端启动会为真实的历史合同补建节点，不创建示例节点和关系。
