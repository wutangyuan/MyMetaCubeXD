# MyMetaCubeXD 修改说明

本仓库基于上游 [MetaCubeX/metacubexd](https://github.com/MetaCubeX/metacubexd) 修改，主要解决使用远程订阅时，网页“配置”页的内核选项会在订阅刷新后被订阅内容覆盖的问题。

## 问题原因

原逻辑会把网页配置直接写回当前活动配置文件。当活动配置来自远程订阅时，自动更新或手动刷新订阅会重新下载并覆盖该文件，因此 `allow-lan`、运行模式、端口、TUN 等用户设置可能恢复成订阅提供者的值。

## 修改内容

### 1. 远程订阅和用户设置分离

修改文件：`packages/agent/src/http.ts`

- 活动配置为远程订阅时，不再直接修改订阅源文件。
- 首次保存网页内核选项时，自动创建一个作用域绑定到当前订阅的合并配置：

  ```text
  MetaCubeXD persistent settings
  ```

- 后续网页修改写入该合并配置。
- 配置组合顺序为：远程订阅源 → 用户持久化设置。
- 自动更新或手动刷新订阅只替换订阅源，用户设置不会被覆盖。
- 本地配置仍按原逻辑直接保存，不额外创建合并配置。

### 2. 配置读取改为读取最终组合结果

修改文件：`packages/agent/src/http.ts`

`GET /api/control/config/section` 现在读取组合后的有效配置，而不是只读取订阅源。这样网页显示值与 Mihomo 实际运行值保持一致。

### 3. 覆盖配置页的内核选项

配置页中通过 `config/section` 保存的顶层 Mihomo 参数均使用同一持久化机制，包括：

- 允许局域网访问 `allow-lan`
- 运行模式 `mode`
- 统一延迟 `unified-delay`
- 出站接口 `interface-name`
- Mixed、HTTP、Socks、Redir、TProxy 端口
- TUN 配置
- DNS 配置
- 网络、规则等使用配置段保存的设置

右侧 XD 界面设置（主题、字体、背景、快捷键、智能推荐等）仍由浏览器的 Local Storage 或 IndexedDB 持久化，不受订阅刷新和容器重启影响。

### 4. 增加回归测试

修改文件：`packages/agent/src/http.test.ts`

- 验证配置段读取的是组合后的结果。
- 验证远程订阅的网页设置会写入作用域合并配置。
- 验证不会把相同设置直接写回远程订阅源。
- 保留本地配置、删除配置段和 `restart: false` 等原有行为测试。

### 5. Docker 入口脚本兼容 Windows 换行符

修改文件：`apps/server/Dockerfile`

镜像构建时先移除 `docker-entrypoint.sh` 可能存在的 CRLF 行尾，再赋予执行权限，避免出现：

```text
exec /docker-entrypoint.sh failed: No such file or directory
```

## 构建方式

在仓库根目录执行：

```bash
docker build \
  -t local/metacubexd-server:persistent-config \
  -f apps/server/Dockerfile \
  .
```

Docker Compose 中使用构建后的镜像，并将 `/data` 映射到持久化目录：

```yaml
services:
  metacubexd:
    image: local/metacubexd-server:persistent-config
    restart: unless-stopped
    volumes:
      - ./data:/data
```

密钥和令牌应通过 `.env` 或 NAS 的环境变量功能传入，不要提交到 Git 仓库。

## 验证结果

实际部署后完成了以下验证：

1. 从网页保存核心配置。
2. 强制重新下载并应用远程订阅。
3. 重启 Docker 容器。
4. 重新读取 Mihomo 运行配置。
5. 测试局域网 Mixed 代理端口。

结果：用户设置在订阅刷新及容器重启后保持不变，容器健康检查通过，代理连通性测试通过。

## 安全说明

仓库不包含以下部署私密信息：

- NAS SSH 用户名或密码
- 订阅地址
- Clash API Secret
- MetaCubeXD Control Token
- 本地 `.env` 文件
