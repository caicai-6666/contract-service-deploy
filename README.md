# 现象合同智能管理平台 · 部署

本机 IDE 开发使用 [docker-compose.dev.yml](docker-compose.dev.yml)，只启动合同部署独立的 Elasticsearch/SmartCN 和 Neo4j，向宿主机发布回环端口。环境准备及统一管理命令见[本机开发基础设施](../dev-infra/README.md)。

本仓库统一构建和运行前端 Nginx、后端 FastAPI、单节点 Elasticsearch 与 Neo4j Community。前后端源码仍由各自仓库维护；不包含 vLLM、中间件或 DeepSeek 服务，也不自动迁移原开发环境数据。本文配置对应 `main` 的多平台审核流程；独立应用部署使用各仓库相匹配的 `local-deploy` 分支，不混用两套流程。

## 阅读入口

- [部署范围与版本](#部署范围与版本)
- [目录与源码](#目录与源码)
- [首次启动](#首次启动)
- [访问地址与网络](#访问地址与网络)
- [数据与初始化](#数据与初始化)
- [中间件与合同审核](#中间件与合同审核)
- [模型与工具配置](#模型与工具配置)
- [用户认证与更新](#用户认证与更新)
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

MLLM 与 Embedding 服务需另行部署；消息中间件及其 RabbitMQ 由中间件项目管理。DeepSeek 为独立外部 API，未填写 Key 时专家和网页搜索工具不能调用。Compose 不代替这些服务的启动或配置。

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
chmod 600 .env
```

编辑源码路径、模型地址、中间件地址及凭证。外部专家 Responses 和独立网页搜索共用 DEEPSEEK_API_KEY、DEEPSEEK_BASE_URL，未填写 Key 时相应工具不能调用；搜索使用 DEEPSEEK_SEARCH_COUNT、DEEPSEEK_SEARCH_TIMEOUT_SECONDS、DEEPSEEK_SEARCH_TRUST_ENV，不再需要博查配置。仅运行 Neo4j 时，只需配置 Neo4j 环境变量；完整部署前须另行补齐模型、源码及中间件身份与权限配置。

首次完整部署至少核对以下项目：

| 配置 | 需要确认的内容 |
| --- | --- |
| `BACKEND_SOURCE` / `FRONTEND_SOURCE` | 路径对应真实仓库，根目录包含 Dockerfile；分支及接口契约匹配。 |
| `VLLM_MLLM_*` | 模型名称与服务一致，具备多模态、函数调用及可用的 `/tokenize` 接口；模板、推理和预算与部署模型对应。 |
| `VLLM_EMBEDDING_*` | 服务可达，向量维度与 `ELASTICSEARCH_VECTOR_DIMENSIONS` 一致。 |
| `MIDDLEWARE_*` | main 的审核流程需填写地址、platform_code、secret，并为平台配置对应消息权限。 |
| 端口与数据卷 | 监听地址可达、端口没有冲突、Compose 项目名与预期的数据环境一致。 |

配置模板里的模型地址与名称是起点，应填写实际部署值；不要直接复制后端的本机 `.env` 覆盖 deploy 环境。

```bash
docker compose config --quiet
docker compose up -d --build
docker compose ps
curl --fail http://127.0.0.1:17080/knowledge-accumulation/contract/api/health
```

上述地址使用默认端口；自定义监听地址或端口后按实际配置访问。首次镜像构建会下载 Python、npm、SmartCN、Rust/Cargo 和词典依赖，构建完成不代表模型及审核链路已验证。应通过一次实际 PDF 提取、一次问答，以及 main 的申请送审和反馈入库检查完整链路。

后端镜像在构建时自动编译 Lindera 中文分词扩展，并验证中文 BM25 检索；无需在宿主机安装 Rust 或运行本地 scripts。原生扩展保存在 `/opt/contract-service/extensions/`，由 Compose 固定加载路径，不受 `/workspace/data` 卷覆盖。

---

## 访问地址与网络

以下均为默认映射；后端源码直接运行的端口是 20000，Compose 的宿主机后端端口是 17000。

| 入口 | 宿主机地址 | 容器内地址 |
| --- | --- | --- |
| 前端网页 | `http://127.0.0.1:17080/knowledge-accumulation/contract/` | `frontend:80` |
| 同源业务 API | `http://127.0.0.1:17080/knowledge-accumulation/contract/api/` | Nginx 转发至 `backend:20000`。 |
| 后端 API | `http://127.0.0.1:17000/knowledge-accumulation/contract/api/` | `backend:20000` |
| Swagger / OpenAPI | `http://127.0.0.1:17000/knowledge-accumulation/docs` / `/knowledge-accumulation/openapi.json` | 后端提供，统一置于 `/knowledge-accumulation` 下。 |
| Neo4j Browser / Bolt | `http://127.0.0.1:7474` / `bolt://127.0.0.1:7687` | `neo4j:7474` / `neo4j:7687` |
| Elasticsearch | 不发布宿主机端口。 | `http://elasticsearch:9200` |

前端构建固定使用 `/knowledge-accumulation/contract/`；同源 API 使用 `/knowledge-accumulation/contract/api/`，支持 History 路由刷新、PDF.js 静态资源及不缓冲的 SSE。上传上限默认为 100 MB，可在 `.env` 设置 `CLIENT_MAX_BODY_SIZE`；图片和图标默认缓存 30 天，可通过非负整数 `ICON_CACHE_MAX_AGE=2592000` 调整缓存秒数。重新创建前端容器后生效。

统一入口与本平台需使用相同协议、域名和端口，以共享 `localStorage.access_token`；外层网关同时转发 `/knowledge-accumulation/` 入口与本平台子路径。后端通过 ASGI 部署时不设置开发入口专用的 `APP_PATH_PREFIX=/dev`。

后端应用路径已统一增加 `/knowledge-accumulation`。前端的 API 基础地址与 Nginx location 必须同步使用上述新路径，并原样转发至后端；仍只代理 `/contract/api/` 的旧前端镜像需要同步更新后重新构建。Compose 的后端健康检查已使用新地址。

除健康检查外，业务 API 使用统一登录平台签发的用户 access_token，以 `Authorization: Bearer <用户 access_token>` 传入。后端以独立平台令牌查询用户登录及当前平台管理员列表，两项通过才放行。旧本地登录接口已移除，当前用户查询为 GET /knowledge-accumulation/contract/api/auth/me。

后端同时消费中间件文件等级调整，先在正式 SQLite 中提交当前等级及历史，再确认消息；文件不存在时记录跳过并确认。`.env` 可通过 `FILE_PERMISSION_ADJUSTMENT_POLL_INTERVAL_SECONDS=30` 和 `FILE_PERMISSION_ADJUSTMENT_BATCH_SIZE=10` 调整轮询，重建后端容器生效。历史与正式元数据共用 backend-data 卷，不随合同删除或审核临时区清理而清除。

前后端共用宿主机监听地址 `BIND_ADDRESS`，默认 `127.0.0.1`。`FRONTEND_PORT=17080` 映射到前端容器 80，`BACKEND_PORT=17000` 映射到后端容器 20000，供映射服务器直接转发。代理位于其他机器时，将公共监听地址改为指定内网 IP 或 `0.0.0.0`，并限制访问来源；同机代理可保持回环地址。两个宿主机端口必须不同且未被占用。

映射服务器的上游设置为 `http://后端宿主机内网IP:BACKEND_PORT`，保留 `/knowledge-accumulation/contract/api/` 路径、Authorization 请求头和 SSE 流式响应，关闭响应缓冲并配置足够的读取超时。前端容器仍通过 `backend:20000` 访问后端，不受宿主机端口修改影响。变更端口后需重新创建后端容器，不能只执行 restart。

宿主机防火墙或安全组应仅允许映射服务器访问后端端口。公网入口需增加 HTTPS、访问控制及上传容量限制，不应直接暴露后端明文 HTTP。ES 不发布宿主机端口，当前关闭认证，仅适用于受信任 Docker 主机与专用网络；本配置不是完整生产安全方案。

模型服务在 Compose 外运行：容器内 `127.0.0.1` 指向容器自身。宿主机模型可使用 `host.docker.internal`，远程模型直接配置可达地址。Linux 已设置 host-gateway 映射，但模型仍需监听容器可达网卡并允许防火墙访问。模型健康不包含在后端 health 检查中，应额外通过真实合同任务验证。

---

## 数据与初始化

| 卷 / 挂载 | 容器位置 | 用途 |
| --- | --- | --- |
| `backend-data` | `/workspace/data` | 整个运行 data：PDF、SQLite（含 WAL/SHM）及其他数据子目录 |
| `elasticsearch-data` | `/usr/share/elasticsearch/data` | ES 索引数据 |
| `neo4j-data` | `/data` | 图数据库、认证数据及事务日志 |
| `neo4j-logs` | `/logs` | Neo4j 运行日志 |

命名卷由 Docker 管理，实际名称以 `COMPOSE_PROJECT_NAME` 为前缀，并非 deploy 下的普通目录。后端整体持久化 `/workspace/data`，包括现有和未来新增的数据子目录。用户配置由统一登录平台与中间件维护，不再挂载 user.yaml；已有数据卷中的旧文件不参与认证。

后端 Compose 固定以下数据位置，避免沿用宿主机 `.env` 中的绝对路径导致写入未持久化的容器目录：

| 内容 | 数据卷内位置 |
| --- | --- |
| 正式合同元数据、摘要、注意事项及检索索引 | `/workspace/data/abstract/contracts.db` |
| 正式合同 PDF | `/workspace/data/contract/` |
| 会话数据库及附件 | `/workspace/data/communication/` |
| 入库审核 SQLite、快照及临时 PDF | `/workspace/data/ingestion-review/` |
| 删除审核 SQLite、备注及处理记录 | `/workspace/data/deletion-review/` |

后端 Dockerfile 已创建审核目录，应用启动时初始化 SQLite。删除审核记录引用正式 PDF，审核批准前不删除原合同。审核数据与正式数据分别管理，均随 `backend-data` 保留。

合同类别、Core 字段和检索指南随镜像位于 `/workspace/assets/definition/`，工具格式位于 `/workspace/assets/tool-tag/`，均不进入 `backend-data`。Compose 固定三个定义目录为镜像内路径；更新静态资源后重新构建并创建后端容器即可生效，无需修改数据卷。旧卷中的 `data/definition` 不再读取；若曾直接修改卷内定义，升级前须将需要保留的内容核对并同步至后端源码的 `assets/definition/` 后构建。不要为更新定义清空数据卷。镜像不包含真实用户、`.env`、历史合同、SQLite 或 Git 内容；整体挂载不代表自动复制开发环境的全部 data。

旧方案的 `backend-contracts`、`backend-abstract` 卷不会自动合并到 `backend-data`。若曾使用旧方案入库，应先停写并备份，再迁移 PDF 和完整 SQLite 数据至新卷对应子目录；迁移需单独执行，修改配置不会自动合并或删除旧卷。

更早版本的数据卷若包含 `data/pending-review` 或 `data/pending-deletion`，升级前停写并备份，将旧目录分别迁移为 `data/ingestion-review` 和 `data/deletion-review`。应用只在新目录内兼容迁移旧表名，不自动搜索旧目录；新旧目录同时有记录时须核对并迁移，不能直接覆盖。

ES 镜像构建时安装同版本 SmartCN 插件，不需要 MySQL 式初始化 SQL。后端等待 ES 和 Neo4j 健康后启动，自动创建或检查索引 mapping、初始化 SQLite 并同步类别目录；不会自动生成示例合同。前端等待后端健康后启动。

这是一套全新数据环境，不自动复用原 `contract-service-elasticsearch-data` 卷，也不复制开发 SQLite/PDF。已有合同需要成套迁移 ES、SQLite 与 PDF，并在停写及备份后执行；只复制其中一项会导致数据不一致。

```bash
docker compose stop
docker compose start
docker compose down
```

以上操作保留命名卷。**不要执行 `docker compose down --volumes`，除非确认要永久清空本部署数据。** 更改 Compose 项目名会切换卷，不代表原数据被删除。备份必须覆盖整个 `backend-data`、ES 和 Neo4j；SQLite 与 PDF 应在停写状态下一致备份，不要在运行时单独拷贝数据库主文件。清空本平台的数据卷不会同步清除中间件已经发布的申请与反馈消息。

---

## 中间件与合同审核

在 deploy 的私有 `.env` 中填写三项；编码和 secret 必须与中间件登记一致：

```dotenv
MIDDLEWARE_BASE_URL=http://host.docker.internal:22000
MIDDLEWARE_PLATFORM_CODE=contract-pheno
MIDDLEWARE_SECRET=replace-with-platform-secret
```

这是宿主机中间件的地址示例；跨机器部署时换成容器可达的实际 HTTP/HTTPS 地址。`127.0.0.1` 在容器内指向后端自身；`host.docker.internal` 不会让仅监听宿主机回环地址的服务自动变为可达，需调整中间件监听与网络规则。中间件独立部署，本 Compose 不启动 RabbitMQ 或中间件，也不把中间件健康作为后端启动依赖。

后端后台登录取得令牌，平台 TTL 与心跳间隔均采用每次中间件响应，无需在 deploy 配置同样数值。取得服务端周期后的普通失败按该周期与本地兜底的较小值重试；平台 401 撤销当前令牌并及时唤醒重登。只有尚未取得周期或登录凭证被拒绝时，才按 `MIDDLEWARE_RETRY_INTERVAL_SECONDS` 兜底。连接故障不阻塞应用启动，但三项缺失或平台连接未就绪时受保护接口返回503；已有本地审核申请保留。后端 health 成功不代表中间件已连接。同一 platform_code 只能保持一个活跃会话，部署后端单 worker，避免开发进程与容器同时使用同一平台凭证。

| 环节 | 配置项 | 默认值及行为 |
| --- | --- | --- |
| 登录与心跳 | `MIDDLEWARE_RETRY_INTERVAL_SECONDS` / `MIDDLEWARE_REQUEST_TIMEOUT_SECONDS` | 未获服务端参数或凭证被拒绝时兜底30秒，单次请求总超时10秒；TTL、心跳及已知周期后的普通重试采用服务端返回值。 |
| 入库送审 | `INGESTION_REVIEW_SCAN_INTERVAL_SECONDS` / `INGESTION_REVIEW_PUBLISH_RETRY_SECONDS` / `INGESTION_REVIEW_PUBLISH_TIMEOUT_SECONDS` / `INGESTION_REVIEW_PUBLISH_BATCH_SIZE` | 扫描30秒，重试等待60秒，上传超时120秒，每轮最多10条。 |
| 入库审核结果 | `INGESTION_REVIEW_RESULT_POLL_INTERVAL_SECONDS` / `INGESTION_REVIEW_RESULT_BATCH_SIZE` | 轮询30秒，每轮最多10条；批准后正式入库，处理成功再 ack。 |
| 删除送审 | `DELETION_REVIEW_PUBLISH_SCAN_INTERVAL_SECONDS` / `DELETION_REVIEW_PUBLISH_RETRY_SECONDS` / `DELETION_REVIEW_PUBLISH_TIMEOUT_SECONDS` / `DELETION_REVIEW_PUBLISH_BATCH_SIZE` | 扫描30秒，重试等待60秒，请求超时120秒，每轮最多10条，不上传文件。 |
| 删除审核反馈 | `DELETION_REVIEW_RESULT_POLL_INTERVAL_SECONDS` / `DELETION_REVIEW_RESULT_BATCH_SIZE` | 每轮等待30秒，最多处理10条；实际删除或拒绝恢复完成后 ack。 |
| 批准后正式删除 | `DELETION_REVIEW_SCAN_INTERVAL_SECONDS` / `DELETION_REVIEW_RETRY_SECONDS` / `DELETION_REVIEW_BATCH_SIZE` | 扫描30秒，失败等待60秒，每轮最多10条；仅处理已批准申请。 |
| 审核区清理 | `INGESTION_REVIEW_*` / `DELETION_REVIEW_*` 的 `RETENTION_SECONDS`、`CLEANUP_INTERVAL_SECONDS`、`CLEANUP_BATCH_SIZE` | 本地处理完成后保留604800秒（7天），每3600秒清理，批次上限100。未处理完成申请不按提交时间清除。 |

前端确认入库先保存待审快照；审核通过后才写正式 SQLite、PDF、ES 和 Neo4j。删除申请先锁定 `can_delete` 并保存备注，再以 URL 编码表单提交七个业务字段，不提交 file 或 file_extension；审批通过才执行正式删除，拒绝则恢复可申请状态。删除审核反馈通过 `/middleware-service/api/deletion-results/pull` 拉取，完成正式删除或拒绝恢复后再 ack，失败和重复投递按本地状态重试。具体接口由对应后端版本的 API 文档定义。

审核历史保留期与再次申请资格分离：正式删除成功后，同一 PDF 重新入库可立即再次申请删除，生成新的申请 ID，无需等待旧审核记录清理；待审、删除中或失败的申请仍阻止重复提交。升级后后端初始化会自动更新删除审核库的部分唯一索引，保留旧审核记录与备注；旧成功反馈重放不会删除新入库实例。

中间件的 passport 为字符串数组；一份合同可关联多个通行证。后端将完整数组存入正式库及审核区，删除送审通过重复 passport 表单字段发送；旧单值存储在重启初始化时迁移，升级前需停写并成套备份三个 SQLite。入库详情 review.passport 和删除详情 passport 均返回数组，未绑定仍为 null。

入库确认和删除申请都要求前端提交申请人 `note`。目前入库审核记录允许所有已登录用户查询，删除审核记录按申请人或原合同上传人过滤；前端应与所部署后端的请求体和查询契约同步。

发送以原 `document_id` 作为 source_id；中间件在各申请流内按平台编码和 source_id 去重。明确暂时失败及发布结果不确定均延迟重试，合法409重复响应关联原 message_id；403、422、本地文件损坏或缺少通行证等进入 blocked，不自动重试。更新配置不会自动解除已有 blocked 记录。审核区 SQLite、正式 PDF 和远端消息不是跨系统原子事务，部署升级及备份须保持这些数据的一致性。

更新源码或拉取 `.env.example` 不会自动合并已有私有 `.env`；部署升级时应补齐新增键并保留原地址、密钥和自定义值。审核轮询、重试和清理配置通过 Compose 的 `backend.env_file: .env` 注入，无需逐项复制到 `environment`；两个审核目录固定在 backend-data 持久卷内。

修改 `.env` 后使用 `docker compose up -d --force-recreate backend frontend` 应用环境变量；只执行 restart 不会更新配置。入库反馈 pull/ack 使用 `/middleware-service/api/ingestion-results/`，删除反馈 pull/ack 使用 `/middleware-service/api/deletion-results/`；中间件需支持当前带 source_id 的送审和去重契约：入库仍为含 PDF 的 multipart，删除为无附件的 URL 编码表单。

---

## 模型与工具配置

`.env.example` 已包含主模型、合同提取和会话门禁独立的推理强度、tool-tag 注入开关、页面保留轮数、中途反馈提醒，以及文件、通行证、网页搜索、网页内容和图片配置。网页图片默认最多 6 个候选、保留 3 张，单图下载及标准化上限为 4194304 字节，分别由 `COMMUNICATION_WEB_IMAGE_MAX_CANDIDATES`、`COMMUNICATION_WEB_IMAGE_MAX_SELECTED` 和 `COMMUNICATION_WEB_IMAGE_MAX_BYTES` 控制。配置值需与所部署模型能力及上下文窗口对应；模型、工具协议文件名和 API 地址不从本地开发 `.env` 自动复制。

独立思考 FIFO 及 `VLLM_MLLM_REASONING_WINDOW_ROUNDS` 已移除，旧 `.env` 中的同名键应删除。已接受消息的完整原生思考随任务备份和恢复，旧库在初始化时事务迁移可定位的旧思考条目；升级前备份会话 SQLite，迁移失败会回滚，已被旧 FIFO 驱逐的内容无法恢复。思考仍计入完整上下文预算，不向用户 SSE 或公开历史披露。

`VLLM_MLLM_TOOL_TAG_ENABLED=false` 时不在消息上下文注入工具格式及相关纠错说明，模型部署侧仍须配置合适的工具解析和模板。Lindera 扩展路径由 Compose 固定为 `/opt/contract-service/extensions/liblindera_sqlite.so`，不能使用 macOS 原生库。私有执行链路调试默认 `COMMUNICATION_TRACE_ENABLED=false`，仅开发环境使用。

Embedding 文本与图像分别使用后端源码中的统一指令，由 `app/core/embedding_prompt.py` 维护，不新增环境变量。全新部署直接使用当前指令；已有数据升级时需成套备份并重算受影响的文档向量，保证查询与存储侧编码一致，包括 SQLite 中的合同摘要、注意事项、会话检索投影，ES 中的问题与页面融合向量，以及 Neo4j 中的关系描述向量。应用启动和镜像重建不会自动完成重算；本机开发环境使用的新索引名称也不会自动同步到其他部署。具体版本与迁移边界见对应后端的 `description/capability/infrastructure/embedding-instruction-selection.md`，不要仅切换查询侧指令后继续使用旧向量。

### A2A 子平台

默认 `A2A_ENABLED=false`，启用前需在中间件登记本平台为支持 A2A 的子管理平台，并配置现有 `MIDDLEWARE_BASE_URL`、`MIDDLEWARE_PLATFORM_CODE` 和 `MIDDLEWARE_SECRET`。后端主动领取和回复任务，不新增公开端口，不直接连接 RabbitMQ；中间件须支持当前 `platform-a2a/requests/pull`、`platform-a2a/replies/publish` 及 `user/file-permission` 契约。

| 配置 | 默认值 | 含义 |
| --- | --- | --- |
| `A2A_CONTEXT_TTL_SECONDS` | `1800` | 内存响应检查点存活秒数。 |
| `A2A_CONTEXT_CAPACITY` | `100` | 内存响应检查点数量上限。 |
| `A2A_RECEIPT_CAPACITY` | `1000` | 任务幂等结果及私有审计数量上限。 |
| `A2A_POLL_INTERVAL_SECONDS` | `2` | 后台轮询及续租周期秒数，须小于中间件任务租约。 |

A2A 复用单 worker 的模型额度和正式合同资源，按用户本平台文件等级过滤读取范围。响应链、回执和审计只驻留内存，不接入 Communication 归档；后端重启会清空它们，调用方需重新建立响应链。配置通过 `backend.env_file` 注入，修改后重新创建后端容器；协议和权限边界见后端 `description/architecture/workflow/contract-communication/a2a-runtime.md`。

---

## 用户认证与更新

主分支不再准备或挂载 user.yaml；REVIEWER_USER_FILE 和 AUTH_LOGIN_CODE_TTL_SECONDS 已移除。用户须在统一登录平台登录，由中间件保存有效登录，并加入当前平台管理员集合。无效用户返回401，非本平台管理员返回403，中间件或平台会话未就绪返回503，不回退到旧用户文件。

前端应移除本地密钥登录流程，改用统一入口传入的用户 token；资源请求与 SSE 同样携带 Bearer。用户登录查询会续期，SSE 只在连接建立时认证，心跳消息不会给用户续期。

会话和任务按稳定 userid 隔离，换 token 或改用户名不改变新数据归属。旧密钥会话保留但未自动认领，需可信映射才能恢复；旧删除审核未知 userid 时也不按同名用户开放。升级前应备份 SQLite，详情以当前后端的用户身份迁移文档为准。

后端仍为单 worker，不启用热重载。更新源码后重新构建后端，更新环境后重新创建容器；重启不主动清空中间件登录，但会释放尚未持久化的内存运行。

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
| 构建找不到源码 | SOURCE 路径、源码 Dockerfile 是否存在。 |
| 后端没有启动完成 | ES/Neo4j 健康、中间件用户认证与业务定义、Lindera 扩展、索引 mapping 及启动对账日志。 |
| health 成功但提取/问答失败 | 模型地址与名称、`/tokenize`、Embedding 维度、生成预算及工具解析。health 只表示 API 存活。 |
| 申请一直待发送 | 中间件地址、平台身份和消息权限；确认已有有效平台会话。 |
| 申请显示发送受阻 | 检查是否403/422，或本地缺少 passport、身份不匹配；入库送审还需检查文件；blocked 不会自动重试。 |
| 修改 `.env` 后没有生效 | 重新创建受影响容器，而非只 restart；固定的数据库地址和路径仍由 Compose environment 覆盖。 |
| 前端没有收到即时输出 | 外层代理的 SSE 缓冲、读取超时和后端事件日志。 |

构建依赖下载失败属于网络或仓库问题，不能用已有服务健康代替新镜像验证。

提交部署仓库时只提交环境模板、Compose、ES/Neo4j Dockerfile 和本文档；前后端 Dockerfile 分别随各自项目提交。不要提交 `.env`、`user.yaml`、源码副本或备份。不要将 `docker compose config` 完整输出粘贴到工单：它会展开模型密钥。检查语法使用 `--quiet`。

后端仓库只分发部署必需源码、配置与模型资源、描述文档和必要目录占位，`scripts/`、`tests/`、`experiment/` 不随克隆提供；Compose 和镜像构建不依赖这些本地目录。Git 白名单只约束源码交付，不改变当前数据卷挂载或运行时自动创建目录的行为。


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
