# Docker 环境搭建 + `ecnu-sa-labs` 镜像部署指导书

> **本文档的读者是在一台全新 arm64 云主机上执行操作的 AI Agent。**
> 目标:一次性完成 —— 安装 Docker、永久配置国内镜像加速、拉取 `ecnusa/ecnu-sa-labs` 镜像、并把它跑起来。
> 预期耗时 5~20 分钟,主要取决于镜像站带宽。
>
> 遇到与本文档描述不符的情况时,**不要臆测**,把实际输出记录下来并向用户报告。

---

## 0. 背景知识(阅读用,不需要执行)

### 目标镜像的已知事实

以下信息已通过多个独立的 registry 代理交叉验证过,是确定的事实,不要重复调研:

| 项目 | 值 |
|---|---|
| 镜像名 | `ecnusa/ecnu-sa-labs:latest` |
| 可用 tag | **只有 `latest`** |
| 架构 | **仅 `arm64/v8`**,没有 amd64 版本 |
| 大小 | **2383 MB,单层(single layer)** |
| 构建方式 | `docker import`(history 注释为 `Imported from -`),即整包文件系统 |
| config | **空 `{}`** —— 没有 `Cmd`、没有 `Entrypoint`、没有 `Env` |

**两个直接后果:**

1. **必须在 arm64 主机上运行。** 在 amd64 主机会 pull 成功但 run 失败(`exec format error`)。
2. **`docker run` 必须显式指定命令。** 直接 `docker run ecnusa/ecnu-sa-labs` 会报
   `no command specified`。正确写法是 `docker run -it --rm ecnusa/ecnu-sa-labs /bin/bash`。

### 网络环境约束

这台机器是**中国大陆的阿里云 ECS**:

- ❌ 无法直连 `download.docker.com`、`registry-1.docker.io`(Docker Hub)、`auth.docker.io`、Google
- ✅ 可以访问 `mirrors.aliyun.com`、`mirrors.cloud.aliyuncs.com`、`mirrors.tuna.tsinghua.edu.cn`

所以 **Docker 的安装源和镜像加速器都必须用国内地址**,这是硬性要求,不是优化项。

### 速度预期(重要)

已实测:阿里云 ECS 的**基础带宽约 17 MB/s**(从 `mirrors.aliyun.com` 实测 17799 KB/s)。

但**公共 Docker 镜像站的速度极不稳定**,实测在 **57 KB/s ~ 761 KB/s** 之间波动,可能比你的带宽慢 30~300 倍。

按 17 MB/s 算,2.4 GB 只需约 **2.5 分钟**;按 100 KB/s 算要 **6.6 小时**。

所以:如果第 5 步拉取速度低于 500 KB/s,请执行 **5.3 提速方案**,不要干等。

---

## 1. 环境前置检查

**先执行,确认所有前置条件满足再继续。**

```bash
uname -m                          # 必须是 aarch64;若是 x86_64 立即停止并报告用户
cat /etc/os-release | head -3     # 期望 Ubuntu 22.04 / 24.04
nproc && free -h                  # 建议 >= 2 核 / >= 4 GB 内存
df -h /                           # 需要 >= 10 GB 可用空间
whoami && id -u                   # 当前用户,非 root(如 root 则脚本中的 sudo 可去掉)
sudo -n true && echo "sudo 免密可用" || echo "sudo 需要密码"
```

**判定标准:**

- `uname -m` **必须是 `aarch64`**。如果是 `x86_64`,停止一切操作并告知用户:该镜像无法在这台机器上原生运行。
- 磁盘可用空间不足 10 GB 时停止并报告。

---

## 2. 安装 Docker

使用**阿里云 Docker CE 源**。注意 `arch=arm64` 这一处与 amd64 机器不同。

