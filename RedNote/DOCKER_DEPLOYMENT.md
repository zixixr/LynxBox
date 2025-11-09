# 小红书 MCP Docker 部署指南

## 项目简介

这是一个用于小红书（Xiaohongshu）的 MCP（Model Context Protocol）服务器，支持通过 AI 助手自动发布内容、搜索、评论等功能。

## 功能特性

- ✅ 登录认证与会话管理
- ✅ 发布图文内容（支持标题、正文、话题标签）
- ✅ 发布视频内容
- ✅ 搜索小红书内容
- ✅ 获取首页推荐
- ✅ 查看笔记详情
- ✅ 发表评论
- ✅ 查看用户主页

## 前置要求

1. **Docker** 和 **Docker Compose** 已安装
   - Windows: [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop)
   - macOS: [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop)
   - Linux: 安装 Docker Engine 和 Docker Compose

2. **小红书账号** 和 **小红书 App**（用于扫码登录）

## 快速部署步骤

### 1. 创建项目目录

```bash
# 在 RedNote 目录下（已有 docker-compose.yml）
cd D:\AIDev\LynxBox\RedNote

# 创建必要的目录
mkdir images
mkdir cookies
```

### 2. 拉取 Docker 镜像

```bash
docker pull xpzouying/xiaohongshu-mcp:latest
```

### 3. 启动服务

```bash
# 启动容器（后台运行）
docker compose up -d

# 查看日志（确认服务启动成功）
docker compose logs -f
```

### 4. 登录小红书账号

服务启动后，需要扫码登录小红书账号：

**方法 1：使用 MCP Inspector（推荐）**

1. 安装 MCP Inspector:
   ```bash
   npx @modelcontextprotocol/inspector
   ```

2. 在浏览器中打开 `http://localhost:5173`

3. 连接到 MCP 服务器：
   - Transport Type: 选择 `HTTP`
   - URL: `http://localhost:18060/mcp`（如果在远程服务器上，替换为服务器IP）

4. 在工具列表中找到 `check_login_status`，点击执行

5. 如果未登录，会显示二维码，用小红书 App 扫码登录

**方法 2：使用 Claude Code**

在 Claude Code 中配置 MCP 服务器后，执行登录命令。

## 目录结构说明

```
RedNote/
├── docker-compose.yml       # Docker Compose 配置文件
├── images/                  # 存储要发布的图片（挂载到容器的 /app/images）
├── cookies/                 # 保存登录 cookie（自动生成）
└── DOCKER_DEPLOYMENT.md     # 本文档
```

## 重要配置说明

### 端口映射

- 服务监听端口: `18060`
- 访问地址: `http://localhost:18060/mcp`

### 数据持久化

1. **images/** 目录：
   - 用途：存放要发布的图片
   - 重要：使用本地图片时，必须放在此目录下，并在 MCP 中指定路径为 `/app/images/你的图片名.jpg`
   - 支持格式：JPG, PNG（推荐使用本地图片，比 HTTP URL 更稳定快速）

2. **cookies/** 目录：
   - 用途：保存小红书登录会话
   - 重要：保持登录状态，避免频繁扫码

## 常用命令

```bash
# 查看容器状态
docker compose ps

# 查看实时日志
docker compose logs -f

# 停止服务
docker compose stop

# 启动服务
docker compose start

# 重启服务
docker compose restart

# 停止并删除容器
docker compose down

# 更新镜像并重启
docker compose pull
docker compose up -d

# 进入容器内部（调试用）
docker exec -it xiaohongshu-mcp bash
```

## 集成到 Claude Code

在你的 `.claude/settings.local.json` 中添加 MCP 配置：

```json
{
  "mcpServers": {
    "xiaohongshu": {
      "url": "http://localhost:18060/mcp",
      "transport": "http"
    }
  }
}
```

然后在 Claude Code 中就可以使用小红书相关功能了。

## 使用限制和注意事项

1. **内容限制**：
   - 标题最多 20 个字
   - 正文不超过 1000 个字
   - 每天大约可发布 50 条内容

2. **登录限制**：
   - 同一账号不允许在多个网页端同时登录
   - 避免在多个地方使用同一账号，会导致会话冲突

3. **图片使用**：
   - 推荐使用本地图片（放在 `./images/` 目录）
   - HTTP URL 可能不稳定

4. **网络要求**：
   - 需要能够访问小红书服务
   - 如在国外服务器部署，可能需要代理

## 故障排查

### 1. 容器无法启动

```bash
# 查看详细日志
docker compose logs

# 检查端口占用
netstat -ano | findstr :18060  # Windows
lsof -i :18060                 # macOS/Linux
```

### 2. 登录失败

- 确保小红书 App 已安装并可以正常使用
- 检查网络连接
- 重启容器重新登录

### 3. 图片上传失败

- 确保图片放在 `./images/` 目录下
- 检查图片路径是否正确（容器内路径为 `/app/images/`）
- 确保图片格式正确（JPG, PNG）

### 4. MCP 连接失败

- 检查服务是否正常运行：`docker compose ps`
- 检查端口映射是否正确
- 如果使用远程服务器，确保防火墙允许 18060 端口

## 更多资源

- 项目仓库: https://github.com/xpzouying/xiaohongshu-mcp
- MCP 协议: https://modelcontextprotocol.io/

## 技术支持

如有问题，可以：
1. 查看项目的 GitHub Issues
2. 参考项目 README 中的详细文档
3. 加入项目社区群获取帮助
