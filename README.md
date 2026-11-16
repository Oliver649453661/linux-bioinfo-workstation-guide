# Ubuntu / Linux 生信科研工作站标准化配置与平移部署指南

> **沉淀背景**：本指南提炼自 Ubuntu `24.04 LTS` 生信服务器的全套环境重构与规范化实战，涵盖 **四层标准环境架构、原生 R 与 Conda 环境 R 物理隔离、Miniforge3 `base` 净化冻结、`uv` 与 `aria2` 配置、Snakemake 9.x 流程迁移，以及保活 Tailscale/SSH/ToDesk 的 FlClash TUN 透明代理分流方案**。所有脚本与配置模板均内嵌于本文档中，方便一键平移部署到新服务器。

---

## 目录

1. [核心操作铁律（执行前必读）](#一核心操作铁律执行前必读)
2. [四层标准环境架构（4-Tier Architecture）](#二四层标准环境架构4-tier-architecture)
3. [五大核心模块标准部署 SOP](#三五大核心模块标准部署-sop)
   - [模块 1：系统底座、高速下载器与现代 Python 包管理](#模块-1系统底座高速下载器与现代-python-包管理)
   - [模块 2：原生 R 与 Conda 环境 R 彻底解耦（`%U` 原生隔离）](#模块-2原生-r-与-conda-环境-r-彻底解耦u-原生隔离)
   - [模块 3：Miniforge3 `.condarc` 规范化与 `base` 净化锁定](#模块-3miniforge3-condarc-规范化与-base-净化锁定)
   - [模块 4：FlClash TUN 透明分流（保活 Tailscale / SSH / ToDesk）](#模块-4flclash-tun-透明分流保活-tailscale--ssh--todesk)
   - [模块 5：Snakemake 9.x 独立环境与流程规范](#模块-5snakemake-9x-独立环境与流程规范)
4. [标准配置文件模板汇编](#四标准配置文件模板汇编)
5. [自动化脚本汇编（体检 / TUN 覆写 / 自动导入）](#五自动化脚本汇编体检--tun-覆写--自动导入)
6. [实战踩坑与排错手册（Troubleshooting & Pitfalls）](#六实战踩坑与排错手册troubleshooting--pitfalls)
7. [部署后一键验收清单（Verification Checklist）](#七部署后一键验收清单verification-checklist)

---

## 一、核心操作铁律（执行前必读）

1. **先只读体检，再动手改造**
   - 改造前先运行本文第 5.1 节的只读体检脚本盘点硬件、磁盘、服务与环境现状。
   - **严禁在大容量目录（如 `~/work`、`~/.config`）裸跑无过滤的 `grep -rn`**：在机械硬盘（HDD）上会刷空系统页缓存并卡死十余分钟；检索代码必须限定 `--include="*.R" --include="*.py" --include="*.sh" --include="*.yaml"`。
2. **改配置必留带时间戳备份**
   - 修改任何配置（`.bashrc`、`.condarc`、`.Rprofile`、`.Renviron`、`Snakefile`、`FlClash` 数据库）前，统一备份至 `~/archive/<name>.bak.YYYYMMDD`，保持家目录 `~/` 顶层整洁。
3. **优先回归软件原生机制，慎用软链或条件补丁**
   - 例如 R 库路径直接使用 Debian/Ubuntu 原生 `/etc/R/Renviron` 的 `%U` 展开机制（`~/R/x86_64-pc-linux-gnu-library/<maj.min>`），不靠 `.Rprofile` 条件判断或软链凑合。
4. **修改桌面端运行中配置（如 FlClash）必须先停进程**
   - 桌面端 GUI 会持有 `database.sqlite` / `shared_preferences.json` 的内存副本并在退出时回写覆盖磁盘；修改前必须先 `pkill -TERM -x FlClash` 并清除 `FlClash.lock`，修改完成后再重新拉起。
5. **敏感信息脱敏**
   - 绝不在脚本输出、Git 仓库或文档中明文写入机场订阅链接（URL）、密码、UUID 或 API Token。

---

## 二、四层标准环境架构（4-Tier Architecture）

生信工作站环境治理的核心原则是：**“底座极简、按用途分层、环境彻底隔离、配置可复现”**。

| 层级 | 定位与职责 | 标准落地方案 |
| --- | --- | --- |
| **Layer 1：系统原生层**（`apt` + `/opt`） | 稳固底座，直接满足 90%+ 日常工作 | • **基础与下载**：`openssh-server`、`tailscale`、`aria2`（`aria2c` 多线程断点下载）、`curl`、`wget`、`rsync`、`git`、`build-essential`<br>• **通用生信 CLI**：`fastqc`、`rna-star`（`STAR`）、`salmon`、`samtools`、`subread`（`featureCounts`）<br>• **系统 Python3**：仅通过 `apt` 安装 `python3-numpy`、`python3-pandas`、`python3-scipy`、`python3-matplotlib`、`python3-statsmodels`、`python3-yaml`；保留 PEP 668 `/usr/lib/python3.x/EXTERNALLY-MANAGED`，清理 `~/.local/lib/python3.x/site-packages/*.pth` 跨环境污染<br>• **唯一主力 R + RStudio**：系统 `R 4.6.x` + `RStudio Server / Desktop 2026.11+`（`:8787` 绑定 `/usr/bin/R`） |
| **Layer 2：包管理器底座**（`Miniforge3` + `uv`） | 纯净控制面，绝不混装业务包 | • **Miniforge3（`~/miniforge3`）**：`.condarc` 设为 `auto_activate: false` + 清华 `conda-forge`/`bioconda` 源 + `nodefaults`；`base` 环境清零 PyPI 包与业务软件，写入 `conda-meta/frozen` 冻结锁定（仅用 `conda self update` 维护本体）<br>• **uv（`~/.local/bin/uv`）**：接管纯 Python 虚拟环境（`uv venv` / `uv run`）与独立 Python CLI 工具（`uv tool`），配置清华 PyPI 镜像 |
| **Layer 3：专项隔离环境**（`miniforge3/envs/*`） | 按用途隔离高版本生信工具或老课题复现 | • **`snakemake` 环境**：独立 `python=3.12` + `snakemake>=9.27` + `multiqc`<br>• **历史项目环境（如 `pan_ana`）**：冻结保留旧课题所需的特定版本 R（如 `R 4.4.3` + Bioconductor `3.20`），与系统主力 R 物理隔离 |
| **Layer 4：规范化家目录**（`~/`） | 系统/软件/数据职责清晰 | • `~/work/<project>/`：科研项目目录（`CleanData/`、`reference/`、`scripts/`、`envs/`、`results/`、`Snakefile`、`config.yaml`）<br>• `~/apps/`：独立客户端与脚本（如 `~/apps/flclash/flclash_tun_override.js`）<br>• `~/archive/`：所有历史备份与归档文件 |

---

## 三、五大核心模块标准部署 SOP

### 模块 1：系统底座、高速下载器与现代 Python 包管理

```bash
# 1. 创建标准家目录分层
mkdir -p ~/work ~/apps ~/archive ~/.local/bin

# 2. 安装系统级下载器、R 编译依赖与基础科学计算包
sudo apt update
sudo apt install -y aria2 curl wget rsync git build-essential \
  libuv1-dev libxml2-dev libcurl4-openssl-dev libssl-dev \
  libfontconfig1-dev libharfbuzz-dev libfribidi-dev libfreetype6-dev \
  libpng-dev libtiff5-dev libjpeg-dev \
  python3-numpy python3-pandas python3-scipy python3-matplotlib python3-statsmodels python3-yaml

# 3. 排查并归档污染系统 Python 的 .pth 文件
find ~/.local/lib -maxdepth 3 -name "*.pth" -exec mv {} ~/archive/ \;

# 4. 安装 uv (现代极速 Python 包与虚拟环境管理器)
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 模块 2：原生 R 与 Conda 环境 R 彻底解耦（`%U` 原生隔离）

1. **隔离原理**
   - Debian/Ubuntu 的 `/etc/R/Renviron` 默认设置 `R_LIBS_USER=${R_LIBS_USER:-'%U'}`，自动按**平台 + 主次版本号**展开为 `~/R/x86_64-pc-linux-gnu-library/4.6`。
   - Conda R 的 `lib/R/etc/Renviron` 展开平台为 `x86_64-conda-linux-gnu-library/4.4`，且环境包安装在 `envs/<env>/lib/R/library`。
   - **只要不在 `~/.Renviron` 或 `~/.Rprofile` 中人为写死不带版本号的 `~/R/library`，两套 R 在 C 启动层天然物理隔离，且 `Rscript --vanilla` 也能原生识别全部用户包。**
2. **标准落地步骤**
   ```bash
   # 将原生 R 4.6 用户库归位至标准 %U 路径
   mkdir -p ~/R/x86_64-pc-linux-gnu-library
   # 若原有包在 ~/R/library 且确认 Built 为 R 4.6：
   mv ~/R/library ~/R/x86_64-pc-linux-gnu-library/4.6

   # 锁定 RStudio Server 使用系统原生 R
   echo "rsession-which-r=/usr/bin/R" | sudo tee /etc/rstudio/rserver.conf
   sudo systemctl restart rstudio-server
   ```

### 模块 3：Miniforge3 `.condarc` 规范化与 `base` 净化锁定

1. **清理重复 `.condarc` 并关闭自动激活**
   - 若存在 `~/miniforge3/.condarc`，将其移入 `~/archive/`，仅保留 `~/.condarc`（模板见第 4.1 节）。
2. **清理 `base` 混装包与残留 `pip` 元数据**
   - 卸载混装在 `base` 中的业务软件（如 `snakemake`、`aria2` 等）；
   - 清理 `~/miniforge3/lib/python3.12/site-packages/` 下非 `conda-meta` 管理的残留 `.dist-info` 目录；
   - 若 `conda doctor -n base` 提示 `Altered Files`，使用 `conda install -y -n base -c conda-forge --force-reinstall <pkg>` 修复。
3. **写入 `conda-meta/frozen` 锁定 `base`**
   - 直接写入 `~/miniforge3/conda-meta/frozen`（见第 4.2 节），避免运行 `conda doctor -n base base-protection --fix` 交互式克隆出冗余的 `envs/default` 环境。
   - 锁定后：普通 `conda install/remove -n base` 会直接报 `EnvironmentIsFrozenError` 拦截；升级 `conda` 本体时使用 `conda self update`。

### 模块 4：FlClash TUN 透明分流（保活 Tailscale / SSH / ToDesk）

在远程 Linux 服务器开启 TUN 模式必须建立 **三道防断连防线**：

1. **第一道防线（Linux 内核策略路由 `ip rule`）**
   ```bash
   # 确保局域网与 Tailscale CGNAT 段在进入 TUN 表 (pref 9000 table 2022) 之前优先查主表/Tailscale 表
   sudo ip rule del pref 5260 2>/dev/null || true
   sudo ip rule del pref 5261 2>/dev/null || true
   sudo ip rule add pref 5260 to 192.168.31.0/24 lookup main
   sudo ip rule add pref 5261 to 100.64.0.0/10 lookup 52
   ```
2. **第二道防线（TUN `route-exclude-address` + `strict-route: false` + `fake-ip-filter`）**
   - 通过覆写脚本 `flclash_tun_override.js` 让内核路由表 `2022` 自动挖除 `100.64.0.0/10`、`192.168.0.0/16`、`10.0.0.0/8`、`172.16.0.0/12`。
3. **第三道防线（进程与 IP 置顶 `DIRECT` + 策略组设为 `自动选择`）**
   - 将 `tailscaled`、`sshd`、`ToDesk`、`rserver`、`rsession`、`ascp` 置顶 `DIRECT`；
   - 将主策略组绑定为 `自动选择`（`url-test`），防止单个写死节点下线导致全网 `SSL_ERROR_SYSCALL` 超时。

### 模块 5：Snakemake 9.x 独立环境与流程规范

```bash
# 创建独立的 snakemake 9 + multiqc 环境
~/miniforge3/bin/mamba create -y -n snakemake python=3.12 snakemake multiqc pandas numpy pyyaml
```
- **MultiQC 规范**：在 `Snakefile` 中运行 `multiqc` 必须加 `-f`（`multiqc -f {params.outdir}/fastqc -o {params.outdir} -n multiqc_report.html`），且 `output` 数据目录名声明为 `directory("results/01_qc/multiqc_report_data")`，防止重跑生成 `_1`、`_2`、`_3` 副本或 `0` 字节文件。

---

## 四、标准配置文件模板汇编

### 4.1 `~/.condarc`

```yaml
channels:
  - conda-forge
  - bioconda
  - nodefaults
show_channel_urls: true
default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  bioconda: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
auto_activate: false
fetch_threads: 8
repodata_threads: 4
```

### 4.2 `~/miniforge3/conda-meta/frozen`

```json
{"message": "Your base environment is protected from direct modifications.\n\nTo manage conda and its plugins, use conda self:\n  conda self --help"}
```

### 4.3 `~/.config/uv/uv.toml` 与 `~/.config/pip/pip.conf`

```toml
# ~/.config/uv/uv.toml
[[index]]
url = "https://pypi.tuna.tsinghua.edu.cn/simple"
default = true
```

```ini
# ~/.config/pip/pip.conf
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
trusted-host = pypi.tuna.tsinghua.edu.cn
```

### 4.4 `~/.Renviron`、`~/.Rprofile` 与 `~/.R/Makevars`

```sh
# ~/.Renviron
# Rely on R native %U expansion (/etc/R/Renviron & conda lib/R/etc/Renviron):
# - System R 4.6.x -> ~/R/x86_64-pc-linux-gnu-library/4.6
# - Conda R 4.4.x  -> ~/R/x86_64-conda-linux-gnu-library/4.4
```

```r
# ~/.Rprofile
options(
  repos = c(CRAN = "https://mirrors.tuna.tsinghua.edu.cn/CRAN/"),
  BioC_mirror = "https://mirrors.tuna.tsinghua.edu.cn/bioconductor"
)
```

```makefile
# ~/.R/Makevars
CC = /usr/bin/gcc
CXX = /usr/bin/g++
CXX11 = /usr/bin/g++
CXX14 = /usr/bin/g++
CXX17 = /usr/bin/g++
```

### 4.5 `~/.bashrc` 末尾追加段（PATH 与按需命令行代理开关）

```bash
# User local binaries (uv, etc.)
export PATH="$HOME/.local/bin:$PATH"

# On-demand CLI proxy helpers (safe for Tailscale & LAN)
proxy_on() {
    local port="${1:-7890}"
    export http_proxy="http://127.0.0.1:${port}"
    export https_proxy="http://127.0.0.1:${port}"
    export all_proxy="socks5h://127.0.0.1:${port}"
    export HTTP_PROXY="$http_proxy"
    export HTTPS_PROXY="$https_proxy"
    export ALL_PROXY="$all_proxy"
    export no_proxy="localhost,127.0.0.1,::1,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,100.64.0.0/10,.local,.lan,.ts.net"
    export NO_PROXY="$no_proxy"
    echo "[proxy] ON -> 127.0.0.1:${port} (Tailscale & LAN bypassed)"
}

proxy_off() {
    unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY no_proxy NO_PROXY
    echo "[proxy] OFF"
}

proxy_status() {
    echo "http_proxy=${http_proxy:-<unset>}"
    echo "https_proxy=${https_proxy:-<unset>}"
    echo "no_proxy=${no_proxy:-<unset>}"
}
```

---

## 五、自动化脚本汇编（体检 / TUN 覆写 / 自动导入）

### 5.1 只读全景体检脚本（`audit_server.sh`）

```bash
#!/usr/bin/env bash
set -u

echo "=== 1. OS / CPU / RAM / DISK LAYOUT ==="
uname -a
lscpu | grep -E 'Model name|^CPU\(s\):' || true
free -h
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
df -h -x tmpfs -x devtmpfs -x squashfs

echo "=== 2. NETWORK / TAILSCALE / PROXY / TUN ==="
ip -br addr
ip rule
ss -tlnp | grep -E ':22|:7890|:8787' || true
ps -ef | grep -iE 'flclash|mihomo|tailscaled|rstudio' | grep -v grep || true

echo "=== 3. SYSTEM BIOINFO & DOWNLOAD TOOLS ==="
for cmd in aria2c curl wget rsync git fastqc STAR salmon samtools featureCounts uv R Rscript python3; do
  if command -v "$cmd" >/dev/null 2>&1; then
    printf "  [FOUND] %-15s -> %s\n" "$cmd" "$(command -v "$cmd")"
  else
    printf "  [MISS]  %-15s\n" "$cmd"
  fi
done

echo "=== 4. R & RSTUDIO ISOLATION CHECK ==="
if command -v R >/dev/null 2>&1; then
  /usr/bin/R -q -e 'cat("System R default:", R.version.string, "\n"); print(.libPaths()); print(Sys.getenv(c("R_LIBS_USER","R_LIBS_SITE")))'
  /usr/bin/R --vanilla -q -e 'cat("System R --vanilla:\n"); print(.libPaths()); cat("pkg count=", nrow(installed.packages()), "\n")'
fi
for f in "$HOME/.Renviron" "$HOME/.Rprofile" "/etc/rstudio/rserver.conf"; do
  echo "--- $f ---"
  [ -f "$f" ] && cat "$f" || echo "(not present)"
done

echo "=== 5. CONDA / MINIFORGE HEALTH & ENVS ==="
if [ -x "$HOME/miniforge3/bin/conda" ]; then
  "$HOME/miniforge3/bin/conda" env list
  "$HOME/miniforge3/bin/conda" config --show-sources
  "$HOME/miniforge3/bin/conda" doctor -n base || true
fi

echo "=== 6. PYTHON3 .pth POLLUTION & PEP 668 CHECK ==="
ls -la /usr/lib/python3*/EXTERNALLY-MANAGED 2>/dev/null || true
find "$HOME/.local/lib" -maxdepth 3 -name "*.pth" -ls 2>/dev/null || true
```

### 5.2 FlClash TUN 安全分流覆写脚本（`~/apps/flclash/flclash_tun_override.js`）

```javascript
// FlClash 脚本覆写 (Script Override) — Linux 生信工作站通用模板
// 目标：开启 TUN 透明代理兜底的同时，确保 Tailscale/SSH/ToDesk/RStudio/国内软件直连，
//       海外 AI 工具走代理，海外生信数据库可控分流。

const main = (config) => {
  const groups = Array.isArray(config["proxy-groups"]) ? config["proxy-groups"] : [];
  const defaultProxy = (groups[0] && groups[0].name) ? groups[0].name : "DIRECT";

  // 1) 强制走代理的海外 AI / CLI 进程名
  const TUN_PROXY_PROCESSES = [
    "antigravity",
    "language_server",
    "claude",
    "claude.exe",
    "codex"
  ];

  // 2) 海外生信数据库/代码托管策略：可设为 defaultProxy（走代理）或 "DIRECT"（直连）
  const BIO_DB_TARGET = defaultProxy;

  const BIO_DB_DOMAINS = [
    "ensembl.org",
    "ncbi.nlm.nih.gov",
    "flybase.org",
    "ebi.ac.uk",
    "ucsc.edu",
    "zenodo.org",
    "github.com",
    "githubusercontent.com"
  ];

  config.mode = "rule";
  config["find-process-mode"] = "always";
  config["allow-lan"] = false;

  config.tun = Object.assign({}, config.tun || {}, {
    enable: true,
    device: "FlClash",
    stack: "mixed",
    "auto-route": true,
    "auto-detect-interface": true,
    "strict-route": false,
    "dns-hijack": ["any:53"],
    "route-exclude-address": [
      "100.64.0.0/10",   // Tailscale CGNAT & MagicDNS (100.100.100.100)
      "192.168.0.0/16",  // 局域网段
      "10.0.0.0/8",      // 校园网/内网直连段
      "172.16.0.0/12"    // 容器与私网段
    ]
  });

  const extraFakeIpFilter = [
    "*.lan",
    "*.local",
    "+.tailscale.com",
    "+.tailscale.io",
    "+.ts.net",
    "+.todesk.com",
    "+.feishu.cn",
    "time.*.com",
    "ntp.*.com"
  ];
  const oldFilter = (config.dns && Array.isArray(config.dns["fake-ip-filter"]))
    ? config.dns["fake-ip-filter"]
    : [];
  const mergedFilter = [];
  oldFilter.concat(extraFakeIpFilter).forEach((item) => {
    if (mergedFilter.indexOf(item) === -1) mergedFilter.push(item);
  });
  config.dns = Object.assign({}, config.dns || {}, {
    enable: true,
    "enhanced-mode": "fake-ip",
    "fake-ip-filter": mergedFilter
  });

  const directRules = [
    "PROCESS-NAME,tailscaled,DIRECT",
    "PROCESS-NAME,sshd,DIRECT",
    "PROCESS-NAME,ToDesk,DIRECT",
    "PROCESS-NAME,ToDesk_Service,DIRECT",
    "PROCESS-NAME,ToDesk_Session,DIRECT",
    "PROCESS-NAME,rserver,DIRECT",
    "PROCESS-NAME,rsession,DIRECT",
    "PROCESS-NAME,qoder-cn,DIRECT",
    "PROCESS-NAME,ollama,DIRECT",
    "PROCESS-NAME,ascp,DIRECT",
    "DST-PORT,33001,DIRECT",
    "IP-CIDR,100.64.0.0/10,DIRECT,no-resolve",
    "IP-CIDR,192.168.0.0/16,DIRECT,no-resolve",
    "IP-CIDR,10.0.0.0/8,DIRECT,no-resolve",
    "IP-CIDR,172.16.0.0/12,DIRECT,no-resolve",
    "DOMAIN-SUFFIX,tailscale.com,DIRECT",
    "DOMAIN-SUFFIX,tailscale.io,DIRECT",
    "DOMAIN-SUFFIX,ts.net,DIRECT",
    "DOMAIN-SUFFIX,todesk.com,DIRECT",
    "DOMAIN-SUFFIX,feishu.cn,DIRECT",
    "DOMAIN-SUFFIX,larkoffice.com,DIRECT",
    "DOMAIN-SUFFIX,tuna.tsinghua.edu.cn,DIRECT",
    "DOMAIN-SUFFIX,ustc.edu.cn,DIRECT",
    "DOMAIN-SUFFIX,aliyun.com,DIRECT"
  ];

  const prependRules = directRules
    .concat(TUN_PROXY_PROCESSES.map((p) => "PROCESS-NAME," + p + "," + defaultProxy))
    .concat(BIO_DB_DOMAINS.map((d) => "DOMAIN-SUFFIX," + d + "," + BIO_DB_TARGET));

  const oldRules = Array.isArray(config.rules) ? config.rules : [];
  config.rules = prependRules.concat(oldRules);

  return config;
};
```

### 5.3 FlClash 自动导入覆写脚本并开启 TUN 工具（`apply_flclash_tun.py`）

> 使用方式：先执行 `pkill -TERM -x FlClash && sleep 2 && rm -f ~/.local/share/com.follow.clash/FlClash.lock`，再运行 `python3 apply_flclash_tun.py`，最后在桌面会话重启 `/usr/bin/FlClash`。

```python
#!/usr/bin/env python3
import json
import shutil
import sqlite3
import sys
import time
from datetime import datetime
from pathlib import Path


def main():
    home = Path.home()
    flclash_dir = home / ".local/share/com.follow.clash"
    override_src = home / "apps/flclash/flclash_tun_override.js"
    archive_dir = home / "archive" / f"flclash_config_bak_{datetime.now().strftime('%Y%m%d_%H%M%S')}"

    if not flclash_dir.exists() or not override_src.exists():
        print("[ERROR] FlClash config dir or override script not found.", file=sys.stderr)
        sys.exit(1)

    archive_dir.mkdir(parents=True, exist_ok=True)
    for fname in ("database.sqlite", "shared_preferences.json", "config.yaml"):
        fpath = flclash_dir / fname
        if fpath.exists():
            shutil.copy2(fpath, archive_dir / fname)

    scripts_dir = flclash_dir / "scripts"
    scripts_dir.mkdir(parents=True, exist_ok=True)
    shutil.copy2(override_src, scripts_dir / "1.js")

    conn = sqlite3.connect(str(flclash_dir / "database.sqlite"))
    cur = conn.cursor()
    now_ms = int(time.time() * 1000)
    cur.execute("DELETE FROM scripts WHERE id = 1")
    cur.execute(
        "INSERT INTO scripts (id, label, last_update_time) VALUES (?, ?, ?)",
        (1, "flclash_tun_override.js", now_ms),
    )
    for pid, label, group_name in cur.execute("SELECT id, label, current_group_name FROM profiles").fetchall():
        grp = group_name or label
        sel_map = json.dumps({grp: "自动选择"}, ensure_ascii=False)
        cur.execute(
            "UPDATE profiles SET overwrite_type = 'script', script_id = 1, selected_map = ? WHERE id = ?",
            (sel_map, pid),
        )
    conn.commit()
    conn.close()

    sp_path = flclash_dir / "shared_preferences.json"
    sp = json.loads(sp_path.read_text(encoding="utf-8"))
    fc = json.loads(sp["flutter.config"])

    fc["appSettingProps"]["autoRun"] = True
    fc["networkProps"]["systemProxy"] = False
    fc["vpnProps"]["enable"] = True
    fc["vpnProps"]["systemProxy"] = False
    fc["vpnProps"]["ipv6"] = False
    fc["vpnProps"]["allowBypass"] = True

    pcc = fc["patchClashConfig"]
    pcc["mixed-port"] = 7890
    pcc["mode"] = "rule"
    pcc["allow-lan"] = False
    pcc["find-process-mode"] = "always"
    pcc["tun"] = {
        "enable": True,
        "device": "FlClash",
        "auto-route": True,
        "stack": "mixed",
        "dns-hijack": ["any:53"],
        "route-address": [],
    }

    sp["flutter.config"] = json.dumps(fc, ensure_ascii=False, separators=(",", ":"))
    sp_path.write_text(json.dumps(sp, ensure_ascii=False, indent=2), encoding="utf-8")
    print("[OK] FlClash script override bound and TUN enabled.")


if __name__ == "__main__":
    main()
```

---

## 六、实战踩坑与排错手册（Troubleshooting & Pitfalls）

1. **R 升级到 `4.5 / 4.6` 后，网页版 RStudio Server 登录即闪退**
   - **现象**：运行 `/usr/lib/rstudio-server/bin/rsession` 报 `undefined symbol: Rf_countContexts`。
   - **根因**：R `4.5+` 移除了旧内部 C 符号 `Rf_countContexts`，`2024.09` 等旧版 RStudio Server 无法兼容。
   - **解法**：同步将 `rstudio-server` 和 `rstudio` 升级至 **`2026.04+`（推荐 `2026.11.0+203`）**。
2. **`~/.Renviron` / `~/.Rprofile` 写死 `~/R/library` 导致 Conda R 崩溃或 `--vanilla` 丢包**
   - **根因**：不带平台和版本号的目录会让原生 R（`4.6`）和 Conda R（`4.4`）混读同一套 `.so` 动态库（如 `Biobase.so`）；而在 `.Rprofile` 用 `if` 挂载又会在 `Rscript --vanilla` 时失效。
   - **解法**：将原生 R 用户包迁回标准 `%U` 目录 `~/R/x86_64-pc-linux-gnu-library/4.6`，清空 `~/.Renviron` 中的 `R_LIBS_USER` 与 `~/.Rprofile` 中的 `.libPaths()`。
3. **`~/.local/lib/python3.x/site-packages/*.pth` 污染系统 Python**
   - **根因**：`.pth` 文件把 `miniforge3` 的 `site-packages` 强行注入到了系统 `/usr/bin/python3`。
   - **解法**：移除该 `.pth` 文件，系统 Python 科学计算包统一用 `apt` 安装。
4. **`conda-meta/frozen` 的保护边界**
   - **现象**：写入 `conda-meta/frozen` 后，`conda install -n base` 会被拦截，但 `mamba install -n base` 仍能写入。
   - **原因**：`conda-meta/frozen` 是 Conda（CEP 22）自身的锁，`mamba 2.4.0` 尚未主动校验该文件，因此使用 `mamba` 时需明确加 `-n <env_name>`。
5. **FlClash 开启 TUN 后海外请求 `SSL_ERROR_SYSCALL` 超时**
   - **根因**：`database.sqlite` 的 `profiles.selected_map` 仍停留在多天前手动选中的单个失效节点。
   - **解法**：将 `selected_map` 设为 `{"<主策略组>":"自动选择"}`，交由 `url-test` 自动切换可用节点。

---

## 七、部署后一键验收清单（Verification Checklist）

```bash
# 1. Conda base 与 snakemake 环境健康度 (全部应为 ✅)
~/miniforge3/bin/conda doctor -n base
~/miniforge3/bin/conda doctor -n snakemake

# 2. 原生 R (--vanilla) 与 Conda R 路径物理隔离验证
/usr/bin/R --vanilla -q -e 'print(.libPaths()); cat("pkg count=", nrow(installed.packages()), "\n")'
~/miniforge3/envs/pan_ana/bin/R --vanilla -q -e 'print(.libPaths()); cat("pkg count=", nrow(installed.packages()), "\n")'

# 3. FlClash TUN 网卡、端口与路由表挖除检查
ip -br addr show FlClash && ss -tlnp | grep 7890 && ip route show table 2022 | head -n 12

# 4. 国内直连 vs 海外 TUN 透明代理 vs 生信数据库连通性
curl -sS -o /dev/null -w "Tsinghua=%{http_code}\n" --max-time 8 https://mirrors.tuna.tsinghua.edu.cn/
curl -sS -o /dev/null -w "Google204=%{http_code}\n" --max-time 8 https://www.gstatic.com/generate_204
curl -sS -o /dev/null -w "Ensembl=%{http_code}\n" --max-time 8 https://rest.ensembl.org/info/ping?content-type=application/json

# 5. Snakemake 9 DAG 干跑校验
conda run -n snakemake snakemake -n
```
