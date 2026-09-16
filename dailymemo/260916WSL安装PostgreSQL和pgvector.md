在 WSL (Windows Subsystem for Linux) 的 Ubuntu 环境中安装 PostgreSQL 和 pgvector 是非常顺畅的，因为 pgvector 官方对 Linux 的支持最完善。

以下是完整的图文级操作步骤，以及 **WSL 环境下特有的 3 个避坑指南**（非常重要）。

---

### 第一步：安装 PostgreSQL 及编译依赖

打开你的 Ubuntu 终端，依次执行以下命令：

```bash
# 1. 更新 Ubuntu 软件包列表
sudo apt update

# 2. 安装 PostgreSQL、C/C++ 编译工具 (build-essential) 和 Git
# 注意：postgresql-server-dev-all 包含了编译 pgvector 所需的 PG 头文件
sudo apt install -y build-essential git postgresql postgresql-server-dev-all
```

### 第二步：启动 PostgreSQL 服务

WSL 默认可能没有开启 `systemd`，因此我们使用传统的 `service` 命令来启动数据库：

```bash
sudo service postgresql start
```

*(💡 提示：每次重启 WSL 或 Windows 电脑后，数据库服务会停止，需要重新运行此命令。)*

### 第三步：下载并编译 pgvector

在终端中执行以下命令，从 GitHub 拉取最新代码并编译安装：

```bash
# 进入临时目录
cd /tmp

# 克隆 pgvector 仓库 (使用目前最新的 v0.8.6 版本)
git clone --branch v0.8.6 https://github.com/pgvector/pgvector.git
cd pgvector

# 编译并安装到 PostgreSQL 的扩展目录中
make
sudo make install
```

*如果看到类似 `mkdir -p '/usr/share/postgresql/16/extension'` 的输出且没有报错，说明编译安装成功。*

### 第四步：在数据库中启用 pgvector

现在我们需要进入 PostgreSQL 的命令行，将 pgvector 注册为可用扩展：

```bash
# 切换到 postgres 用户并进入 psql 交互终端
sudo -u postgres psql
```

在出现的 `postgres=#` 提示符下，执行以下 SQL 语句：

```sql
-- 1. 创建 pgvector 扩展
CREATE EXTENSION vector;

-- 2. 验证是否安装成功（查看已安装的扩展列表）
\dx
```

如果输出列表中包含 `vector` 及其版本号（如 `0.8.6`），说明安装完美成功。输入 `\q` 退出 psql。

### 第五步：快速测试（可选）

你可以直接在 psql 中运行以下 SQL，测试向量检索功能：

```sql
-- 创建一个包含 3 维向量的表
CREATE TABLE items (id bigserial PRIMARY KEY, embedding vector(3));

-- 插入两条测试数据
INSERT INTO items (embedding) VALUES ('[1,2,3]'), ('[4,5,6]');

-- 查询与 [3,1,2] 距离最近的向量 (使用 L2 欧氏距离)
SELECT * FROM items ORDER BY embedding <-> '[3,1,2]' LIMIT 5;
```

---

### 🚨 WSL 环境下的 3 个重要避坑指南

在 WSL 中运行数据库，有几个极易踩坑的地方，请务必注意：

#### 1. 性能警告：千万不要跨文件系统

**绝对不要**把 PostgreSQL 的数据目录（默认在 `/var/lib/postgresql/...`）映射、移动或软链接到 Windows 的 `/mnt/c/` 目录下。

* **原因**：WSL 跨文件系统（Windows NTFS 到 Linux ext4）的 I/O 性能会**下降 10 倍以上**，会导致数据库读写极慢。
* **正确做法**：让数据乖乖待在 WSL 的 Linux 虚拟磁盘内（即默认的 `/var/lib/...` 路径）。

#### 2. 内存黑洞：限制 WSL 的最大内存

WSL2 默认会“贪婪”地占用 Windows 一半以上的物理内存，如果不加限制，可能会导致你的 Windows 宿主机卡顿。

* **解决方法**：在 **Windows 宿主机** 的 `C:\Users\<你的用户名>\` 目录下，创建一个名为 `.wslconfig` 的文件（注意开头有个点），写入以下内容：
  ```ini
  [wsl2]
  memory=4GB   # 限制 WSL 最多使用 4GB 内存（根据你的电脑总内存调整）
  swap=2GB     # 限制交换空间
  ```
* 保存后，在 Windows 的 PowerShell 中运行 `wsl --shutdown`，然后重新打开 Ubuntu 即可生效。

#### 3. 服务不会开机自启

WSL 每次启动都是全新的会话，PostgreSQL 不会自动运行。

* **懒人解决办法**：每次打开 Ubuntu 终端时，习惯性地敲一下 `sudo service postgresql start`。
* **自动化办法**：可以在 Ubuntu 的 `~/.bashrc` 文件末尾加上一行判断代码，实现打开终端时自动静默启动：
  ```bash
  # 打开 ~/.bashrc，在文件末尾添加：
  if service postgresql status | grep -q "is stopped"; then
      sudo service postgresql start > /dev/null 2>&1
  fi
  ```

在 WSL2 中安装的 PostgreSQL 默认只允许本地（WSL 内部）连接，且默认使用系统用户认证（peer）。要让 Windows 宿主机的 pgAdmin 连接它，需要完成 **3 个核心配置**：设置数据库密码、开放网络监听、配置访问控制。

以下是保姆级操作步骤：

---

### 第一步：为 `postgres` 用户设置密码

pgAdmin 需要通过密码连接数据库，而 WSL 默认安装的 `postgres` 用户是没有密码的。

1. 在 WSL 终端中进入 psql：
   ```bash
   sudo -u postgres psql
   ```
2. 在 `postgres=#` 提示符下，修改密码（将 `yourpassword` 替换为你自己想设置的密码）：
   ```sql
   ALTER USER postgres PASSWORD 'yourpassword';
   ```
