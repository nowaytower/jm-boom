# JM Boom

当前分支是「Rust 服务端 + React 前端」的前后端分离项目，有两种用法：

- **便携版**：下载对应平台的压缩包，解压双击/`./run.sh` 即可，浏览器打开 <http://localhost:3000>（见[下载](#下载多平台发行包)）
- **Docker**：服务器上跑一个容器，适合长期挂机和多设备访问（见[Docker 部署](#docker-部署)）

## 下载（多平台发行包）

每次发版（推 `v*.*.*` tag）由 GitHub Actions 自动构建 5 个平台的原生包，全部挂在 [Releases](https://github.com/nowaytower/jm-boom/releases) 下，不需要自己在本地编译：

| 平台 | 文件 | 启动方式 |
|---|---|---|
| Windows 10/11 x64 | `JM-Boom-vX.Y.Z-Windows-x64.zip` | 解压后双击「启动 JM Boom.bat」 |
| Linux x64 | `JM-Boom-vX.Y.Z-Linux-x64.tar.gz` | 解压后 `./run.sh`（静态编译，不依赖 glibc） |
| Linux arm64 | `JM-Boom-vX.Y.Z-Linux-arm64.tar.gz` | 解压后 `./run.sh`（静态编译，树莓派/ARM 服务器通用） |
| macOS Apple Silicon | `JM-Boom-vX.Y.Z-macOS-arm64.tar.gz` | 解压后 `./run.sh` |
| macOS Intel | `JM-Boom-vX.Y.Z-macOS-x64.tar.gz` | 解压后 `./run.sh` |
| Docker（服务器） | `ghcr.io/nowaytower/jm-boom:X.Y.Z` | `docker compose up -d` |

每个包都自带服务端二进制、前端静态文件和 `README.txt`，**解压即用**，不依赖 Python / Node / 数据库。运行后浏览器访问 <http://localhost:3000>，账号与缓存写在同目录的 `data/` 里（想搬家就整目录拷走）。

```bash
# 校验下载是否完整（同一 Release 里的 SHA256SUMS.txt）
sha256sum -c SHA256SUMS.txt
```

**Windows 说明**

- 解压到普通目录即可，别放在 `C:\Program Files` 这类需要管理员权限的位置
- 双击「启动 JM Boom.bat」会开一个黑窗口，浏览器自动弹出；**关掉黑窗口服务就停了**
- 首次运行 Windows 防火墙可能询问是否允许联网，选「允许」；只在本机用可以取消勾选公用网络
- 二进制没有代码签名，SmartScreen 可能提示「未知发布者」→ 更多信息 → 仍要运行
- 想开机自启：把「启动 JM Boom.bat」的快捷方式放进 `shell:startup` 目录

**macOS 说明**

- 首次运行若提示「已损坏，无法打开」或「无法验证开发者」，在终端执行
  `xattr -dr com.apple.quarantine ./jm-boom-server` 再运行
- Apple Silicon 机器不要下 `macOS-x64`（会走 Rosetta，性能与兼容性都差）

**Linux 说明**

- Linux 包是 musl 静态链接，Debian 11 / Ubuntu 20.04 / Alpine 都能直接跑；只有 32 位系统需要改用 Docker
- 需要图形界面浏览器的场景（桌面 Linux）同样浏览器访问 3000 端口即可

## 发行流程（维护者）

版本号在三个地方保持一致：`server/Cargo.toml`、`server/Cargo.lock` 里 `jm-boom-server` 条目、`web/package.json`。改完版本号后：

```bash
git commit -am "chore(release): bump version to X.Y.Z"
git tag vX.Y.Z
git push origin main --tags
```

推 tag 会同时触发两个工作流：

- `release.yml`：构建 Windows x64 / Linux x64 / Linux arm64 / macOS arm64 / macOS x64 五个原生包 + `SHA256SUMS.txt`，创建 GitHub Release 并上传产物（跑在各自的官方 runner 上，无模拟）
- `docker-publish.yml`：构建 `ghcr.io/<仓库归属>/jm-boom` 多架构镜像（amd64 + arm64）

不想打 tag 也能试构建：Actions → `release` → Run workflow，填版本号、`publish` 保持关闭，产物只在本次运行的 Artifacts 里，不会动 Release。

## Docker 部署

公开镜像：

```text
ghcr.io/nowaytower/jm-boom
```

直接使用 Docker 运行（请替换访问密码）：

```bash
docker run -d \
  --name jm-boom \
  --restart unless-stopped \
  --security-opt no-new-privileges:true \
  -p 3000:3000 \
  -v jm-boom-data:/app/data \
  -e JM_BOOM_ACCESS_PASSWORD="change-this-password" \
  ghcr.io/nowaytower/jm-boom:latest
```

使用仓库中的 Compose 部署：

```bash
# 先编辑 docker-compose.yml 中的 JM_BOOM_ACCESS_PASSWORD
docker compose pull
docker compose up -d
```

默认访问地址：<http://localhost:3000>。

查看状态和日志：

```bash
docker compose ps
docker compose logs -f jm-boom
```

默认使用名为 `jm-boom-data` 的 Docker 命名卷持久化 SQLite、图片缓存和离线下载。升级或重建容器不会删除该命名卷。

升级：

```bash
docker compose pull
docker compose up -d
```

## 本地前端开发

使用 Node.js 24 和 pnpm（版本由 `web/package.json` 固定）：

```bash
corepack enable
cd web
pnpm install
pnpm dev
```

开发服务器将 `/api` 请求代理到 `http://127.0.0.1:3000`。类型、测试及构建验证使用 `pnpm typecheck`、`pnpm test`、`pnpm lint`、`pnpm fmt:check`、`pnpm build`。

## 运行时配置

- `RUST_LOG`：服务端日志过滤，默认 `info,jm_boom_server=info`。
- `JM_BOOM_ACCESS_PASSWORD`：进入 Web 界面前使用的轻量访问密码；部署前必须修改 Compose 中的默认值。
- `JM_BOOM_CACHE_MAX_MB`：封面与阅读图片共享的服务端缓存容量上限，单位 MB，默认 `5120`；达到上限后按 LRU 自动清理至容量的一半。
- `JM_BOOM_CORS_ORIGINS`：可选的逗号分隔跨域 Origin；同源部署不需要设置。

## NSFW 警告

本软件可能存在裸露、暴力、色情或冒犯等不适宜公众场合的内容，请勿在公共场合使用本软件，避免不必要的纷争。

## 致谢

本项目参考了以下项目的部分实现，在此表示衷心的感谢！

- [Breeze](https://github.com/deretame/Breeze)
- [jmcomic-next](https://github.com/HongShi2333/jmcomic-next)

同时感谢社区 [LinuxDO](https://linux.do) 的帮助。

## 免责声明

本项目仅供学习、研究和技术交流使用。项目作者与任何第三方服务、原始应用或内容提供方无关。
使用者应自行遵守当地法律法规以及相关服务条款。因使用本项目产生的任何法律、版权、账号、数据或财务风险均由使用者自行承担。

## 联系我们

[Telegram](https://t.me/miyabi_cn)

## License

遵循 [MIT](LICENSE) 协议。
