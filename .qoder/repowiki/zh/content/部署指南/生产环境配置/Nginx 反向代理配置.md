# Nginx 反向代理配置

<cite>
**本文档引用的文件**
- [README.md](file://README.md)
- [docker-compose.yaml](file://docker-compose.yaml)
- [Dockerfile](file://Dockerfile)
- [start.sh](file://start.sh)
- [backend/app/api/main.py](file://backend/app/api/main.py)
- [backend/app/config.py](file://backend/app/config.py)
- [frontend/vite.config.ts](file://frontend/vite.config.ts)
- [frontend/src/services/api.ts](file://frontend/src/services/api.ts)
- [frontend/index.html](file://frontend/index.html)
- [backend/app/api/routes/trip.py](file://backend/app/api/routes/trip.py)
- [backend/app/api/routes/chat.py](file://backend/app/api/routes/chat.py)
- [backend/app/models/schemas.py](file://backend/app/models/schemas.py)
</cite>

## 目录
1. [简介](#简介)
2. [项目结构](#项目结构)
3. [核心组件](#核心组件)
4. [架构概览](#架构概览)
5. [详细组件分析](#详细组件分析)
6. [依赖分析](#依赖分析)
7. [性能考虑](#性能考虑)
8. [故障排除指南](#故障排除指南)
9. [结论](#结论)
10. [附录](#附录)

## 简介

本文档提供了基于 TripStar 项目的 Nginx 反向代理完整配置指南。TripStar 是一个基于 HelloAgents 框架的多智能体协作文旅规划平台，采用前后端分离架构，包含 Vue 3 前端、FastAPI 后端和 LLM/Agents 智能推理层。

该指南涵盖了 Nginx 的安装和基础配置、主配置文件和站点配置文件结构、静态资源服务配置、HTTPS 配置、负载均衡配置、反向代理规则配置以及缓存策略配置等关键内容。

## 项目结构

TripStar 项目采用标准的前后端分离架构，主要组件包括：

```mermaid
graph TB
subgraph "前端层"
FE[Vite 开发服务器<br/>端口 5173]
SPA[Vue 3 单页应用<br/>静态资源]
end
subgraph "后端层"
API[FastAPI 应用<br/>端口 7860]
WS[WebSocket 服务<br/>任务状态推送]
end
subgraph "容器层"
NGINX[Nginx 反向代理<br/>端口 80/443]
DOCKER[Docker 容器编排]
end
subgraph "外部服务"
LLM[大语言模型 API]
MAP[高德地图 API]
XHS[小红书 API]
end
FE --> NGINX
SPA --> NGINX
NGINX --> API
API --> LLM
API --> MAP
API --> XHS
API --> WS
```

**图表来源**
- [docker-compose.yaml:1-24](file://docker-compose.yaml#L1-L24)
- [Dockerfile:1-64](file://Dockerfile#L1-L64)
- [backend/app/api/main.py:1-147](file://backend/app/api/main.py#L1-L147)

**章节来源**
- [README.md:43-97](file://README.md#L43-L97)
- [docker-compose.yaml:1-24](file://docker-compose.yaml#L1-L24)
- [Dockerfile:1-64](file://Dockerfile#L1-L64)

## 核心组件

### 服务器端口配置

根据项目配置，各组件使用的端口如下：

| 组件 | 端口 | 用途 | 配置来源 |
|------|------|------|----------|
| 前端开发服务器 | 5173 | Vue 开发环境 | [vite.config.ts:14-21](file://frontend/vite.config.ts#L14-L21) |
| 后端 API 服务 | 7860 | FastAPI 应用 | [docker-compose.yaml:11-21](file://docker-compose.yaml#L11-L21) |
| Nginx 反向代理 | 80/443 | 外部访问入口 | Nginx 配置 |
| WebSocket | 7860 | 实时任务状态推送 | [trip.py:390-440](file://backend/app/api/routes/trip.py#L390-L440) |

### API 路由结构

后端 API 采用统一的路由前缀 `/api`，包含以下主要路由：

```mermaid
graph TD
API_ROOT[/api] --> TRIP[/trip - 旅行规划]
API_ROOT --> CHAT[/chat - AI 问答]
API_ROOT --> POI[/poi - POI 搜索]
API_ROOT --> MAP[/map - 地图服务]
API_ROOT --> SETTINGS[/settings - 配置管理]
TRIP --> PLAN[/plan - 规划任务]
TRIP --> STATUS[/status/{task_id} - 状态查询]
TRIP --> WS[/ws/{task_id} - WebSocket]
TRIP --> HISTORY[/history - 历史记录]
CHAT --> ASK[/ask - 问答接口]
```

**图表来源**
- [backend/app/api/main.py:55-60](file://backend/app/api/main.py#L55-L60)
- [backend/app/api/routes/trip.py:17](file://backend/app/api/routes/trip.py#L17)
- [backend/app/api/routes/chat.py:7](file://backend/app/api/routes/chat.py#L7)

**章节来源**
- [backend/app/api/main.py:55-60](file://backend/app/api/main.py#L55-L60)
- [backend/app/api/routes/trip.py:17](file://backend/app/api/routes/trip.py#L17)
- [backend/app/api/routes/chat.py:7](file://backend/app/api/routes/chat.py#L7)

## 架构概览

### 整体系统架构

```mermaid
sequenceDiagram
participant Client as 客户端浏览器
participant Nginx as Nginx 反向代理
participant API as FastAPI 后端
participant LLM as LLM 服务
participant Map as 高德地图
participant XHS as 小红书
Client->>Nginx : HTTP 请求
Nginx->>Nginx : 路由匹配和反向代理
Nginx->>API : 转发请求
API->>LLM : AI 生成请求
API->>Map : 地图服务请求
API->>XHS : 小红书数据请求
XHS-->>API : 游记数据
Map-->>API : POI 信息
LLM-->>API : AI 生成结果
API-->>Nginx : 响应数据
Nginx-->>Client : 返回响应
```

**图表来源**
- [backend/app/api/main.py:138-147](file://backend/app/api/main.py#L138-L147)
- [backend/app/api/routes/trip.py:315-388](file://backend/app/api/routes/trip.py#L315-L388)

### 静态资源服务架构

```mermaid
graph LR
subgraph "静态资源层"
HTML[index.html]
CSS[CSS 文件]
JS[JavaScript 文件]
IMG[图片资源]
FONT[字体文件]
end
subgraph "CDN 缓存层"
CDN_CACHE[CDN 缓存]
EDGE_CACHE[边缘缓存]
end
subgraph "Nginx 层"
NGINX_STATIC[Nginx 静态文件服务]
NGINX_COMPRESS[Nginx 压缩服务]
NGINX_CACHE[Nginx 缓存控制]
end
HTML --> NGINX_STATIC
CSS --> NGINX_STATIC
JS --> NGINX_STATIC
IMG --> NGINX_STATIC
FONT --> NGINX_STATIC
NGINX_STATIC --> CDN_CACHE
CDN_CACHE --> EDGE_CACHE
EDGE_CACHE --> Client
```

**图表来源**
- [backend/app/api/main.py:121-136](file://backend/app/api/main.py#L121-L136)

## 详细组件分析

### Nginx 安装和基础配置

#### 基础环境准备

```bash
# Ubuntu/Debian 系统
sudo apt update
sudo apt install nginx

# CentOS/RHEL 系统
sudo yum install epel-release
sudo yum install nginx

# 验证安装
nginx -v
systemctl status nginx
```

#### 主配置文件结构

Nginx 主配置文件通常位于 `/etc/nginx/nginx.conf`，包含以下关键部分：

```mermaid
graph TD
MAIN[nginx.conf] --> HTTP_BLOCK[http 块]
MAIN --> EVENTS_BLOCK[events 块]
MAIN --> INCLUDE_CONF[include /etc/nginx/conf.d/*.conf]
HTTP_BLOCK --> SERVER_BLOCK[server 块]
HTTP_BLOCK --> INCLUDE_HTTP_CONF[include /etc/nginx/http.d/*.conf]
SERVER_BLOCK --> LISTEN[listen 80]
SERVER_BLOCK --> SERVER_NAME[server_name]
SERVER_BLOCK --> LOCATION[location 块]
LOCATION --> STATIC[静态资源]
LOCATION --> PROXY[反向代理]
LOCATION --> WEBSOCKET[WebSocket]
```

**图表来源**
- [docker-compose.yaml:11-21](file://docker-compose.yaml#L11-L21)

### 站点配置文件结构

#### 主要站点配置

```mermaid
flowchart TD
START[站点配置开始] --> SERVER_BLOCK[server 块配置]
SERVER_BLOCK --> LISTEN_CONFIG[监听配置]
LISTEN_CONFIG --> PORT_80[listen 80]
LISTEN_CONFIG --> PORT_443[listen 443 ssl]
SERVER_BLOCK --> SERVER_NAME[server_name 配置]
SERVER_NAME --> DOMAIN[域名配置]
SERVER_NAME --> ALIAS[别名配置]
SERVER_BLOCK --> STATIC_CONFIG[静态资源配置]
STATIC_CONFIG --> ROOT_DIR[root 目录]
STATIC_CONFIG --> INDEX_FILES[index 文件]
STATIC_CONFIG --> FAVICON[favicon.ico]
SERVER_BLOCK --> PROXY_CONFIG[反向代理配置]
PROXY_CONFIG --> API_PROXY[API 代理]
PROXY_CONFIG --> WS_PROXY[WebSocket 代理]
SERVER_BLOCK --> CACHE_CONFIG[缓存配置]
CACHE_CONFIG --> STATIC_CACHE[静态资源缓存]
CACHE_CONFIG --> API_CACHE[API 响应缓存]
SERVER_BLOCK --> LOG_CONFIG[日志配置]
LOG_CONFIG --> ACCESS_LOG[访问日志]
LOG_CONFIG --> ERROR_LOG[错误日志]
SERVER_BLOCK --> END[站点配置结束]
```

**图表来源**
- [backend/app/api/main.py:121-136](file://backend/app/api/main.py#L121-L136)

### 静态资源服务配置

#### 前端构建产物服务

根据项目配置，前端构建产物位于 `frontend/dist` 目录，需要配置静态文件服务：

```mermaid
graph LR
subgraph "静态文件服务"
DIST[frontend/dist]
ASSETS[assets 目录]
INDEX[index.html]
ASSETS --> STATIC_ASSETS[静态资源]
INDEX --> SPA_FALLBACK[SPA 回退]
STATIC_ASSETS --> CACHE[缓存控制]
SPA_FALLBACK --> CACHE
CACHE --> COMPRESS[Gzip 压缩]
CACHE --> EXPIRES[过期时间]
end
subgraph "Nginx 配置"
NGINX_DIST[alias /app/frontend/dist]
NGINX_ASSETS[location /assets]
NGINX_SPA[location /]
NGINX_DIST --> DIST
NGINX_ASSETS --> ASSETS
NGINX_SPA --> INDEX
end
```

**图表来源**
- [backend/app/api/main.py:121-136](file://backend/app/api/main.py#L121-L136)

#### 缓存策略配置

```mermaid
flowchart TD
CACHE_REQUEST[缓存请求] --> CACHE_CHECK{检查缓存}
CACHE_CHECK --> |命中| RETURN_CACHE[返回缓存]
CACHE_CHECK --> |未命中| FETCH_BACKEND[获取后端]
FETCH_BACKEND --> STORE_CACHE[存储缓存]
STORE_CACHE --> SET_EXPIRES[设置过期时间]
SET_EXPIRES --> RETURN_RESPONSE[返回响应]
RETURN_CACHE --> END[结束]
RETURN_RESPONSE --> END
```

**图表来源**
- [backend/app/api/main.py:121-136](file://backend/app/api/main.py#L121-L136)

### HTTPS 配置

#### SSL 证书配置

```mermaid
graph TD
SSL_CONFIG[SSL 配置] --> CERTIFICATE[SSL 证书]
SSL_CONFIG --> PRIVATE_KEY[私钥文件]
SSL_CONFIG --> CA_CERT[CA 证书链]
CERTIFICATE --> FULLCHAIN[fullchain.pem]
PRIVATE_KEY --> PRIVKEY[privkey.pem]
CA_CERT --> CHAIN[chain.pem]
SSL_CONFIG --> TLS_VERSION[TLS 版本配置]
TLS_VERSION --> TLS_1_2[TLS 1.2]
TLS_VERSION --> TLS_1_3[TLS 1.3]
SSL_CONFIG --> CIPHER_SUITE[加密套件]
CIPHER_SUITE --> SECURE_CIPHERS[安全套件]
CIPHER_SUITE --> BACKWARD_COMPAT[向后兼容]
SSL_CONFIG --> OCSP_STAPLING[OCSP Stapling]
SSL_CONFIG --> HSTS[HSTS 配置]
```

**图表来源**
- [docker-compose.yaml:11-21](file://docker-compose.yaml#L11-L21)

#### TLS 版本和加密套件选择

根据现代安全最佳实践，建议配置：

- **TLS 版本**: TLS 1.2 和 TLS 1.3
- **加密套件**: 优先使用 ECDHE 密码套件
- **禁用弱加密**: 禁用 RC4、3DES 等弱加密算法
- **启用 OCSP Stapling**: 提升证书验证性能

### 负载均衡配置

#### upstream 服务器组配置

```mermaid
graph LR
subgraph "客户端请求"
CLIENT[客户端]
end
subgraph "负载均衡器"
LB[Nginx 负载均衡]
end
subgraph "后端服务器组"
SERVER1[Server 1: 7860]
SERVER2[Server 2: 7860]
SERVER3[Server 3: 7860]
end
subgraph "健康检查"
HEALTH[健康检查]
FAIL[故障转移]
end
CLIENT --> LB
LB --> SERVER1
LB --> SERVER2
LB --> SERVER3
HEALTH --> SERVER1
HEALTH --> SERVER2
HEALTH --> SERVER3
SERVER1 -.-> FAIL
SERVER2 -.-> FAIL
SERVER3 -.-> FAIL
```

**图表来源**
- [docker-compose.yaml:11-21](file://docker-compose.yaml#L11-L21)

#### 负载均衡算法选择

根据应用特点，推荐以下算法：

- **轮询算法**: 默认算法，适合服务器性能相近的情况
- **最少连接**: 适合请求处理时间差异较大的情况
- **IP 哈希**: 适合需要粘性会话的应用

### 反向代理规则配置

#### API 路由转发

```mermaid
sequenceDiagram
participant Client as 客户端
participant Nginx as Nginx
participant API as FastAPI
Client->>Nginx : /api/trip/plan
Nginx->>Nginx : 路径匹配
Nginx->>API : 转发到 127.0.0.1 : 7860
API->>API : 处理请求
API-->>Nginx : 返回响应
Nginx-->>Client : 返回响应
Note over Client,Nginx : WebSocket 连接
Client->>Nginx : /api/trip/ws/{task_id}
Nginx->>Nginx : 升级协议
Nginx->>API : 转发 WebSocket
API-->>Nginx : WebSocket 流
Nginx-->>Client : 实时推送
```

**图表来源**
- [backend/app/api/routes/trip.py:390-440](file://backend/app/api/routes/trip.py#L390-L440)
- [frontend/src/services/api.ts:268-318](file://frontend/src/services/api.ts#L268-L318)

#### 请求头处理

```mermaid
flowchart TD
REQUEST[HTTP 请求] --> HEADER_PROCESS[请求头处理]
HEADER_PROCESS --> PROXY_PASS[代理传递]
HEADER_PROCESS --> CUSTOM_HEADER[自定义头部]
PROXY_PASS --> X_FORWARDED_FOR[X-Forwarded-For]
PROXY_PASS --> X_FORWARDED_PROTO[X-Forwarded-Proto]
PROXY_PASS --> X_REAL_IP[X-Real-IP]
CUSTOM_HEADER --> API_KEY[API Key]
CUSTOM_HEADER --> AUTH_TOKEN[认证令牌]
CUSTOM_HEADER --> TRACE_ID[追踪 ID]
PROXY_PASS --> API_SERVER[API 服务器]
CUSTOM_HEADER --> API_SERVER
```

**图表来源**
- [backend/app/api/main.py:33-44](file://backend/app/api/main.py#L33-L44)

### 缓存策略配置

#### 静态资源缓存

```mermaid
graph TD
subgraph "静态资源缓存策略"
HTML_CACHE[HTML: 0s 缓存]
CSS_CACHE[CSS: 1年缓存]
JS_CACHE[JavaScript: 1年缓存]
IMG_CACHE[图片: 1年缓存]
FONT_CACHE[字体: 1年缓存]
HTML_CACHE --> CACHE_CONTROL[Cache-Control: no-cache]
CSS_CACHE --> CACHE_CONTROL
JS_CACHE --> CACHE_CONTROL
IMG_CACHE --> CACHE_CONTROL
FONT_CACHE --> CACHE_CONTROL
CACHE_CONTROL --> EXPIRES[Expires: 1年]
end
subgraph "动态内容缓存"
API_CACHE[API 响应: 5-60s]
WS_CACHE[WebSocket: 无缓存]
UPLOAD_CACHE[上传文件: 1小时]
API_CACHE --> CACHE_CONTROL
WS_CACHE --> CACHE_CONTROL
UPLOAD_CACHE --> CACHE_CONTROL
end
```

**图表来源**
- [backend/app/api/main.py:121-136](file://backend/app/api/main.py#L121-L136)

#### API 响应缓存

```mermaid
flowchart TD
API_REQUEST[API 请求] --> CACHE_CHECK{检查缓存}
CACHE_CHECK --> |命中| RETURN_CACHE[返回缓存]
CACHE_CHECK --> |未命中| FETCH_DATA[获取数据]
FETCH_DATA --> CACHE_RESPONSE[缓存响应]
CACHE_RESPONSE --> SET_TTL[设置 TTL]
SET_TTL --> RETURN_RESPONSE[返回响应]
RETURN_CACHE --> END[结束]
RETURN_RESPONSE --> END
subgraph "缓存键生成"
URL_HASH[URL 哈希]
QUERY_PARAMS[查询参数]
HEADERS[请求头]
USER_CONTEXT[用户上下文]
end
CACHE_CHECK --> URL_HASH
CACHE_CHECK --> QUERY_PARAMS
CACHE_CHECK --> HEADERS
CACHE_CHECK --> USER_CONTEXT
```

**图表来源**
- [backend/app/api/routes/trip.py:243-274](file://backend/app/api/routes/trip.py#L243-L274)

### WebSocket 支持配置

#### WebSocket 协议升级

```mermaid
sequenceDiagram
participant Client as 客户端
participant Nginx as Nginx
participant WebSocket as WebSocket 服务器
Client->>Nginx : Upgrade : websocket
Nginx->>Nginx : 检查 Upgrade 头
Nginx->>Nginx : 设置代理参数
Nginx->>WebSocket : 转发升级请求
WebSocket->>WebSocket : 建立连接
WebSocket-->>Nginx : 连接确认
Nginx-->>Client : 协议升级成功
loop 实时通信
Client->>Nginx : 发送消息
Nginx->>WebSocket : 转发消息
WebSocket->>Nginx : 返回消息
Nginx-->>Client : 转发消息
end
Client->>Nginx : 关闭连接
Nginx->>WebSocket : 通知关闭
WebSocket-->>Nginx : 确认关闭
Nginx-->>Client : 关闭确认
```

**图表来源**
- [backend/app/api/routes/trip.py:390-440](file://backend/app/api/routes/trip.py#L390-L440)
- [frontend/src/services/api.ts:268-318](file://frontend/src/services/api.ts#L268-L318)

## 依赖分析

### 组件耦合关系

```mermaid
graph TB
subgraph "前端依赖"
VUE[Vue 3]
AXIOS[Axios]
ROUTER[Vue Router]
I18N[Vue I18n]
end
subgraph "后端依赖"
FASTAPI[FastAPI]
UVICORN[Uvicorn]
GUNICORN[Gunicorn]
PYDANTIC[Pydantic]
ASYNCIO[Asyncio]
end
subgraph "Nginx 依赖"
NGINX[Nginx]
SSL[SSL/TLS]
GEOIP[GEOIP]
CACHE[缓存模块]
HTTP2[HTTP/2]
end
subgraph "外部服务"
LLM[LLM API]
MAP[高德地图]
XHS[小红书]
CDN[CDN 服务]
end
VUE --> AXIOS
AXIOS --> NGINX
ROUTER --> NGINX
I18N --> NGINX
NGINX --> FASTAPI
NGINX --> SSL
NGINX --> GEOIP
NGINX --> CACHE
NGINX --> HTTP2
FASTAPI --> LLM
FASTAPI --> MAP
FASTAPI --> XHS
FASTAPI --> CDN
```

**图表来源**
- [backend/app/api/main.py:138-147](file://backend/app/api/main.py#L138-L147)
- [docker-compose.yaml:1-24](file://docker-compose.yaml#L1-24)

### 数据流分析

```mermaid
flowchart TD
subgraph "用户请求流"
USER[用户请求] --> FRONTEND[前端应用]
FRONTEND --> API[API 端点]
API --> BUSINESS[业务逻辑]
BUSINESS --> EXTERNAL[外部服务]
EXTERNAL --> RESPONSE[响应返回]
RESPONSE --> FRONTEND
FRONTEND --> USER
end
subgraph "静态资源流"
STATIC_REQ[静态资源请求] --> NGINX_STATIC[Nginx 静态服务]
NGINX_STATIC --> CACHE[缓存层]
CACHE --> CDN[CDN 分发]
CDN --> USER
end
subgraph "实时通信流"
WS_CLIENT[WebSocket 客户端] --> WS_PROXY[WebSocket 代理]
WS_PROXY --> WS_SERVER[WebSocket 服务器]
WS_SERVER --> WS_CLIENT
end
```

**图表来源**
- [backend/app/api/main.py:121-136](file://backend/app/api/main.py#L121-L136)
- [backend/app/api/routes/trip.py:390-440](file://backend/app/api/routes/trip.py#L390-L440)

**章节来源**
- [backend/app/api/main.py:121-136](file://backend/app/api/main.py#L121-L136)
- [backend/app/api/routes/trip.py:390-440](file://backend/app/api/routes/trip.py#L390-L440)

## 性能考虑

### 缓存优化策略

#### 多层缓存架构

```mermaid
graph TD
subgraph "缓存层次"
CDN_CACHE[CDN 缓存]
EDGE_CACHE[边缘缓存]
NGINX_CACHE[Nginx 缓存]
APP_CACHE[应用缓存]
DATABASE_CACHE[数据库缓存]
end
subgraph "缓存策略"
STATIC_CACHE[静态资源缓存]
API_CACHE[API 响应缓存]
IMAGE_CACHE[图片缓存]
SEARCH_CACHE[搜索结果缓存]
end
CDN_CACHE --> STATIC_CACHE
EDGE_CACHE --> STATIC_CACHE
NGINX_CACHE --> API_CACHE
APP_CACHE --> SEARCH_CACHE
DATABASE_CACHE --> SEARCH_CACHE
```

#### 性能监控指标

- **响应时间**: 目标 < 200ms
- **并发连接数**: 支持 > 1000 concurrent connections
- **吞吐量**: > 100 requests/second
- **缓存命中率**: > 90%
- **CPU 使用率**: < 70%
- **内存使用率**: < 80%

### 压缩配置

#### Gzip 压缩策略

```mermaid
flowchart TD
CONTENT[响应内容] --> COMPRESSION_CHECK{检查压缩}
COMPRESSION_CHECK --> |HTML/CSS/JS| ENABLE_GZIP[启用 Gzip]
COMPRESSION_CHECK --> |图片/视频| SKIP_COMPRESSION[跳过压缩]
ENABLE_GZIP --> COMPRESSION_LEVEL[压缩级别]
COMPRESSION_LEVEL --> LEVEL_1[Level 1 - 最快]
COMPRESSION_LEVEL --> LEVEL_6[Level 6 - 平衡]
COMPRESSION_LEVEL --> LEVEL_9[Level 9 - 最佳]
ENABLE_GZIP --> BUFFER_SIZE[缓冲区大小]
BUFFER_SIZE --> SMALL_BUFFER[Small Buffer]
BUFFER_SIZE --> LARGE_BUFFER[Large Buffer]
ENABLE_GZIP --> MIN_LENGTH[最小长度]
MIN_LENGTH --> MIN_100B[100 bytes]
MIN_LENGTH --> MIN_1KB[1 KB]
MIN_LENGTH --> MIN_10KB[10 KB]
```

**图表来源**
- [backend/app/api/main.py:121-136](file://backend/app/api/main.py#L121-L136)

## 故障排除指南

### 常见问题诊断

#### 连接问题排查

```mermaid
flowchart TD
CONNECTION_ERROR[连接错误] --> CHECK_NGINX[检查 Nginx 状态]
CHECK_NGINX --> NGINX_RUNNING{Nginx 是否运行}
NGINX_RUNNING --> |否| START_NGINX[启动 Nginx]
NGINX_RUNNING --> |是| CHECK_PORT[检查端口监听]
CHECK_PORT --> PORT_LISTENING{端口是否监听}
PORT_LISTENING --> |否| CONFIG_PORT[配置端口]
PORT_LISTENING --> |是| CHECK_PROXY[检查代理配置]
CHECK_PROXY --> PROXY_WORKING{代理是否工作}
PROXY_WORKING --> |否| FIX_PROXY[修复代理配置]
PROXY_WORKING --> |是| CHECK_BACKEND[检查后端服务]
CHECK_BACKEND --> BACKEND_HEALTH{后端是否健康}
BACKEND_HEALTH --> |否| START_BACKEND[启动后端服务]
BACKEND_HEALTH --> |是| CHECK_FIREWALL[检查防火墙]
```

#### 性能问题排查

```mermaid
flowchart TD
PERFORMANCE_ISSUE[性能问题] --> CHECK_LOAD[检查系统负载]
CHECK_LOAD --> LOAD_HIGH{负载过高?}
LOAD_HIGH --> |是| OPTIMIZE_RESOURCES[优化资源使用]
LOAD_HIGH --> |否| CHECK_CONNECTIONS[检查连接数]
CHECK_CONNECTIONS --> TOO_MANY_CONN{连接数过多?}
TOO_MANY_CONN --> |是| LIMIT_CONNECTIONS[限制连接数]
TOO_MANY_CONN --> |否| CHECK_CACHE[检查缓存效率]
CHECK_CACHE --> LOW_HIT_RATE{缓存命中率低?}
LOW_HIT_RATE --> |是| TUNE_CACHE[调整缓存策略]
LOW_HIT_RATE --> |否| CHECK_DATABASE[检查数据库性能]
CHECK_DATABASE --> SLOW_QUERY{慢查询?}
SLOW_QUERY --> |是| OPTIMIZE_QUERIES[优化查询]
SLOW_QUERY --> |否| CHECK_NETWORK[检查网络延迟]
```

### 日志分析

#### Nginx 日志配置

```mermaid
graph LR
subgraph "访问日志"
ACCESS_LOG[access.log]
LOG_FORMAT[自定义日志格式]
LOG_ROTATION[日志轮转]
end
subgraph "错误日志"
ERROR_LOG[error.log]
DEBUG_LOG[调试日志]
WARN_LOG[警告日志]
end
subgraph "分析工具"
AWSTATS[AWStats]
GOACCESS[GoAccess]
ELK_STACK[ELK Stack]
end
ACCESS_LOG --> LOG_FORMAT
ACCESS_LOG --> LOG_ROTATION
ERROR_LOG --> DEBUG_LOG
ERROR_LOG --> WARN_LOG
LOG_FORMAT --> AWSTATS
LOG_ROTATION --> AWSTATS
DEBUG_LOG --> GOACCESS
WARN_LOG --> GOACCESS
```

**图表来源**
- [docker-compose.yaml:11-21](file://docker-compose.yaml#L11-L21)

**章节来源**
- [docker-compose.yaml:11-21](file://docker-compose.yaml#L11-L21)

## 结论

本文档提供了基于 TripStar 项目的完整 Nginx 反向代理配置指南。通过合理配置 Nginx，可以实现：

1. **高性能静态资源服务**: 通过多级缓存和压缩提升用户体验
2. **可靠的 API 代理**: 支持 HTTP 和 WebSocket 协议
3. **安全的 HTTPS 传输**: 配置现代 TLS 版本和加密套件
4. **灵活的负载均衡**: 支持多种算法和健康检查
5. **完善的缓存策略**: 针对静态资源和动态内容的不同缓存需求

建议在生产环境中实施以下最佳实践：
- 定期更新 SSL 证书和加密套件
- 监控系统性能指标和缓存效果
- 配置适当的超时和重试机制
- 实施安全防护措施和访问控制
- 建立完善的日志记录和分析体系

## 附录

### 配置文件模板

#### 基础 Nginx 配置模板

```nginx
# 基础配置
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# 事件模块
events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

# HTTP 核心配置
http {
    # 基本设置
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                   '$status $body_bytes_sent "$http_referer" '
                   '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main;
    
    # 性能优化
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    # Gzip 压缩
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_comp_level 6;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
    
    # 包含其他配置
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/http.d/*.conf;
}
```

#### 站点配置模板

```nginx
# 站点配置
server {
    # 监听配置
    listen 80;
    listen 443 ssl http2;
    server_name tripstar.example.com www.tripstar.example.com;
    
    # SSL 配置
    ssl_certificate /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;
    ssl_trusted_certificate /path/to/chain.pem;
    
    # TLS 配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # 安全头
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    
    # 静态资源配置
    location / {
        root /app/frontend/dist;
        try_files $uri $uri/ /index.html;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
    
    # API 代理配置
    location /api/ {
        proxy_pass http://127.0.0.1:7860/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 超时设置
        proxy_connect_timeout 300s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
        
        # 缓冲设置
        proxy_buffering on;
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
        proxy_busy_buffers_size 256k;
    }
    
    # WebSocket 配置
    location /api/trip/ws/ {
        proxy_pass http://127.0.0.1:7860/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket 超时
        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }
    
    # 健康检查
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

#### 负载均衡配置模板

```nginx
# 负载均衡配置
upstream tripstar_backend {
    # 轮询算法
    server 127.0.0.1:7860 weight=1 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:7861 weight=1 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:7862 weight=1 max_fails=3 fail_timeout=30s;
    
    # 健康检查
    keepalive 32;
}

# 负载均衡服务器配置
server {
    listen 80;
    server_name tripstar.example.com;
    
    # 负载均衡代理
    location / {
        proxy_pass http://tripstar_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 连接池
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        # 健康检查
        proxy_next_upstream on;
        proxy_next_upstream_timeout 0;
        proxy_next_upstream_tries 3;
    }
}
```

### 性能调优建议

#### 系统级优化

```bash
# 文件描述符限制
echo '* soft nofile 65536' >> /etc/security/limits.conf
echo '* hard nofile 65536' >> /etc/security/limits.conf

# 内核参数优化
echo 'net.core.somaxconn = 65535' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_max_syn_backlog = 65535' >> /etc/sysctl.conf
echo 'net.ipv4.ip_local_port_range = 1024 65535' >> /etc/sysctl.conf

# 应用程序优化
ulimit -n 65536
sysctl -p
```

#### Nginx 优化配置

```nginx
# worker 进程优化
worker_processes auto;
worker_cpu_affinity auto;
worker_rlimit_nofile 65536;

# 事件模型优化
events {
    use epoll;
    worker_connections 65536;
    multi_accept on;
    accept_mutex off;
}

# HTTP 优化
http {
    # 连接池优化
    keepalive_timeout 65;
    keepalive_requests 1000;
    
    # 缓冲区优化
    client_body_buffer_size 128k;
    client_max_body_size 10m;
    client_body_timeout 300s;
    
    # 发送缓冲区优化
    send_timeout 300s;
    send_lowat 16384;
    
    # 网络优化
    tcp_nopush on;
    tcp_nodelay on;
}
```

**章节来源**
- [docker-compose.yaml:11-21](file://docker-compose.yaml#L11-L21)
- [backend/app/api/main.py:121-136](file://backend/app/api/main.py#L121-L136)
- [backend/app/api/routes/trip.py:390-440](file://backend/app/api/routes/trip.py#L390-L440)