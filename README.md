使用 GitHub Actions 定时爬取任意网址（域名）的 DNS 解析记录，并生成Hosts文件格式，是一个非常实用的自动化需求。我们可以利用 Linux 自带的 `dig` 命令来完成查询，并结合 GitHub Actions 的 `schedule` 功能实现定时触发，最后将结果提交回仓库保存历史记录。

以下是完整的操作步骤和代码实现。

### 核心思路

1.  **触发机制**：利用 GitHub Actions 的 `cron` 语法定时运行（如每小时一次）。
2.  **解析逻辑**：在运行器中提取网址中的**域名**，使用 `dig` 命令查询 DNS 记录。
3.  **数据存储**：将查询结果追加到一个文件中，并利用 `git` 命令提交回仓库，形成历史监控日志。

---

### 第一步：创建仓库和文件

1.  在 GitHub 上新建一个仓库（例如命名为 `dns-monitor`）。
2.  在仓库中创建目录结构：`.github/workflows/`
3.  在该目录下创建一个 YAML 文件，例如 `scratchdns-check.yml`。

### 第二步：编写 Workflow 脚本

将以下代码复制到 `scratchdns-check.yml` 中。

### 第三步：创建网址配置文件
监控多个网址，最好的做法是将网址列表放在一个单独的配置文件中（例如 `urls.txt`），然后通过脚本循环读取这个文件进行批量查询。
在你的仓库根目录下新建一个名为 `urls.txt` 的文件，把你要监控的网址一行写一个。支持写 `http://` 开头的完整链接，也可以直接写域名。

这样管理起来非常方便，以后只需要修改 `urls.txt` 文件，不用去动 GitHub Actions 的配置代码。

**`urls.txt` 内容示例：**
```text
https://www.google.com
https://github.com
bing.com
https://www.apple.com
# 这是注释，以 # 开头的一行会被自动忽略
example.com
```

### 代码关键点解析

1.  **Cron 定时 (`schedule`)**:
    *   `- cron: '0 * * * *'` 代表每小时第 0 分钟执行一次。
    *   **注意时区**：GitHub Actions 默认使用 **UTC 时间**。如果你需要北京时间（UTC+8）早上 8 点运行，你需要设置 UTC 时间 0 点（即 `0 0 * * *`）。最小间隔通常是 5 分钟。

2.  **域名提取**:
    *   因为 DNS 只能解析域名（如 `baidu.com`），不能解析带路径的完整 URL。脚本中 `sed` 命令负责把 `https://www.baidu.com/s?wd=test` 处理成 `www.baidu.com`。

3.  **dig 命令**:
    *   `dig +short $DOMAIN A`：这是最核心的命令。
    *   `A` 代表查询 IPv4 地址。如果你想查询 IPv6，可以改成 `AAAA`；查询 CNAME（别名），可以改成 `CNAME`。



4.  **日志文件 (`dns_current.log`)**：
    *   文件里永远只保留**最新一次**的数据。
    *   格式严格按照你的要求：`IP 域名 # 注释`。

5.  **`get_old_ip` 函数**：
    *   这是核心。在生成新日志前，它利用 `awk` 命令去读取现有的 `dns_current.log`。
    *   它会比较每一行的第2列（`$2`），如果匹配当前的域名，就返回第1列（`$1`，即旧的IP）。

6.  **失败处理逻辑**：
    *   如果 `dig` 查不到 IP：
        *   先调用 `get_old_ip`。
        *   如果拿到了旧 IP，就输出 `1.2.3.4 example.com # Timeout`。
        *   如果从来没查到过这个域名（旧日志里也没有），就输出 `Unknown example.com # Timeout`，保证日志格式的一致性。
7. **只读取输出的第一行**：
非常简单，只需要在 `dig` 命令后加上管道符 `| head -n 1` 即可。

`head -n 1` 的作用是只读取输出的第一行，这样即使 DNS 返回了多个 IP（比如 CDN 返回的多个节点），我们也只取第一个，保证日志格式整洁（一行一个域名）。
关键代码在这一行：

```bash
NEW_IP=$(dig +short "$DOMAIN" A +time=3 +tries=1 2>/dev/null | head -n 1)
```

*   **没有 `head -n 1` 时**：如果查询 `google.com`，`dig` 可能会返回：
    ```text
    142.250.1.100
    142.250.1.101
    142.250.1.102
    ```
    这会导致你的日志文件变成：
    ```text
    142.250.1.100
    142.250.1.101
    142.250.1.102
     www.google.com
    ```
    格式完全错乱。

*   **加上 `head -n 1` 后**：只抓取第一行 `142.250.1.100`。
    日志文件保持完美格式：
    ```text
    142.250.1.100 www.google.com
    ```
7.  **临时文件 (`dns_current.tmp`)**：
    *   我们先往 `.tmp` 文件里写，写完了再把 `.tmp` 改名为 `.log`。这样做是为了防止万一脚本中途出错，导致原来的日志文件被清空或损坏。


