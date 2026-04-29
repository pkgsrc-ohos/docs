# pkgsrc-ohos

pkgsrc-ohos 项目是一个将 pkgsrc 移植到 OpenHarmony 平台的项目，为鸿蒙用户提供开箱即用的包管理器以及配套的软件仓库，支持鸿蒙 PC、鸿蒙开发板和鸿蒙容器。

> 仅支持 arm64 设备。

## 快速开始

### 鸿蒙 PC

下载 bootstrap kit（引导套件）后，将其加入到 PATH 中使用

```sh
curl -f -O https://cdn.pkgsrc-ohos.com/bootstrap/bootstrap-2025Q4-arm64.zip
unzip -uq bootstrap-2025Q4-arm64.zip -d /storage/Users/currentUser
export PATH=/storage/Users/currentUser/.pkg/bin:/storage/Users/currentUser/.pkg/sbin:$PATH

# 现在可以使用 pkgin 命令来安装软件包了（以 xz 为例）
pkgin update
pkgin install xz
```

### 鸿蒙开发板

下载 bootstrap kit（引导套件）后，将其加入到 PATH 中使用

```sh
# 此文件在这里下载：https://cdn.pkgsrc-ohos.com/bootstrap/bootstrap-2025Q4-arm64.tar.gz
hdc file send bootstrap-2025Q4-arm64.tar.gz /data
hdc shell

mount -o remount,rw /
mkdir -p /data/storage/Users/currentUser

# /storage 目录在 tmpfs 上，数据无法持久化，每次重启设备后需要重新创建软链接
ln -s /data/storage/Users /storage/Users

cd /data
tar -zxf bootstrap-2025Q4-arm64.tar.gz -C /
export PATH=/storage/Users/currentUser/.pkg/bin:/storage/Users/currentUser/.pkg/sbin:$PATH

# 现在可以使用 pkgin 命令来安装软件包了（以 xz 为例）
pkgin update
pkgin install xz
```

### 鸿蒙容器

下载 bootstrap kit（引导套件）后，将其加入到 PATH 中使用

```sh
curl -f -O https://cdn.pkgsrc-ohos.com/bootstrap/bootstrap-2025Q4-arm64.tar.gz
tar -zxf bootstrap-2025Q4-arm64.tar.gz -C /
export PATH=/storage/Users/currentUser/.pkg/bin:/storage/Users/currentUser/.pkg/sbin:$PATH

# 现在可以使用 pkgin 命令来安装软件包了（以 xz 为例）
pkgin update
pkgin install xz
```

## 常用操作

```sh
pkgin update             # 更新包索引
pkgin avail              # 查询可用的软件包
pkgin install [package]  # 安装软件包
```

## 贡献指南

用户的 pkgin 能下载到哪些预构建包，是由 `pkgsrc-ohos/ci` 仓库里面的 whitelist.txt 文件决定的。

pkgsrc 源码树里面有大量软件包，但只有白名单里面的软件包会被本项目的流水线构建。如果想要录入新的软件包，就把这个软件包加入到白名单列表。

具体操作步骤如下

### 1. 准备开发环境

