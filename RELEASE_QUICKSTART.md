# 快速开始指南

## 自动发布二进制包到 GitHub Release

### 1. 启用新的工作流

新增的 `.github/workflows/release-binaries.yml` 已经配置完成，无需额外设置。

### 2. 创建新版本

#### 方式 A：GitHub 网页界面（推荐）

1. 访问 https://github.com/你的用户名/vaultwarden/releases
2. 点击 "Draft a new release"
3. 填写信息：
   - Tag: `1.30.0` (版本号)
   - Title: `v1.30.0`
   - Description: 更新说明
4. 点击 "Publish release"

#### 方式 B：命令行

```bash
git tag 1.30.0
git push origin 1.30.0
```

然后在 GitHub 网页创建 Release 并选择这个标签。

### 3. 自动构建

发布 Release 后，GitHub Actions 会自动：

1. ✅ 构建 8 个平台的二进制文件：
   - Linux: x86_64 (glibc), x86_64 (musl), aarch64, armv7, armv6
   - macOS: x86_64, aarch64 (Apple Silicon)
   - Windows: x86_64

2. ✅ 生成 SHA256 校验文件

3. ✅ 自动上传到 Release 页面

4. ✅ 生成 Build Provenance 证明

### 4. 验证构建

1. 访问仓库的 "Actions" 标签
2. 查看 "Build and Release Binaries" 运行状态
3. 约 1-2 小时后，所有文件会出现在 Release 页面

### 5. 下载使用

用户可以从 Release 页面下载：

```bash
# 示例：下载 Linux x86_64 版本
wget https://github.com/你的用户名/vaultwarden/releases/download/1.30.0/vaultwarden-linux-x86_64-gnu.tar.gz

# 验证文件完整性
wget https://github.com/你的用户名/vaultwarden/releases/download/1.30.0/vaultwarden-linux-x86_64-gnu.sha256
sha256sum -c vaultwarden-linux-x86_64-gnu.sha256

# 解压
tar xzf vaultwarden-linux-x86_64-gnu.tar.gz
chmod +x vaultwarden-linux-x86_64-gnu

# 运行
./vaultwarden-linux-x86_64-gnu
```

## 配置选项

### 只构建部分平台（减少构建时间）

编辑 `.github/workflows/release-binaries.yml`，注释掉不需要的平台：

```yaml
strategy:
  matrix:
    include:
      # 保留需要的平台
      - os: ubuntu-24.04
        target: x86_64-unknown-linux-gnu
        features: sqlite,mysql,postgresql
        artifact_name: vaultwarden-linux-x86_64-gnu

      # 注释掉不需要的平台
      # - os: ubuntu-24.04
      #   target: armv7-unknown-linux-gnueabihf
      #   ...
```

### 仅在正式发布时触发（不在标签推送时触发）

修改 workflow 触发条件：

```yaml
on:
  release:
    types: [published]  # 只在发布时触发，不在 draft 时
```

### 添加自定义构建特性

修改对应平台的 `features` 字段：

```yaml
- os: ubuntu-24.04
  target: x86_64-unknown-linux-gnu
  features: sqlite,mysql,postgresql,s3  # 添加 s3 支持
  artifact_name: vaultwarden-linux-x86_64-gnu
```

## 常见问题

**Q: 构建失败怎么办？**
- 查看 Actions 页面的错误日志
- 检查是否有语法错误或依赖问题
- 可以先在本地测试构建：`cargo build --release --features sqlite --target x86_64-unknown-linux-gnu`

**Q: 想在发布前测试工作流？**
- 添加 `workflow_dispatch` 触发器手动运行
- 或者创建 Draft Release 测试

**Q: 文件没有上传到 Release？**
- 检查 Settings → Actions → General → Workflow permissions
- 确保设置为 "Read and write permissions"

**Q: 构建太慢？**
- 减少构建的平台数量
- 使用 self-hosted runners
- 启用构建缓存（需要额外配置）

## 与 Docker 镜像构建的关系

现在有两个并行的发布流程：

1. **release.yml** → 构建 Docker 镜像（ghcr.io, docker.io, quay.io）
2. **release-binaries.yml** → 构建独立二进制文件（GitHub Releases）

两者独立运行，可以同时使用。

完整的发布清单：
- ✅ Docker 镜像（多架构）
- ✅ 独立二进制文件（多平台）
- ✅ 源代码归档（GitHub 自动生成）

## 下一步

详细文档请查看 `RELEASE_AUTOMATION.md`。