8.  **自动提交**:
    *   为了让机器人能提交代码，我们需要 `permissions: contents: write`。
    *   使用了 `git status --porcelain` 来检查是否有变化，如果 DNS 记录没变（且之前没跑过），就不会触发无效的 commit。


以下是更新后的完整 Workflow 代码：

```yaml
name: DNS Monitor (Latest Only)

on:
  schedule:
    # 每小时运行一次
    - cron: '0 * * * *'
  workflow_dispatch: # 允许手动触发

permissions:
  contents: write

jobs:
  check-dns:
    runs-on: ubuntu-latest
    steps:
      # 1. 检出代码
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      # 2. 配置 Git
      - name: Configure Git
        run: |
          git config --global user.name "GitHub Actions Bot"
          git config --global user.email "actions@github.com"

      # 3. 执行 DNS 查询与日志生成
      - name: Query DNS and Update Log
        run: |
          # 定义文件名
          LOG_FILE="dns_current.log"
          TMP_LOG="dns_current.tmp"
        
          # --- 函数：获取某个域名的上一次IP ---
          function get_old_ip() {
            local domain=$1
            # 从旧日志查找对应域名的IP (精确匹配第2列)
            awk -v d="$domain" '$2 == d {print $1}' "$LOG_FILE" 2>/dev/null | head -n 1
          }

          # --- 开始处理 ---
        
          # 创建临时文件，写入表头
          echo "# Latest DNS Status - Updated: $(date -u +"%Y-%m-%d %H:%M:%S UTC")" > "$TMP_LOG"
          echo "# Format: IP Domain # Optional Comment" >> "$TMP_LOG"
        
          if [ ! -f "urls.txt" ]; then
            echo "Error: urls.txt not found!"
            exit 1
          fi

          # 循环读取 urls.txt
          while IFS= read -r line || [[ -n "$line" ]]; do
            # 跳过空行和注释
            if [[ -z "$line" || "$line" =~ ^# ]]; then
              continue
            fi

            # 提取域名
            DOMAIN=$(echo "$line" | sed -e 's|^[^/]*//||' -e 's|/.*$||')
          
            # 查询 DNS (A记录)
            # --- 核心修改在这里 ---
            # 新增了: | head -n 1
            # 作用: 无论 dig 返回多少个 IP，只取第一个
            NEW_IP=$(dig +short "$DOMAIN" A +time=3 +tries=1 2>/dev/null | head -n 1)

            # --- 判断结果 ---
            if [ -n "$NEW_IP" ]; then
              # 1. 查询成功：写入 "IP 域名"
              echo "$NEW_IP $DOMAIN" >> "$TMP_LOG"
            else
              # 2. 查询失败：尝试获取上一次的 IP
              OLD_IP=$(get_old_ip "$DOMAIN")
            
              if [ -n "$OLD_IP" ]; then
                # 有历史记录：写入 "旧IP 域名 # Timeout"
                echo "$OLD_IP $DOMAIN # Timeout" >> "$TMP_LOG"
              else
                # 无历史记录：标记为 Unknown
                echo "Unknown $DOMAIN # Timeout (No History)" >> "$TMP_LOG"
              fi
            fi

          done < "urls.txt"

          # 覆盖旧日志
          mv "$TMP_LOG" "$LOG_FILE"
        
          # 输出结果预览
          echo "=== Current Log File Contents ==="
          cat "$LOG_FILE"

      # 4. 提交更改
      - name: Commit changes
        run: |
          if [[ -n $(git status --porcelain) ]]; then
            git add dns_current.log
            git commit -m "Update latest DNS snapshot"
            git push
          else
            echo "No changes to commit."
          fi
```

### 第五步：如何使用

1.  将上述文件保存并推送到 GitHub。
2.  你可以点击仓库上方的 **Actions** 标签页。
3.  找到 **"DNS Monitor"** workflow，点击右侧的 **"Run workflow"** 按钮进行第一次手动测试。
4.  测试成功后，代码库根目录下会出现一个 `dns_current.log` 文件。
5.  之后，它会根据你设定的时间自动运行并更新这个文件。




这样你就可以清晰地看到所有网址在不同时间点的解析状态变化了。
### 进阶玩法：查询多条记录并发送通知

如果你需要监控多个网址，或者希望解析结果发生变化时发送邮件/钉钉通知，可以修改脚本逻辑：

```bash
# 示例：简单的变化检测
# 假设上次的结果存在 last_ip.txt

if [ "$CURRENT_IP" != "$(cat last_ip.txt 2>/dev/null)" ]; then
  echo "IP Changed! New IP: $CURRENT_IP"
  echo "$CURRENT_IP" > last_ip.txt
  # 这里可以调用 curl 发送 Webhook 通知
  # curl -X POST "你的钉钉/Slack Webhook URL" -d '{"text":"DNS Changed"}'
fi
```

这个方案无需任何服务器，完全依托于 GitHub 的免费资源，非常适合个人开发者做长期监控。


Hosts管理,推荐用SwitchHosts一键切换，SwitchHosts 的 官网(https://switchhosts.vercel.app/ ) 或 Github仓库(https://github.com/oldj/SwitchHosts ) ,下载对应系统版本的安装包