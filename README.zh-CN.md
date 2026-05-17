# codex-gateway

[English](./README.md)

`codex-gateway` 是一个便携的 Docker Compose 启动包，用来通过 `VPS + CLIProxyAPI/CPA + Cloudflare` 转发 Codex 兼容的 code agent 请求。适合已经持有域名、已经接入 Cloudflare、希望快速落一套自托管网关的使用场景。

这个仓库不负责自动创建 VPS、Cloudflare 或 Nginx 配置，它提供的是最小可部署运行时：`config.yaml`、`docker-compose.yml`、持久化目录，以及一个可直接合并到现有边缘代理中的 Nginx 示例配置。

## 项目作用

- 用 `codex.abc.com` 暴露 Codex 兼容的 `/v1` API
- 用 `cpam.abc.com` 暴露 CPA Manager 管理页面
- 将容器端口限制在 `127.0.0.1`，由 Nginx 和 Cloudflare 对外提供访问
- 提供一套可重复使用的认证、日志和本地数据目录结构

## 部署流程

### 1. 前置条件

- 一台安装了 Docker Engine 和 Docker Compose 插件的 Linux VPS
- 一个已经托管到 Cloudflare 的域名，例如 `abc.com`
- VPS 上已有 HTTPS 反向代理。仓库中的 [nginx.sample.conf](./nginx.sample.conf) 是参考配置。
- VPS 上已安装 `openssl`

### 2. 使用 OpenSSL 生成管理密钥和客户端 API Key

先生成两个随机值：

```bash
MANAGEMENT_KEY="$(openssl rand -hex 32)"
CLIENT_API_KEY="$(openssl rand -hex 32)"

printf 'Management Key: %s\n' "$MANAGEMENT_KEY"
printf 'Client API Key: %s\n' "$CLIENT_API_KEY"
```

然后编辑 [config.yaml](./config.yaml)，替换以下两个占位符：

- `CHANGE_ME_MANAGEMENT_KEY` -> `MANAGEMENT_KEY`
- `CHANGE_ME_CLIENT_API_KEY` -> `CLIENT_API_KEY`

这里有一个和当前仓库实现强相关的细节：`docker-compose.yml` 里的 CPA Manager 还会从 `./secrets/cpa_management_key` 读取管理密钥，所以第一次启动前还要把同一个明文管理密钥写进去：

```bash
mkdir -p auths data logs secrets
printf '%s' "$MANAGEMENT_KEY" > secrets/cpa_management_key
chmod 600 secrets/cpa_management_key
```

注意：CLIProxyAPI 首次启动时可能会把 `config.yaml` 里的管理密钥改写成哈希值，所以一定要把最初的明文 `MANAGEMENT_KEY` 另外保存好。后面登录管理页面时，仍然要输入这个明文值。

### 3. 启动 Docker Compose

```bash
docker compose up -d
docker compose logs -f --tail=100
```

可以先做本地检查：

```bash
curl -i http://127.0.0.1:8317/v1/models
curl -i http://127.0.0.1:18317/health
```

这一阶段服务应该只监听本机回环地址，不要把 `8317`、`18317`、`1455` 直接暴露到公网。

### 4. 接入 Nginx

把 [nginx.sample.conf](./nginx.sample.conf) 里的两个 `server` 块合并到你现有的 Nginx 配置中，并按你的环境修改证书路径。

目标转发关系如下：

- `codex.abc.com` -> `127.0.0.1:8317`
- `cpam.abc.com` -> `127.0.0.1:18317`

合并完成后重载 Nginx。

### 5. 添加 Cloudflare DNS

在 Cloudflare 中增加两条 DNS 记录：

- `A` 记录：`codex.abc.com` -> 你的 VPS 公网 IP
- `A` 记录：`cpam.abc.com` -> 你的 VPS 公网 IP

建议的 Cloudflare 设置：

- 两条记录都开启橙云代理
- `SSL/TLS` 设为 `Full (strict)`
- 对 `codex.abc.com/*` 和 `cpam.abc.com/*` 配置绕过缓存
- 不要给 `codex.abc.com` 增加交互式挑战页，否则 CLI 客户端容易失败

### 6. 管理界面截图

![Cloudfalre DNS Config](./docs/images/cloudflare-dns-config.jpg)

![Cloudfalre Security Rule](./docs/images/cloudflare-security-rule.jpg)

### 7. 登录服务

浏览器打开：

```text
https://cpam.abc.com/
```

按提示输入前面生成的原始明文 `CHANGE_ME_MANAGEMENT_KEY`，即可完成管理登录。

### 8. 登录 Codex OAuth

在管理页面中继续发起 Codex OAuth 登录流程。登录过程中你会拿到一个很长的回调地址，格式类似：

```text
http://localhost:1455/...
```

把这整条 `localhost:1455` 回调地址完整复制回页面中。凭据写入 `auths/` 后，整个服务部署就完成了。

## 运维提示

- `auths/`、`data/`、`logs/`、`secrets/` 都属于敏感目录，不要提交到 Git，也不要通过公网目录暴露。
- `1455` 只用于 OAuth 回调，应始终保持为 `127.0.0.1` 本地监听。
- 如果你希望给 `cpam.abc.com` 再加一层防护，可以放在 Cloudflare Access 后面。

## 致谢

这套部署方式直接受益于以下开源项目：

- [router-for-me/CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI)
- [seakee/CPA-Manager](https://github.com/seakee/CPA-Manager)
- [nginx/nginx](https://github.com/nginx/nginx)
- [docker/compose](https://github.com/docker/compose)
