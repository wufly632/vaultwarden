# GitHub Actions 自动发布配置说明

## 概述

新增的 `.github/workflows/release-binaries.yml` 工作流会在创建新的 release 版本时自动构建并上传各个平台的二进制包。

## 触发条件

此工作流在以下情况下触发：

1. **创建 GitHub Release** - 推荐方式
2. **推送版本标签** - 如 `1.29.2`

## 支持的平台

### Linux
- ✅ `x86_64-unknown-linux-gnu` - 64位 Linux (glibc)
- ✅ `x86_64-unknown-linux-musl` - 64位 Linux (musl, 静态链接)
- ✅ `aarch64-unknown-linux-gnu` - ARM64 Linux (glibc)
- ✅ `aarch64-unknown-linux-musl` - ARM64 Linux (musl)
- ✅ `armv7-unknown-linux-gnueabihf` - ARMv7 Linux
- ✅ `arm-unknown-linux-gnueabihf` - ARMv6 Linux (Raspberry Pi)

### macOS
- ✅ `x86_64-apple-darwin` - Intel Mac
- ✅ `aarch64-apple-darwin` - Apple Silicon (M1/M2/M3)

### Windows
- ✅ `x86_64-pc-windows-msvc` - 64位 Windows

## 使用方法

### 方法一：通过 GitHub Web 界面创建 Release（推荐）

1. 在 GitHub 仓库页面，点击右侧的 "Releases"
2. 点击 "Draft a new release"
3. 填写以下信息：
   - **Choose a tag**: 创建新标签，格式如 `1.29.2`
   - **Release title**: 版本号，如 `v1.29.2`
   - **Description**: 更新日志和说明
4. 点击 "Publish release"
5. GitHub Actions 会自动开始构建所有平台的二进制包
6. 大约 1-2 小时后，所有平台的包会自动上传到 Release 页面

### 方法二：通过 Git 命令行创建标签

```bash
# 确保在 main 分支
git checkout main
git pull origin main

# 创建并推送标签
git tag 1.29.2
git push origin 1.29.2

# 然后在 GitHub 上创建 Release（选择刚才推送的标签）
```

### 方法三：使用 GitHub CLI

```bash
# 安装 gh CLI: https://cli.github.com/

# 创建 release
gh release create 1.29.2 \
  --title "v1.29.2" \
  --notes "Release notes here" \
  --draft  # 可选：先创建草稿

# GitHub Actions 会自动构建并上传二进制文件
```

## 构建产物

每个平台会生成以下文件：

- `vaultwarden-{platform}.tar.gz` (Linux/macOS) 或 `.zip` (Windows) - 压缩包
- `vaultwarden-{platform}.sha256` - SHA256 校验和

例如：
```
vaultwarden-linux-x86_64-gnu.tar.gz
vaultwarden-linux-x86_64-gnu.sha256
vaultwarden-macos-aarch64.tar.gz
vaultwarden-macos-aarch64.sha256
vaultwarden-windows-x86_64.exe.zip
vaultwarden-windows-x86_64.exe.sha256
```

## 校验下载的文件

下载后应该验证文件完整性：

```bash
# Linux/macOS
sha256sum -c vaultwarden-linux-x86_64-gnu.sha256

# macOS (如果没有 sha256sum)
shasum -a 256 -c vaultwarden-linux-x86_64-gnu.sha256

# Windows (PowerShell)
Get-FileHash vaultwarden-windows-x86_64.exe -Algorithm SHA256
```

## 监控构建进度

1. 访问仓库的 "Actions" 页面
2. 找到 "Build and Release Binaries" 工作流
3. 点击最新的运行记录查看详情
4. 可以查看每个平台的构建日志

## 自定义配置

### 修改支持的平台

编辑 `.github/workflows/release-binaries.yml` 中的 `matrix.include` 部分：

```yaml
matrix:
  include:
    - os: ubuntu-24.04
      target: x86_64-unknown-linux-gnu
      cross: false
      features: sqlite,mysql,postgresql
      artifact_name: vaultwarden-linux-x86_64-gnu
```

### 修改构建特性

通过 `features` 字段控制编译时包含的功能：

- `sqlite` - SQLite 支持
- `mysql` - MySQL/MariaDB 支持
- `postgresql` - PostgreSQL 支持
- `enable_mimalloc` - MiMalloc 内存分配器（推荐用于 musl 构建）
- `s3` - S3 存储支持

### 添加额外的构建步骤

在 workflow 中添加新的 step：

```yaml
- name: Custom step
  run: |
    # 你的自定义命令
```

## 与现有 release.yml 的关系

- **release.yml** - 构建和发布 Docker 镜像（多架构）
- **release-binaries.yml** (新) - 构建和发布独立二进制文件

两个 workflow 可以同时运行，互不干扰。

## 注意事项

### 1. 构建时间

- 完整的多平台构建大约需要 1-2 小时
- ARM 平台使用交叉编译，速度较慢
- Windows 和 macOS 构建在各自的原生 runner 上

### 2. GitHub Actions 配额

免费账户限制：
- 公开仓库：无限分钟
- 私有仓库：2000 分钟/月

如果构建失败超过配额，可以考虑：
- 减少构建的平台数量
- 使用 self-hosted runners

### 3. 发布权限

需要确保 GitHub Actions 有权限上传到 Releases：
- 仓库设置 → Actions → General → Workflow permissions
- 选择 "Read and write permissions"

### 4. 跨平台编译

使用 `cross` 工具进行交叉编译，它会自动处理：
- 工具链安装
- 依赖库配置
- 目标平台环境

## 故障排除

### 问题：构建失败，提示找不到依赖

**解决方案**：检查对应平台的依赖安装步骤，可能需要添加额外的系统包。

### 问题：Release 页面没有看到二进制文件

**解决方案**：
1. 检查 Actions 页面，确认工作流运行成功
2. 检查仓库的 Workflow permissions 设置
3. 确认使用的是正确的标签格式 (1.x.x)

### 问题：某个平台的二进制文件无法运行

**解决方案**：
1. 检查是否下载了正确的平台版本
2. 验证 SHA256 校验和
3. 确保目标系统满足最低要求（glibc 版本等）
4. 查看该平台的构建日志

### 问题：想要在不创建 Release 的情况下测试构建

**解决方案**：
```bash
# 手动触发 workflow（需要添加 workflow_dispatch 触发器）
gh workflow run release-binaries.yml
```

或者临时修改 workflow 触发条件：
```yaml
on:
  workflow_dispatch:  # 添加手动触发
  release:
    types: [created]
```

## 未来改进

可以考虑的增强功能：

1. **增加更多平台**：
   - FreeBSD
   - riscv64 (RISC-V)

2. **优化构建缓存**：
   - 使用 sccache 或 cargo-chef 加速编译

3. **添加签名**：
   - GPG 签名二进制文件
   - 代码签名（macOS/Windows）

4. **自动生成 changelog**：
   - 从 git commits 自动生成更新日志

5. **构建产物测试**：
   - 自动运行基本功能测试确保二进制可用

## 参考资料

- [GitHub Actions 文档](https://docs.github.com/en/actions)
- [cross 交叉编译工具](https://github.com/cross-rs/cross)
- [Rust 平台支持](https://doc.rust-lang.org/nightly/rustc/platform-support.html)
- [Vaultwarden Wiki](https://github.com/dani-garcia/vaultwarden/wiki)