准备好 Docker 环境，把 [ci-runner](https://github.com/pkgsrc-ohos/ci-runner) 容器运行起来，进入容器的命令行环境中。

```sh
docker pull ghcr.io/pkgsrc-ohos/ci-runner:latest
docker run -itd --name=ohos ghcr.io/pkgsrc-ohos/ci-runner:latest
docker exec -it ohos sh
```

> 这个容器里面不仅有编译器，还有 vim、git、openssh，你完全可以直接在里面改代码和提代码，无需在容器和宿主机之间反复传递文件。

### 2. 准备 bootstrap kit

按照上面的安装教程，在 ci-runner 容器里面配置好 bootstrap kit，确保 bmake 命令能正常执行。

### 3. 下载 pkgsrc 源码树

在容器内，下载 `pkgsrc-ohos/pkgsrc` 仓库，并切换到 pkgsrc-2025Q4 分支中。

```sh
cd /opt
git clone --depth 1 -b pkgsrc-2025Q4 https://github.com/pkgsrc-ohos/pkgsrc.git
```

### 4. 构建验证

在把软件包添加到白名单之前，需要先在本地对目标软件包进行构建验证，确保它能成功构建出来。

以 unzip 软件包为例，操作如下

```sh
cd /opt/pkgsrc/archivers/unzip
bmake install
```

如果构建成功，流程可以往下走；如果构建不成功，需要先自己进行鸿蒙适配，并把鸿蒙适配的内容提交到 `pkgsrc-ohos/pkgsrc` 仓库的 `pkgsrc-2025Q4` 分支。

### 5. 更新白名单

把 `pkgsrc-ohos/ci` 仓库的 whitelist.txt 拷贝到这个软件包的源码目录中，然后执行这几句命令，它会自动把这个软件包和级联依赖都更新到 whitelist.txt 文件中。

以 unzip 软件包为例，操作如下

```sh
# 进入源码目录
cd /opt/pkgsrc/archivers/unzip

# 下载原有的白名单
curl -fLO https://raw.githubusercontent.com/pkgsrc-ohos/ci/refs/heads/main/whitelist.txt

# 获取当前包的 PKGPATH 以及所有递归依赖的包
DEPS_LIST=$( { bmake show-var VARNAME=PKGPATH; bmake show-depends-recursive; } )

# 将新增内容与原有白名单内容合并，排序去重后写入临时文件
echo "$DEPS_LIST" | cat - whitelist.txt | sort -u > whitelist.txt.tmp

# 替换原有的白名单文件
mv whitelist.txt.tmp whitelist.txt
```

### 6. 提交白名单

提交 PR，把更新过的 whitelist.txt 文件提交到 `pkgsrc-ohos/ci` 仓库。

## 二次开发指南

### 1. 架构概述

本项目由四个仓库组成：

* **pkgsrc-ohos/pkgsrc**：Fork 自 pkgsrc 的源码树，包含针对 OpenHarmony 的适配补丁（pkgsrc-2025Q4 分支）
* **pkgsrc-ohos/ci**：自动化流水线脚本，包含两个 Shell 脚本和一个 Python 工具
  - `bootstrap.sh`：构建 bootstrap kit
  - `bulk-build.sh`：批量编译 whitelist.txt 中的软件包
  - `objctl.py`：OSS/CDN 交互工具，处理制品上传和缓存刷新
  - `whitelist.txt`：需要构建的软件包列表
* **pkgsrc-ohos/ci-runner**：CI 执行环境容器镜像，基于 DockerHarmony 二次封装
  - 集成 OpenHarmony SDK
  - 预置必要的命令行工具（coreutils、git、vim、ssh 等）
  - 预装阿里云 OSS/CDN Python SDK
  - 通过 GitHub Actions 自动构建和发布到 GHCR
* **pkgsrc-ohos/docs**：文档仓库

流水线工作流程：

1. **容器环境**（ci-runner）：GitHub Actions 基于 ci-runner 镜像启动 arm64 执行器
2. **Bootstrap 阶段**（bootstrap.sh）：创建 .pkg 目录、装入 pkgin 和 SSL 证书、进行代码签名、打包上传
3. **构建阶段**（bulk-build.sh）：下载 bootstrap kit、循环编译白名单中的每个软件包、自动增量检查（已构建则跳过）、更新包索引、上传制品
4. **分发阶段**（objctl.py）：所有制品存储在阿里云 OSS，通过 CDN 加速分发给最终用户

### 2. 基础设施准备

在配置流水线前，请确保以下资源已就绪：

* **域名资产**：准备一个域名用于制品分发。建议完成备案（非刚需）。
* **对象存储 (OSS)**：创建阿里云 OSS Bucket，记录其 Endpoint（外网访问节点）、Bucket Name 及 Region。
* **内容分发 (CDN)**：将域名绑定至 CDN 服务，回源地址配置为上述 OSS Bucket。若域名未备案，加速区域请选择"全球（不包含中国内地）"。
* **访问控制 (RAM)**：创建具有 OSS/CDN 操作权限的 AccessKey ID (AK) 与 AccessKey Secret (SK)。

### 3. 仓库分叉 (Fork)

将以下组件仓库 Fork 至你的个人空间或组织：

* `pkgsrc-ohos/pkgsrc`
* `pkgsrc-ohos/ci`
* `pkgsrc-ohos/ci-runner`（可选，除非需要自定义构建环境）

### 4. 配置流水线 Secrets

进入 Fork 后的 `ci` 仓库，在 **Settings** -> **Secrets and variables** -> **Actions** 中录入以下 **Repository secrets**：

| 密钥名称 | 功能说明 | 示例 |
| :--- | :--- | :--- |
| `ALIBABA_CLOUD_ACCESS_KEY_ID` | 阿里云 AccessKey ID | `LTAI...` |
| `ALIBABA_CLOUD_ACCESS_KEY_SECRET` | 阿里云 AccessKey Secret | `M79S...` |
| `OSS_ENDPOINT` | OSS 地域节点 (Endpoint) | `oss-cn-hongkong.aliyuncs.com` |
| `OSS_BUCKET` | OSS 桶名称 (Bucket) | `my-pkg-repo` |
| `OSS_REGION` | OSS 所在地域 (Region) | `cn-hongkong` |
| `CDN_DOMAIN` | CDN 加速域名 | `cdn.example.com` |

### 5. 运行流水线

进入 `ci` 仓库的 **Actions** 标签页，启用所有 Workflow 并手动触发：

1.  **执行 bootstrap**：构建 bootstrap kit。
2.  **执行 bulk-build**：待 bootstrap 任务完成后触发，开始批量构建软件制品。

### 6. 制品分发

参考本项目的安装指南，将你的 bootstrap kit 分发给用户使用即可。

在执行 `bootstrap` 工作流时，构建脚本会自动修改 `etc/pkgin/repositories.conf`，将默认软件源指向你的 `CDN_DOMAIN`。因此你只需向用户分发你的 bootstrap kit，用户使用时会自动从你的 CDN 拉取软件包。