```bash
# 2.1 导入 GPG 密钥
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg \
  | sudo gpg --dearmor --yes -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 2.2 添加软件源(注意 arch=arm64)
. /etc/os-release   # 载入 $VERSION_CODENAME,Ubuntu 24.04 为 noble
echo "deb [arch=arm64 signed-by=/etc/apt/keyrings/docker.gpg] \
https://mirrors.aliyun.com/docker-ce/linux/ubuntu ${VERSION_CODENAME} stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list

# 2.3 安装
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y \
  docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**验证:**

```bash
docker --version
docker compose version
systemctl is-active docker        # 期望 active
systemctl is-enabled docker       # 期望 enabled(开机自启)
```

若 `docker --version` 报 command not found,或 `is-active` 不是 `active`,
执行 `sudo systemctl status docker --no-pager -l` 并把输出报告给用户。

---

## 3. 永久配置镜像加速器(核心步骤)

写入 `/etc/docker/daemon.json`。这是**永久生效**的配置,重启后依然有效。

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json >/dev/null <<'EOF'
{
  "registry-mirrors": [
    "https://docker.1panel.live",
    "https://docker.xuanyuan.me",
    "https://docker.m.daocloud.io",
    "https://docker.1ms.run"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "max-concurrent-downloads": 10,
  "live-restore": true
}
EOF
```

**配置项说明(供理解,不要删):**

- `registry-mirrors`:4 个镜像站,按顺序尝试,前一个失败自动切下一个。**这是拉取能成功的唯一依赖。**
- `log-opts`:容器日志轮转,单个文件最大 100 MB、最多保留 3 个,防止日志写满磁盘。
- `live-restore`:守护进程重启时容器不中断。

**应用配置并验证:**

```bash
python3 -c "import json;json.load(open('/etc/docker/daemon.json'));print('JSON 合法')"
sudo systemctl daemon-reload
sudo systemctl restart docker
sleep 3

# 关键验证:守护进程是否真的加载了加速器
sudo docker info 2>/dev/null | sed -n '/Registry Mirrors/,/^$/p'
```

期望输出 4 行镜像地址。**如果这里为空,说明 daemon.json 没生效,不要继续,先排查。**

---

## 4. 配置用户权限(免 sudo 使用 docker)

```bash
sudo usermod -aG docker "$USER"
id -nG | tr ' ' '\n' | grep -x docker && echo "已在组内"
```

**⚠️ 重要:Linux 的用户组在登录时确定,已存在的 shell 不会自动刷新。**

`usermod` 之后,当前这个 shell **仍然无法**直接使用 `docker`(会报
`permission denied while trying to connect to the docker API at unix:///var/run/docker.sock`)。

**Agent 请这样处理:** 在本次会话的后续所有命令中,统一用下面两种写法之一,不要依赖 `newgrp`:

```bash
sudo docker ...                  # 方式一:直接加 sudo
sg docker -c "docker ..."        # 方式二:临时以 docker 组执行
```

**若要彻底免 sudo(推荐给用户),把下面这段追加到 `~/.bashrc` 末尾。**
它的作用是:检测到"用户已在 docker 组、但当前会话未生效"时,自动带着 docker 组重新加载 shell。
这段逻辑已在生产环境验证过,对 SSH 终端和 VS Code Remote 终端都有效。

```bash
cat >> ~/.bashrc <<'EOF'

# >>> docker-group-autofix >>>
# 已加入 docker 组、但当前会话的用户组是"加入之前"的快照时,自动以 docker 组重新加载 shell。
# 触发场景:改组前就存在的 SSH 长连接;VS Code Remote 终端继承了早于改组的 server 进程的组。
if [ -z "${DOCKER_GROUP_FIXED:-}" ] \
   && ! id -nG 2>/dev/null | grep -qw docker \
   && getent group docker 2>/dev/null | grep -qw "$(id -un)" \
   && command -v sg >/dev/null 2>&1; then
    export DOCKER_GROUP_FIXED=1
    _docker_argv=()
    while IFS= read -r -d '' _docker_arg; do
        _docker_argv+=("$_docker_arg")
    done < "/proc/$$/cmdline" 2>/dev/null
    if [ "${#_docker_argv[@]}" -gt 0 ]; then
        case "${_docker_argv[0]}" in
            -*) exec sg docker -c "$(printf '%q ' "${SHELL:-/bin/bash}" -l)" ;;
            *)  exec sg docker -c "$(printf '%q ' "${_docker_argv[@]}")" ;;
        esac
    fi
    unset _docker_argv _docker_arg
fi
# <<< docker-group-autofix <<<
EOF

bash -n ~/.bashrc && echo "bashrc 语法合法"
```

---

## 5. 拉取镜像

### 5.1 先测速(不要盲目开始)

```bash
time sg docker -c "docker pull hello-world:latest"
```

这是 10 KB 的小镜像,用于验证**加速器链路是否通**。若这一步就失败,说明第 3 步没生效,先回到第 3 步排查。

