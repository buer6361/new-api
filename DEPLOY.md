# 部署流程

前后端一体构建，在本地 Mac 编译，上传服务器部署。

## 前置条件

```bash
# 本地需安装
brew install oven-sh/bun/bun   # 前端构建
go version                      # Go ≥ 1.23（后端编译）
```

## 完整升级流程

### 1. 拉取最新代码

```bash
cd ~/Desktop/new-api
git checkout dev
git pull origin dev        # 拉你自己的提交
# 或
git fetch upstream         # 拉上游 New API 更新
git merge upstream/main    # 合并到 dev
```

### 2. 构建前端

```bash
# Classic 主题
cd web/classic
bun install --legacy-peer-deps
bun run build

# Default 主题（新版前端）
cd ../default
bun install
bun run build
```

### 3. 编译 Go 二进制（Linux 目标）

```bash
cd ~/Desktop/new-api
GOOS=linux GOARCH=amd64 go build -o /tmp/new-api-custom .
```

### 4. 上传到服务器

```bash
gzip -c /tmp/new-api-custom | ssh root@8.210.130.59 \
  'cat > /tmp/new-api-custom.gz && gunzip -f /tmp/new-api-custom.gz && chmod +x /tmp/new-api-custom'
```

### 5. 部署到 Docker

```bash
ssh root@8.210.130.59

# 停旧容器
docker stop new-api && docker rm new-api

# 构建自定义镜像（基于官方镜像替换二进制）
docker run -d --name new-api-tmp calciumion/new-api:latest sleep 30
docker cp /tmp/new-api-custom new-api-tmp:/new-api
docker commit new-api-tmp new-api-custom:latest
docker rm -f new-api-tmp

# 启动
docker run -d \
  --name new-api \
  --restart always \
  --network host \
  -e TZ=Asia/Shanghai \
  -e SQL_DSN='postgres://newapi:newapi_pass_2025@127.0.0.1:5432/newapi?sslmode=disable' \
  -v /opt/new-api/data:/data \
  new-api-custom:latest

# 验证
sleep 8
curl -s http://127.0.0.1:3000/ | grep '<title>'
```

### 6. 验证外部访问

```bash
curl -s https://tokens.hanfuvela.com | grep '<title>'
# 应输出: <title>TokensVela</title>
```

## 仅改前端文案（不编译 Go）

如果只改前端展示文案（Footer、About 等），改完后重跑构建和部署即可。

修改的文件通常在：
- `web/classic/src/components/layout/Footer.jsx`
- `web/classic/src/pages/About/index.jsx`
- `web/default/src/components/layout/components/footer.tsx`
- `web/default/src/features/about/index.tsx`
- `web/default/index.html`（标题）
- `web/classic/index.html`（标题）

改完执行步骤 2→3→4→5。

## 同步上游更新

```bash
git fetch upstream
git checkout dev
git merge upstream/main
# 解决冲突后
git push origin dev
# 按上面流程重新构建部署
```

## 服务器环境（备忘）

| 项目 | 内容 |
|------|------|
| 服务器 | 8.210.130.59（阿里云 ECS，Ubuntu 22.04） |
| Docker | 自定义镜像 `new-api-custom:latest` |
| 数据库 | PostgreSQL 16，用户 `newapi`，库 `newapi` |
| 域名 | tokens.hanfuvela.com（Cloudflare 代理） |
| nginx | 反向代理到 localhost:3000 |
| 管理员 | admin / admin123456（首次登录后请改密码） |
