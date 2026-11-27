# OCI Free VM Retry Script | OCI 免费 VM 抢机脚本

Automated retry scripts to claim Oracle Cloud Always Free VMs (ARM A1.Flex / x86 E2.1.Micro) via GitHub Actions — no need to keep your computer on 24/7.

🌐 [English](#english) | [简体中文](#简体中文)

---

<a name="english"></a>
# English

## Two Scripts

| Script | Instance Type | Spec | Architecture |
|--------|--------------|------|--------------|
| `oci_retry.py` | VM.Standard.A1.Flex | 2 OCPU / 12 GB RAM / 100 GB disk | ARM (Ampere) |
| `oci_retry_micro.py` | VM.Standard.E2.1.Micro | 1/8 OCPU / 1 GB RAM / 50 GB disk | x86 |

Both shapes are part of the Oracle Always Free tier. Always Free instances and volumes must be created in your tenancy's **home region**. The free allowance is:

- **Ampere A1:** 2 OCPUs + 12 GB RAM in total per tenancy
- **Micro:** up to 2 instances (1/8 OCPU each, can burst higher)
- **Storage:** 200 GB of boot + block volumes in total

> 📌 **Always Free update (June 15, 2026):** Oracle's official Always Free page lowered the Ampere A1 allowance from 4 OCPUs + 24 GB RAM to **2 OCPUs + 12 GB RAM** per tenancy.  
> Official source: https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm

---

## Two Ways to Run

| Method | Best For | Pros | Cons |
|--------|----------|------|------|
| GitHub Actions (recommended) | Long-term automated retrying | 24/7, no computer resources needed | Requires a **public** repo + Secrets setup |
| Run locally | Quick testing | No GitHub setup needed | Your computer must stay on |

> 💡 **Why public repo?** Public repositories get unlimited GitHub Actions minutes for free. Private repos have a monthly limit.

---

## Before You Start (Both Methods)

### Get an OCI API Key

1. OCI Console → top-right **Profile** menu → **User settings**
2. Left menu → **My profile** → top menu → **Tokens and keys** → **Add API key**
3. Choose **Generate API key pair** → download the private key (`.pem` file)
4. Click **Add** → a configuration snippet appears with your `user`, `fingerprint`, `tenancy` and `region`. Copy it and keep it with the `.pem` file; the next steps use these values

### Find Your Home Region

Always Free resources can only be created in your tenancy's **home region**. To find it, click the region name at the top right of the Console; the menu shows your **Home region** (e.g. `Singapore (Singapore)`). Its Region Identifier (e.g. `ap-singapore-1`, see Oracle's [region list](https://docs.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm)) is the `region` value to use.

### Get SSH Public Key

**Purpose**: Used to SSH into the instance after it's created.

1. When creating an instance in Oracle Cloud, choose **Generate key pair** (Generate SSH key pair)
2. Download two files:
   - `*.key` or `*.pem` (private key — keep this safe, used for SSH login)
   - `*.pub` (public key — GitHub Actions: add its contents as the `OCI_SSH_PUBLIC_KEY` GitHub Secret, see Step 2: Set GitHub Secrets; local runs: set `ssh_public_key_file` in your OCI config to this file)
3. Open the `.pub` file in a text editor, copy all content (starts with `ssh-rsa AAAA...`)

Or generate your own key locally with `ssh-keygen`.

### Script Configuration

The settings are at the top of each script. Edit them after you get the repository (GitHub Actions: Step 3; Run Locally: step 3):

#### ARM script: `oci_retry.py`

```python
INSTANCE_NAME = "arm-server"
UBUNTU_VERSION = "24.04"  # OS for the new instance: "24.04" (default), "26.04" or "22.04"
ARM_OCPUS = 2
ARM_MEMORY_IN_GBS = 12
BOOT_VOLUME_SIZE_IN_GBS = 100
RETRY_INTERVAL = 90  # Retry interval in seconds
```

> `BOOT_VOLUME_SIZE_IN_GBS` sets the boot volume size. Always Free includes `200 GB` of boot + block volume storage in total; the ARM default of `100 GB` leaves room for two Micro instances (`50 GB` each). Lower `BOOT_VOLUME_SIZE_IN_GBS` if you also use other volumes.

#### Micro script: `oci_retry_micro.py`

```python
INSTANCE_NAME  = "micro-server"   # Change to "micro-server-2" when claiming your second Micro instance
UBUNTU_VERSION = "24.04"  # OS for the new instance: "24.04" (default), "26.04" or "22.04"
BOOT_VOLUME_SIZE_IN_GBS = 50
RETRY_INTERVAL = 90  # Retry interval in seconds
```

> Each OCI account can have up to **2 free Micro instances**. To claim a second one, change `INSTANCE_NAME` to a different value (e.g. `"micro-server-2"`) so both instances can be identified separately in the OCI Console. The VCN and subnet are shared between instances automatically.

> ⚠️ **Regions with multiple availability domains (ADs):** Oracle's [Always Free page](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm) states that Micro instances "can only be created in one availability domain", but does not say which one. The scripts always use the **first** AD returned by the API (`ads[0]` in `get_availability_domain()`). If your region has several ADs and the Micro script never succeeds, check your Micro limit per AD in the OCI Console (search for **Limits, Quotas and Usage**), then change `ads[0]` to the index of that AD. Regions with a single AD (e.g. `ap-singapore-1`) are not affected.

#### Ubuntu version (both scripts)

`UBUNTU_VERSION` sets which Ubuntu LTS is installed on the **new** instance. The script always picks the newest Oracle image build of that version. Changing it only affects instances created afterwards; instances you already have keep their OS.

| Version | Standard security updates until | Default Python |
|---------|-------------------------------|----------------|
| `"26.04"` | May 2031 | 3.14 |
| `"24.04"` (default) | May 2029 | 3.12 |
| `"22.04"` | May 2027 | 3.10 |

Sources: [Ubuntu release cycle](https://ubuntu.com/about/release-cycle), [python3-defaults](https://launchpad.net/ubuntu/+source/python3-defaults).

---

## GitHub Actions — Automated 24/7 Retry (Recommended)

No need to keep your computer running. GitHub Actions retries continuously, 24 hours a day.

### How It Works

```
Job starts
  └─ Attempts every 90 seconds for ~330 minutes (~220 tries)
       ├─ Instance claimed → workflow auto-disables, done
       ├─ Script error (e.g. wrong secrets) → chain stops with an error message
       └─ Timeout → immediately triggers next Job (virtually no gap)
                        └─ Repeats...

If the chain breaks: manually go to the Actions tab and click Run workflow again
```

| Item | Details |
|------|---------|
| Duration per Job | ~330 minutes (~220 attempts) |
| Relay method | Auto-triggers next Job when the time limit is reached, virtually no gap |
| Max concurrency | 1 running + 1 queued (no pile-up) |
| On script error | Chain stops and shows an error; fix the problem, then click Run workflow again |
| If chain breaks | Manually trigger from Actions tab |
| After success | Workflow auto-disables; if that fails, the run shows an "Auto-disable failed" error and you disable it manually |
| ARM vs Micro | Separate workflows — **running only one at a time is recommended** |

> ⚠️ Both workflows use the same OCI account. Running ARM and Micro at the same time with the default `RETRY_INTERVAL` (90 seconds) causes OCI to return `TooManyRequests` errors. We recommend not starting both workflows at the same time.

### Setup Steps

#### Step 1: Get This Repository

Click **Fork** at the top of this repository's page to copy it to your GitHub account. Keep it **public** (public repos get unlimited Actions minutes).

If you prefer to create your own **public** repository instead, create it empty (no README), then copy this code into it:

```bash
git clone https://github.com/WendyChen330/oci-arm-retry.git
cd oci-arm-retry
git remote set-url origin https://github.com/<your-username>/<your-repo>.git
git push -u origin main
```

#### Step 2: Set GitHub Secrets

Go to your repository → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

You need **6 secrets** in total:

| Secret Name | Value | Where to Get It |
|-------------|-------|----------------|
| `OCI_USER` | `ocid1.user.oc1..xxxxxx` | `user` in the configuration snippet from **Get an OCI API Key** above |
| `OCI_FINGERPRINT` | `xx:xx:xx:xx:xx` | `fingerprint` in the configuration snippet from **Get an OCI API Key** above |
| `OCI_TENANCY` | `ocid1.tenancy.oc1..xxxxxx` | OCI Console → top-right **Profile** menu → **Tenancy: &lt;name&gt;** → OCID |
| `OCI_REGION` | e.g. `ap-singapore-1` | Your home region's Region Identifier, see **Find Your Home Region** above. Always Free resources can only be created there |
| `OCI_PRIVATE_KEY` | Full contents of `.pem` file | Open the `.pem` from **Get an OCI API Key** above, copy everything (including `-----BEGIN/END PRIVATE KEY-----`) |
| `OCI_SSH_PUBLIC_KEY` | Full contents of your `.pub` file | See **Get SSH Public Key** above (starts with `ssh-`) |

> If an Actions run reports `Failed to verify the HTTP(S) Signature`, confirm `OCI_PRIVATE_KEY` and `OCI_FINGERPRINT` come from the same OCI API key, and that the key is registered for the `OCI_USER` in OCI. The ARM workflow checks that the key and fingerprint match before making API calls.

> No GitHub token is needed: relaying to the next Job and auto-disabling use the workflow's built-in `GITHUB_TOKEN`, which GitHub creates for each Job automatically.

#### Step 3: Adjust Script Settings (Optional)

To change the instance name, Ubuntu version or other settings, edit the variables at the top of the script in your repository and commit the change. See **Script Configuration** above.

#### Step 4: Start the Workflow

1. Go to your repository → **Actions** tab. On a new fork, first click the green **I understand my workflows, go ahead and enable them** button
2. In the left sidebar, select **OCI ARM Retry** or **OCI Micro Retry**, click **Run workflow** on the right, then click the green **Run workflow** button in the menu

ARM and Micro have separate workflows. Running only one at a time is recommended (see the note under How It Works).

---

## Run Locally

### 1. Download This Repository

Requires Python 3 with pip. Download this repository (**Code** → **Download ZIP**, or `git clone`) and open a terminal in its folder.

### 2. Create the OCI Config File

**Location**: `~/.oci/config` (on Windows: `C:\Users\<username>\.oci\config`, no file extension)

On Windows, you can create the folder and an empty file in PowerShell, then open it in Notepad:

```powershell
New-Item -ItemType Directory -Force "$HOME\.oci" | Out-Null
if (-not (Test-Path "$HOME\.oci\config")) { New-Item -ItemType File "$HOME\.oci\config" | Out-Null }
notepad "$HOME\.oci\config"
```

```
[DEFAULT]
user=ocid1.user.oc1..xxxxxx
fingerprint=xx:xx:xx:xx:xx
tenancy=ocid1.tenancy.oc1..xxxxxx
region=ap-singapore-1
key_file=C:\Users\<username>\.oci\oci_api_key.pem
ssh_public_key_file=C:\Users\<username>\.oci\ssh-key.pub
```

1. Paste the configuration snippet from **Get an OCI API Key**
2. Update `key_file` to the actual path of your `.pem` file
3. Make sure `region` is your home region (see **Find Your Home Region**)
4. Add `ssh_public_key_file` with the path to your SSH public key (`.pub`, see **Get SSH Public Key**). The script installs this key on the new instance

> ⚠️ Use file paths with English letters and numbers only. On Windows, a path containing Chinese or other non-English characters can make the config file fail to load.

### 3. Adjust Script Settings (Optional)

Instance name, Ubuntu version and the other settings are at the top of each script. See **Script Configuration** above.

### 4. Install and Run

In that terminal, run:

```bash
# Install dependencies (once only)
pip install oci

# Run ARM version
python oci_retry.py

# Run Micro version
python oci_retry_micro.py
```

---

## Script Behavior

| Status | Description |
|--------|-------------|
| Initializing | Auto-creates VCN + subnet (skips if already exists) |
| Retry loop | Attempts every `RETRY_INTERVAL` seconds |
| Out of capacity | Prints ❌ and keeps retrying |
| Network timeout | Prints ⚠️ and keeps retrying (won't crash) |
| Other errors when creating the instance (e.g. limit reached) | Prints ❌ with the error message and keeps retrying — if it never succeeds, check the log |
| Setup errors (e.g. wrong OCID or API key) | Stops with an error before the retry loop starts |
| Success | Prints ✅ instance ID and stops automatically |

> **Open ports:** When the script creates a new VCN, it opens inbound TCP **22 (SSH), 80 (HTTP) and 443 (HTTPS)** from anywhere (`0.0.0.0/0`). An existing VCN is reused as-is. To open other ports, add them to the port list in `create_vcn_and_subnet()` before the first run, or add ingress rules to the VCN's security list in the OCI Console later. On OCI Ubuntu images, the port must also be allowed in the instance's `iptables` — see the "Update firewall settings" step in Oracle's [Apache on Ubuntu tutorial](https://docs.oracle.com/en-us/iaas/developer-tutorials/tutorials/apache-on-ubuntu/01oci-ubuntu-apache-summary.htm).

---

## After Success

Once an instance is claimed, the workflow **auto-disables**. If the run shows an **Auto-disable failed** error instead, disable it manually: Actions tab → the workflow → **...** → **Disable workflow**.

Go to OCI Console → **Compute** → **Instances** → find your instance's public IP, then SSH in:

```bash
ssh -i ssh-key.key ubuntu@your-public-ip
```

Use the SSH private key from **Get SSH Public Key** (not the OCI API key `.pem`).

---

## Notes

- Sensitive values (OCI API private key) are stored in GitHub Secrets and never committed
- **Change on November 7, 2026:** the workflows no longer use a `GH_PAT` secret. If you forked or copied this repo before that date, replace your `.github/workflows/oci_retry.yml` and `.github/workflows/oci_retry_micro.yml` with the current versions from this repo (the scripts don't need to change). After that, you can delete the `GH_PAT` secret and revoke the token in GitHub → Settings → Developer settings
- **Change on November 8, 2026:** `COMPARTMENT_ID` and `SSH_PUBLIC_KEY` are no longer settings in the scripts. Instances are now always created in your tenancy (root compartment), and the SSH public key is read from the `OCI_SSH_PUBLIC_KEY` GitHub Secret (GitHub Actions) or from `ssh_public_key_file` in your OCI config (local runs). If you update your copy of the scripts to this version: for GitHub Actions, also replace both `.yml` files in `.github/workflows/` and add that secret; for local runs, add that config entry
- `.pem` / `.key` / `.pub` files are in `.gitignore` and won't be tracked

---
---

<a name="简体中文"></a>
# 简体中文

## 两种脚本

| 脚本 | 目标机型 | 规格                             | 架构 |
|------|---------|--------------------------------|------|
| `oci_retry.py` | VM.Standard.A1.Flex | 2 OCPU / 12 GB RAM / 100 GB 存储 | ARM (Ampere) |
| `oci_retry_micro.py` | VM.Standard.E2.1.Micro | 1/8 OCPU / 1 GB RAM / 50 GB 存储 | x86 |

两种机型都属于 Oracle Always Free 永久免费方案。Always Free 的机器和存储都必须建立在租用户（tenancy）的**主区域（home region）**。免费额度如下：

- **Ampere A1：** 每个租用户合计 2 OCPU + 12 GB RAM
- **Micro：** 最多 2 台（每台 1/8 OCPU，可短暂使用更多 CPU）
- **储存空间：** boot volume + block volume 合计 200 GB

> 📌 **官方更新（2026-06-15 起）：** Oracle 官方 Always Free 页面将 Ampere A1 额度从 4 OCPU + 24 GB RAM 调降为每个租用户 **2 OCPU + 12 GB RAM**。  
> 官方来源：https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm

---

## 两种使用方式

| 方式 | 适合对象 | 优点 | 缺点 |
|------|---------|------|------|
| GitHub Actions（推荐）| 长期自动抢机 | 24 小时不间断，不占电脑资源 | 需要 **public repo** + 设置 Secrets |
| 本机执行 | 想快速试试 | 不需要设置 GitHub | 电脑要一直开着 |

> 💡 **为什么需要 public repo？** Public repo 可以免费使用无限制的 GitHub Actions 分钟数。Private repo 每月有上限。

---

## 开始前的准备（两种方式共用）

### 取得 OCI API 密钥

1. OCI 主控台 → 右上角 **Profile** 选项 → **用户设置**（User settings）
2. 左侧选项 → **我的概要信息**（My profile）→ 上方选项 → **令牌和密钥**（Tokens and keys）→ **新增 API 密钥**（Add API key）
3. 选 **生成 API 密钥对**（Generate API key pair）→ 下载私钥（`.pem` 文件）
4. 点 **新增**（Add）→ 画面会出现一段设置文字，包含 `user`、`fingerprint`、`tenancy`、`region`。把它复制下来，跟 `.pem` 文件一起保存，后面的步骤会用到

### 查询主区域

Always Free 资源只能建立在租用户的**主区域**。查询方式：点主控台右上角的区域名称，选项会标出 **Home region**（例如 `Singapore (Singapore)`）。它的区域识别码（Region Identifier，例如 `ap-singapore-1`，可对照 Oracle 的[区域列表](https://docs.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm)）就是 `region` 要填的值。

### 取得 SSH 公钥

**用途**：建立 instance 后用来 SSH 连接进入 server

1. 在 Oracle Cloud 建立 instance 时，选「**生成密钥对**」（Generate key pair）
2. 下载两个文件：
   - `*.key` 或 `*.pem`（私钥，自己保管，连接时用）
   - `*.pub`（公钥。GitHub Actions：内容要设置到 GitHub Secrets 的 `OCI_SSH_PUBLIC_KEY`，设置方式见「Step 2：设置 GitHub Secrets」；本机执行：在 OCI 设置文件的 `ssh_public_key_file` 填入这个文件的路径）
3. 用记事本开启 `.pub` 文件，复制全部内容（格式：`ssh-rsa AAAA...` 开头）

或在本机执行 `ssh-keygen` 自己生成。

### 脚本设置

设置都在每支脚本的最上方，取得 repo 之后再修改即可（GitHub Actions 见 Step 3；本机执行见步骤 3）：

#### ARM 脚本：`oci_retry.py`

```python
INSTANCE_NAME = "arm-server"
UBUNTU_VERSION = "24.04"  # 新机器安装的作业系统："24.04"（预设）、"26.04" 或 "22.04"
ARM_OCPUS = 2
ARM_MEMORY_IN_GBS = 12
BOOT_VOLUME_SIZE_IN_GBS = 100
RETRY_INTERVAL = 90  # 重试间隔（秒）
```

> `BOOT_VOLUME_SIZE_IN_GBS` 是开机存储（boot volume）的大小。Always Free 的 boot volume + block volume 储存空间合计 `200 GB`；ARM 预设 `100 GB`，剩下的空间刚好够两台 Micro（各 `50 GB`）。如果你还有其他存储，请把 `BOOT_VOLUME_SIZE_IN_GBS` 调小。

#### Micro 脚本：`oci_retry_micro.py`

```python
INSTANCE_NAME  = "micro-server"   # 抢第二台时改成 "micro-server-2"，方便在 OCI 主控台区分
UBUNTU_VERSION = "24.04"  # 新机器安装的作业系统："24.04"（预设）、"26.04" 或 "22.04"
BOOT_VOLUME_SIZE_IN_GBS = 50
RETRY_INTERVAL = 90  # 重试间隔（秒）
```

> 每个 OCI 帐号最多可以有 **2 台免费 Micro**。抢第二台时，将 `INSTANCE_NAME` 改成不同的名称（例如 `"micro-server-2"`），方便在 OCI 主控台区分。VCN 和子网路会自动共用，不需要另外建立。

> ⚠️ **有多个可用性网域（AD）的区域：** Oracle 官方 [Always Free 页面](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)写明 Micro 只能在其中一个 AD 建立，但没有说是哪一个。脚本一律使用 API 回传的**第一个** AD（`get_availability_domain()` 里的 `ads[0]`）。如果你的区域有多个 AD，而 Micro 脚本一直抢不到，请到 OCI 主控台搜寻 **Limits, Quotas and Usage**，查看 Micro 的额度分配在哪个 AD，再把 `ads[0]` 改成那个 AD 的顺序编号。只有一个 AD 的区域（例如 `ap-singapore-1`）不受影响。

#### Ubuntu 版本（两支脚本通用）

`UBUNTU_VERSION` 决定**新建立的机器**要安装哪一版 Ubuntu LTS，脚本会自动挑该版本在 Oracle 上最新的映像文件。修改这个值只影响之后新建的机器，已经建立的机器会维持原本的作业系统。

| 版本 | 标准安全更新到 | 内建 Python |
|------|--------------|------------|
| `"26.04"` | 2031 年 5 月 | 3.14 |
| `"24.04"`（预设） | 2029 年 5 月 | 3.12 |
| `"22.04"` | 2027 年 5 月 | 3.10 |

资料来源：[Ubuntu 版本周期](https://ubuntu.com/about/release-cycle)、[python3-defaults](https://launchpad.net/ubuntu/+source/python3-defaults)。

---

## GitHub Actions 自动抢机（推荐）

不需要电脑一直开着，24 小时不间断自动重试。

### 运作方式

```
Job 开始
  └─ 每 90 秒尝试一次，持续约 330 分钟（~220 次）
       ├─ 抢到了 → 自动停用 workflow，结束
       ├─ 脚本出错（例如 secrets 填错）→ 链条停止，并显示错误讯息
       └─ 超时 → 立刻触发下一个 Job，空窗几乎为零
                        └─ 重复循环...

链条中断时：手动到 Actions 页签再按一次 Run workflow
```

| 项目 | 说明 |
|------|------|
| 每次 Job 持续时间 | 约 330 分钟（约 220 次尝试）|
| 接力方式 | 时间到时自动触发下一个，空窗几乎为零 |
| 同时执行上限 | 最多 1 个跑 + 1 个排队（不会累积）|
| 脚本出错时 | 链条停止并显示错误讯息；修正问题后再按一次 Run workflow |
| 链条中断时 | 手动到 Actions 页签再按一次 Run workflow |
| 抢到后 | workflow 自动停用；如果停用失败，执行结果会显示「Auto-disable failed」错误，请手动停用 |
| ARM 与 Micro | 各自独立的 workflow，**建议一次只跑一个** |

> ⚠️ 两个 workflow 使用同一个 OCI 帐号。用预设的 `RETRY_INTERVAL`（90 秒）同时执行 ARM 和 Micro，OCI 会回传 `TooManyRequests` 错误。建议不要同时启动两个 workflow。

### 设置步骤

#### Step 1：取得这个 repo

在这个 repo 页面上方点 **Fork**，把它复制到你的 GitHub 帐号。请保持 **public**（public 才有无限制的 Actions 分钟数）。

如果你想自己建立一个 **public** repo，请建立空的 repo（不要加 README），再把程式码复制过去：

```bash
git clone https://github.com/WendyChen330/oci-arm-retry.git
cd oci-arm-retry
git remote set-url origin https://github.com/<你的 GitHub 帐号>/<repo 名称>.git
git push -u origin main
```

#### Step 2：设置 GitHub Secrets

到 repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

共需设置 **6 个** secret：

| Secret 名称 | 填入的值 | 取得位置 |
|------------|---------|----------|
| `OCI_USER` | `ocid1.user.oc1..xxxxxx` | 上方「取得 OCI API KEY」设置文字里的 `user` |
| `OCI_FINGERPRINT` | `xx:xx:xx:xx:xx` | 上方「取得 OCI API KEY」设置文字里的 `fingerprint` |
| `OCI_TENANCY` | `ocid1.tenancy.oc1..xxxxxx` | OCI 主控台 → 右上角 **Profile** 选项 → **租用户: &lt;名称&gt;** → OCID |
| `OCI_REGION` | 例如 `ap-singapore-1` | 主区域的区域识别码，见上方「查询主区域」。Always Free 资源只能建立在主区域 |
| `OCI_PRIVATE_KEY` | `.pem` 文件完整内容 | 开启上方「取得 OCI API KEY」下载的 `.pem` 文件，复制全部文字（含头尾 `-----BEGIN/END PRIVATE KEY-----`）|
| `OCI_SSH_PUBLIC_KEY` | `.pub` 文件完整内容 | 见上方「取得 SSH 公钥」（以 `ssh-` 开头）|

> 不需要另外建立 GitHub token：接力触发下一个 Job 和自动停用，都是用 GitHub 每个 Job 自动生成的内建 `GITHUB_TOKEN`。

#### Step 3：调整脚本设置（选择性）

想改机器名称、Ubuntu 版本等设置，在你的 repo 里修改脚本最上方的变数并 commit 即可，详见上方「脚本设置」。

#### Step 4：启动 workflow

1. 到 repo → **Actions** 页签。刚 fork 的 repo 要先点绿色的 **I understand my workflows, go ahead and enable them**
2. 在左侧选 **OCI ARM Retry** 或 **OCI Micro Retry**，点右侧的 **Run workflow**，再点选项里绿色的 **Run workflow**

ARM 和 Micro 各自有独立的 workflow，建议一次只启动一个（原因见「运作方式」下方的说明）。

---

## 本机执行

### 1. 下载这个 repo

需要先安装 Python 3 和 pip。下载这个 repo（**Code** → **Download ZIP**，或 `git clone`），并在它的资料夹开启终端机。

### 2. 建立 OCI 设置文件

**储存位置**：`~/.oci/config`（Windows 为 `C:\Users\<username>\.oci\config`，文件没有副文件名）

Windows 可以在 PowerShell 用下面的指令建立资料夹和空白文件，再用记事本打开：

```powershell
New-Item -ItemType Directory -Force "$HOME\.oci" | Out-Null
if (-not (Test-Path "$HOME\.oci\config")) { New-Item -ItemType File "$HOME\.oci\config" | Out-Null }
notepad "$HOME\.oci\config"
```

```
[DEFAULT]
user=ocid1.user.oc1..xxxxxx
fingerprint=xx:xx:xx:xx:xx
tenancy=ocid1.tenancy.oc1..xxxxxx
region=ap-singapore-1
key_file=C:\Users\<username>\.oci\oci_api_key.pem
ssh_public_key_file=C:\Users\<username>\.oci\ssh-key.pub
```

1. 贴上「取得 OCI API KEY」那段设置文字
2. 把 `key_file` 的路径改成 `.pem` 文件的实际存放位置
3. 确认 `region` 是你的主区域（见「查询主区域」）
4. 加上 `ssh_public_key_file`，填入你的 SSH 公钥（`.pub`，见「取得 SSH 公钥」）的路径。脚本会把这把密钥装到新机器上

> ⚠️ 文件路径请只用英文和数字。在 Windows 上，路径里有中文等非英文字元，可能会让设置文件读取失败。

### 3. 调整脚本设置（选择性）

机器名称、Ubuntu 版本等设置都在脚本最上方，请见上方「脚本设置」。

### 4. 安装套件并执行

在这个终端机执行：

```bash
# 安装套件（只需一次）
pip install oci

# 执行 ARM 版
python oci_retry.py

# 执行 Micro 版
python oci_retry_micro.py
```

---

## 脚本执行行为

| 状态 | 说明                              |
|------|---------------------------------|
| 初始化 | 自动建立 VCN + 子网路（已存在则跳过）          |
| 抢机循环 | 每隔 `RETRY_INTERVAL` 秒尝试一次       |
| 容量不足 | 输出 ❌ 并继续重试                      |
| 网路超时 | 输出 ⚠️ 并继续重试（不会当掉）               |
| 建立机器时的其他错误（例如额度已满）| 输出 ❌ 和错误讯息并继续重试；如果一直不成功，请查看 log |
| 初始化错误（例如 OCID 或 API KEY填错）| 在进入抢机循环前就出错停止                   |
| 成功 | 输出 ✅ 机器 ID 并自动停止                |

> **开放的 port：** 脚本建立新的 VCN 时，会对所有来源（`0.0.0.0/0`）开放 TCP **22（SSH）、80（HTTP）、443（HTTPS）**；如果 VCN 已经存在，则沿用原本的设置不做修改。需要其他 port 的话，可以在第一次执行前加进 `create_vcn_and_subnet()` 里的 port 清单，或之后到 OCI 主控台，在 VCN 的安全清单（Security List）新增输入规则（ingress rule）。另外，OCI 的 Ubuntu 映像文件还需要在机器的 `iptables` 放行该 port，做法请见 Oracle 官方 [Apache on Ubuntu 教学](https://docs.oracle.com/en-us/iaas/developer-tutorials/tutorials/apache-on-ubuntu/01oci-ubuntu-apache-summary.htm)的「Update firewall settings」步骤。

---

## 成功后

抢到机器后，workflow 会**自动停用**。如果执行结果显示 **Auto-disable failed** 错误，请手动停用：Actions 页签 → 该 workflow → **...** → **Disable workflow**。

到 OCI 主控台 → **Compute** → **执行处理** → 查看公用 IP，即可 SSH 连接：

```bash
ssh -i ssh-key.key ubuntu@<公用 IP>
```

请用「取得 SSH 公钥」那组的私钥，不是 OCI API 的 `.pem`。

---

## 注意事项

- OCI API 私钥等敏感信息全部放在 GitHub Secrets，不进版本控制
- **2026-11-07 更新：** workflow 不再使用 `GH_PAT` secret。如果你在这个日期之前 fork 或复制了这个 repo，请用这个 repo 目前的版本取代你的 `.github/workflows/oci_retry.yml` 和 `.github/workflows/oci_retry_micro.yml`（脚本不需要更新），之后就可以删掉 `GH_PAT` secret，并到 GitHub → Settings → Developer settings 撤销（revoke）那组 token
- **2026-11-08 更新：** 脚本移除了 `COMPARTMENT_ID` 和 `SSH_PUBLIC_KEY` 两个设置。机器现在一律建在你的租用户（根区间），SSH 公钥改从 GitHub Secrets 的 `OCI_SSH_PUBLIC_KEY`（GitHub Actions）或 OCI 设置文件的 `ssh_public_key_file`（本机执行）读取。如果你要把自己的脚本更新到这个版本：GitHub Actions 请同时更新 `.github/workflows/` 里的两个 `.yml` 文件并新增这个 secret；本机执行请在设置文件加上这个栏位
- `.pem` / `.key` / `.pub` 已加入 `.gitignore`，不会被 commit