### 5.2 拉取目标镜像

```bash
sg docker -c "docker pull ecnusa/ecnu-sa-labs"
```

**这是 2.4 GB 的单层镜像,单层意味着无法并行下载。**

**边拉边监控速度**(在另一个终端,或拉取放后台):

```bash
sudo sh -c 'cat /var/lib/docker/tmp/GetImageBlob* 2>/dev/null | wc -c' \
  | awk '{printf "已下载 %.0f MB / 2383 MB (%.0f%%)\n", $1/1048576, $1*100/2499800000}'
```

### 5.3 提速方案(速度 < 500 KB/s 时执行)

**方案 A:使用用户专属的阿里云加速器(最有效,推荐)**

请用户打开**阿里云控制台 → 容器镜像服务 → 镜像工具 → 镜像加速器**,拿到形如
`https://xxxxxxxx.mirror.aliyuncs.com` 的专属地址,然后:

```bash
sudo python3 - <<'PY'
import json
p = "/etc/docker/daemon.json"
d = json.load(open(p))
d["registry-mirrors"] = ["https://<这里替换成用户专属地址>.mirror.aliyuncs.com"] + d["registry-mirrors"]
json.dump(d, open(p, "w"), indent=2, ensure_ascii=False)
print("已更新:", d["registry-mirrors"])
PY
sudo systemctl restart docker
```

阿里云 ECS 访问自己的加速器通常走内网,速度可达几十 MB/s。

> **此方案需要用户提供专属地址。如果 Agent 拿不到,请明确向用户索要,不要跳过。**

**方案 B:逐一探测镜像站可用性**

镜像站会不时失效。探测方法(200/401 表示存活,000 表示不可达):

```bash
for m in docker.1panel.live docker.xuanyuan.me docker.m.daocloud.io \
         docker.1ms.run docker.1panel.top hub.rat.dev; do
  code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 12 \
    -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
    "https://$m/v2/library/hello-world/manifests/latest")
  echo "$code  $m"
done
```

把存活的地址按快慢排序,写进 `daemon.json` 的 `registry-mirrors`(靠前的优先),重启 docker。

**注意:不要用 `docker.m.daocloud.io` 拉取本镜像 —— 它是白名单制,已确认拒绝 `ecnusa/ecnu-sa-labs`。**

### 5.4 拉取完成验证

```bash
sg docker -c "docker images ecnusa/ecnu-sa-labs"
sg docker -c "docker inspect ecnusa/ecnu-sa-labs --format '{{.Architecture}}/{{.Os}}'"
```

期望输出 `arm64/linux`。

---

## 6. 运行镜像

### 6.1 第一步:先探明环境里有什么

因为该镜像 `config` 为空,**必须显式给出命令**。第一条命令用来摸清环境:

```bash
sg docker -c "docker run --rm ecnusa/ecnu-sa-labs sh -c 'uname -m; cat /etc/os-release | head -3; echo ---; ls /; echo ---; ls /bin | head -30'"
```

**判读:**

- `uname -m` 应输出 `aarch64`
- 看根目录和 `/bin` 里有什么,判断这是个什么环境(纯命令行工具集?还是带桌面?)

若报 `exec format error` → 架构不对,这台机器不是 arm64,停止并报告。
若报 `no command specified` → 命令没传对,检查是不是漏了 `sh -c '...'`。

### 6.2 进去交互式使用

```bash
# 基础进入
sg docker -c "docker run -it --rm ecnusa/ecnu-sa-labs /bin/bash"
```

若提示 `/bin/bash: not found`,改用:

```bash
sg docker -c "docker run -it --rm ecnusa/ecnu-sa-labs /bin/sh"
```

### 6.3 推荐使用方式(挂载工作目录,成果不丢)

```bash
mkdir -p ~/sa-labs

# 方式一:一次性进入,退出即销毁(隔离干净,适合做实验)
sg docker -c "docker run -it --rm -v $HOME/sa-labs:/work -w /work ecnusa/ecnu-sa-labs /bin/bash"

# 方式二:常驻容器,可反复进出(适合持续性实验)
sg docker -c "docker run -dit --name sa-labs -v $HOME/sa-labs:/work -w /work ecnusa/ecnu-sa-labs /bin/bash"
sg docker -c "docker exec -it sa-labs /bin/bash"    # 之后随时再进
sg docker -c "docker stop sa-labs && docker rm sa-labs"   # 用完清理
```