3. 输入 `\q` 退出 psql。

---

### 第二步：修改 PostgreSQL 配置文件（允许外部连接）

我们需要修改两个配置文件，让 PostgreSQL 监听所有网络接口，并允许外部 IP 访问。

1. **查找配置文件路径**：
   在终端运行以下命令查看你的 PostgreSQL 版本和配置路径（假设输出是 `/etc/postgresql/16/main/postgresql.conf`，那么版本号就是 `16`）：
   
   ```bash
   sudo -u postgres psql -c "SHOW config_file;"
   ```
2. **修改 `postgresql.conf`（开放监听）**：
   使用 nano 编辑器打开该文件（**请将下面命令中的 `16` 替换为你实际的版本号**）：
   
   ```bash
   sudo nano /etc/postgresql/16/main/postgresql.conf
   ```
   
   找到 `#listen_addresses = 'localhost'` 这一行（按 `Ctrl+W` 搜索），去掉前面的 `#` 注释，并将值改为 `*`：
   
   ```ini
   listen_addresses = '*'
   ```
   
   保存并退出（按 `Ctrl+O` 回车保存，`Ctrl+X` 退出）。
3. **修改 `pg_hba.conf`（允许密码登录）**：
   打开同目录下的访问控制文件：
   
   ```bash
   sudo nano /etc/postgresql/16/main/pg_hba.conf
   ```
   
   滚动到文件**最末尾**，添加以下两行（允许所有 IPv4 地址通过密码连接）：
   
   ```text
   # 允许所有 IPv4 地址使用密码连接 (适用于 PG 14 及以上版本)
   host    all             all             0.0.0.0/0               scram-sha-256
   # 如果你的 PG 版本低于 14，请将上面的 scram-sha-256 改为 md5
   ```
   
   保存并退出。
4. **重启 PostgreSQL 服务**使配置生效：
   
   ```bash
   sudo service postgresql restart
   ```

---

### 第三步：在 pgAdmin 中连接数据库

从 Windows 10 Build 21362 和 Windows 11 开始，微软引入了 **localhost 映射** 功能，Windows 可以直接通过 `localhost` 访问 WSL2 内部的服务。

打开 Windows 上的 pgAdmin，右键点击 **Servers** -> **Register** -> **Server**，按以下信息填写：

#### 1. General (常规) 选项卡

* **Name**: 随便填，例如 `WSL-PostgreSQL`

#### 2. Connection (连接) 选项卡

* **Host name/address**: 填写 `localhost` （**首选**，最稳定）
  *(💡 如果 `localhost` 连不上，请在 WSL 终端输入 `ip addr show eth0 | grep inet` 获取 WSL 的 IP 地址（如 `172.x.x.x`），填在这里。)*
* **Port**: `5432`
* **Maintenance database**: `postgres`
* **Username**: `postgres`
* **Password**: 第一步中你设置的 `yourpassword`
* **Save password**: 勾选（可选）

点击 **Save**，如果左侧树状图展开并显示数据库列表，说明连接成功！

---

### 🚨 常见连接失败排查指南

如果点击 Save 后报错，请按以下顺序排查：

#### 报错 1：`Connection refused` (拒绝连接)

* **原因 1**：PostgreSQL 没启动。在 WSL 运行 `sudo service postgresql start`。
* **原因 2**：使用了 WSL 的 IP 连接，但 WSL 重启后 IP 变了。**解决办法**：在 pgAdmin 的 Host 中改回 `localhost`。
* **原因 3**：Windows 防火墙拦截。在 Windows 搜索“Windows Defender 防火墙” -> “允许应用通过防火墙”，确保 pgAdmin 被允许，或者临时关闭防火墙测试。

#### 报错 2：`FATAL: password authentication failed` (密码认证失败)

* **原因 1**：密码填错了。
* **原因 2**：`pg_hba.conf` 中的加密方式不匹配。如果你用的是 PG 14+，必须写 `scram-sha-256`；如果是 PG 13 及以下，必须写 `md5`。修改后记得 `sudo service postgresql restart`。

#### 报错 3：`FATAL: no pg_hba.conf entry for host...`

* **原因**：`pg_hba.conf` 没配置好。检查是否成功添加了 `host all all 0.0.0.0/0 ...` 这一行，并且没有语法错误。

### 💡 进阶建议：固定 WSL 的 IP（如果你必须用 IP 连接）

如果你因为某些原因必须使用 WSL 的 IP（如 `172.x.x.x`）连接，由于 WSL2 每次重启 IP 都会变，非常麻烦。你可以在 Windows 的 `C:\Users\<你的用户名>\.wslconfig` 文件中加入以下配置，让 WSL 使用镜像网络模式（此时 WSL 和 Windows 共享同一个 IP，直接用 `localhost` 即可）：

```ini
[wsl2]
networkingMode=mirrored
```

保存后在 PowerShell 运行 `wsl --shutdown` 重启 WSL 即可。

你使用的命令有误。正确的查看 IP 地址的命令是：

```bash
# 方法 1：查看所有网络接口信息（推荐）
ip addr show

# 或者简写为：
ip a

# 方法 2：只看 IPv4 地址
ip -4 addr show

# 方法 3：只看特定接口（如 eth0）
ip addr show eth0

# 方法 4：使用 ifconfig（如果已安装）
ifconfig

# 方法 5：使用 hostname（查看本机 IP）
hostname -I
```

**你刚才输入的是 `ip -a`，但正确的简写应该是 `ip a`（中间没有横杠）**。

在 WSL 中，通常你的 IP 地址会显示在 `eth0` 网卡下，查找 `inet` 后面的地址就是 IPv4 地址。