> `-v $HOME/sa-labs:/work` 把宿主机目录挂进容器,实验文件写在 `/work` 里就不会随容器销毁而丢失。

### 6.4 完成判据

**以下三条全部满足,才算任务完成:**

1. `docker info` 的 `Registry Mirrors` 列出加速器地址
2. `docker images ecnusa/ecnu-sa-labs` 能看到镜像
3. `docker run --rm ecnusa/ecnu-sa-labs sh -c 'uname -m'` 输出 `aarch64`,且交互式 `docker run -it --rm ... /bin/bash` 能进入 shell

---

## 7. 故障排查速查

| 现象 | 原因 | 处理 |
|---|---|---|
| `permission denied ... docker.sock` | 用户组未生效 | 用 `sudo docker` 或 `sg docker -c`,或追加第 4 步的 `.bashrc` 段后重开终端 |
| `docker: command not found` | 未安装成功 | 重跑第 2.3 步,看 apt 报错 |
| 拉取超时 / `i/o timeout` 到 `registry-1.docker.io` | 加速器没生效,直连了 Docker Hub | 检查 `daemon.json` 与 `docker info \| grep -A4 "Registry Mirrors"` |
| 拉取极慢(<500 KB/s) | 公共镜像站限速 | 执行 5.3 方案 A(阿里云专属加速器) |
| `no matching manifest` | 镜像站不支持该镜像 | 换其他镜像站;确认是 arm64 主机 |
| `exec format error` | 主机不是 arm64 | 换 arm64 主机,本镜像无 amd64 版本 |
| `no command specified` | 镜像 config 为空 | 必须显式给命令,如 `/bin/bash` |
| `/bin/bash: not found` | 镜像里没有 bash | 改用 `/bin/sh` |
| 磁盘写满 | 镜像 + 层占用 | `sudo docker system prune -af`,并确认 `daemon.json` 的 log-opts 已生效 |

---

## 附录:一键脚本

> 覆盖第 2~4 步(安装 + 永久配置 + 权限)。拉取和运行请按第 5、6 步单独执行,以便观察输出。
> **执行前请确认 `uname -m` 为 `aarch64`。**

```bash
#!/usr/bin/env bash
set -euo pipefail
[ "$(uname -m)" = "aarch64" ] || { echo "❌ 非 arm64 主机,已中止"; exit 1; }

echo "==> 1/4 添加阿里云 Docker 源"
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg \
  | sudo gpg --dearmor --yes -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
. /etc/os-release
echo "deb [arch=arm64 signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu ${VERSION_CODENAME} stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null

echo "==> 2/4 安装 Docker"
sudo apt-get update -qq
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y -qq \
  docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

echo "==> 3/4 写入镜像加速器"
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json >/dev/null <<'JSON'
{
  "registry-mirrors": [
    "https://docker.1panel.live",
    "https://docker.xuanyuan.me",
    "https://docker.m.daocloud.io",
    "https://docker.1ms.run"
  ],
  "log-driver": "json-file",
  "log-opts": { "max-size": "100m", "max-file": "3" },
  "max-concurrent-downloads": 10,
  "live-restore": true
}
JSON
sudo systemctl daemon-reload
sudo systemctl enable --now docker
sudo systemctl restart docker
sleep 3

echo "==> 4/4 用户权限"
sudo usermod -aG docker "$USER"

echo
echo "=== 结果验证 ==="
docker --version
sudo docker info 2>/dev/null | sed -n '/Registry Mirrors/,/^$/p'
echo
echo "✅ 完成。后续命令请用: sudo docker ...  或  sg docker -c \"docker ...\""
echo "   拉取镜像: sg docker -c \"docker pull ecnusa/ecnu-sa-labs\""
```

---

## 交付物清单

执行完毕后,请向用户报告:

- [ ] 主机架构确认(`uname -m` 的实际输出)
- [ ] Docker 版本
- [ ] `docker info` 中 `Registry Mirrors` 的实际内容
- [ ] 实际拉取耗时与平均速度
- [ ] 镜像架构确认(`docker inspect ... --format '{{.Architecture}}/{{.Os}}'`)
- [ ] 容器内 `uname -m` 输出
- [ ] 容器内环境探查结果(根目录、关键工具),以及建议用户从哪个命令/入口开始做实验
- [ ] 任何偏离本文档预期的情况及原始报错
