使用 GitHub Actions 定时爬取任意网址（域名）的 DNS 解析记录是一个非常实用的自动化需求。我们可以利用 Linux 自带的 dig 命令来完成查询，并结合 GitHub Actions 的 schedule 功能实现定时触发，最后将结果提交回仓库保存历史记录。

以下是完整的操作步骤和代码实现。

核心思路
触发机制：利用 GitHub Actions 的 cron 语法定时运行（如每小时一次）。
解析逻辑：在运行器中提取网址中的域名，使用 dig 命令查询 DNS 记录。
数据存储：将查询结果追加到一个文件中，并利用 git 命令提交回仓库，形成历史监控日志。
第一步：创建仓库和文件
在 GitHub 上新建一个仓库（例如命名为 dns-monitor）。
在仓库中创建目录结构：.github/workflows/
在该目录下创建一个 YAML 文件，例如 dns-check.yml。
第二步：编写 Workflow 脚本
将以下代码复制到 dns-check.yml 中。

<YAML>
name: DNS Monitor
# 触发条件：定时任务 (cron) 和 手动触发
on:
  schedule:
    # Cron 表达式：这里设置每小时运行一次 (UTC时间)
    # 分 时 日 月 周
    - cron: '0 * * * *'
  workflow_dispatch: # 允许你在 GitHub 网页上手动点击运行
# 设置权限，允许脚本推送到代码库
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
          token: ${{ secrets.GITHUB_TOKEN }} # 使用自带的 Token 进行提交
      # 2. 配置 Git 用户信息 (必须，否则无法提交)
      - name: Configure Git
        run: |
          git config --global user.name "GitHub Actions Bot"
          git config --global user.email "actions@github.com"
      # 3. 运行 DNS 查询脚本
      - name: Query DNS Record
        id: dns_query
        env:
          # 在这里修改你要监控的网址（URL），脚本会自动提取域名
          TARGET_URL: "https://www.google.com"
        run: |
          # --- 脚本开始 ---
          
          # 1. 从 URL 中提取纯域名 (例如把 https://www.google.com 变成 www.google.com)
          DOMAIN=$(echo "$TARGET_URL" | sed -e 's|^[^/]*//||' -e 's|/.*$||')
          
          # 2. 获取当前时间 (UTC)
          TIME=$(date -u +"%Y-%m-%d %H:%M:%S UTC")
          
          # 3. 执行 dig 命令查询 A 记录 (IPv4)
          # +short: 只输出 IP 地址，去掉其他多余信息
          # +noall: 不显示默认的 header 等信息
          # +answer: 显示回答部分
          DNS_RESULT=$(dig +short $DOMAIN A +noall +answer)
          
          # 如果查询结果为空，设置默认值
          if [ -z "$DNS_RESULT" ]; then
            DNS_RESULT="No Record Found / Timeout"
          fi
          # 4. 将结果写入日志文件 history.log
          # 格式：时间 | 域名 | 解析结果
          echo "$TIME | $DOMAIN | $DNS_RESULT" >> history.log
          
          # --- 脚本结束 ---
          
          # 打印结果到控制台方便查看
          cat history.log
          echo "Query finished."
      # 4. 提交更改到 GitHub
      - name: Commit changes
        run: |
          # 检查是否有文件变更
          if [[ -n $(git status --porcelain) ]]; then
            git add history.log
            git commit -m "Update DNS record log"
            git push
          else
            echo "No changes to commit."
          fi
代码关键点解析
Cron 定时 (schedule):

- cron: '0 * * * *' 代表每小时第 0 分钟执行一次。
注意时区：GitHub Actions 默认使用 UTC 时间。如果你需要北京时间（UTC+8）早上 8 点运行，你需要设置 UTC 时间 0 点（即 0 0 * * *）。最小间隔通常是 5 分钟。
域名提取:

因为 DNS 只能解析域名（如 baidu.com），不能解析带路径的完整 URL。脚本中 sed 命令负责把 https://www.baidu.com/s?wd=test 处理成 www.baidu.com。
dig 命令:

dig +short $DOMAIN A：这是最核心的命令。
A 代表查询 IPv4 地址。如果你想查询 IPv6，可以改成 AAAA；查询 CNAME（别名），可以改成 CNAME。
自动提交:

为了让机器人能提交代码，我们需要 permissions: contents: write。
使用了 git status --porcelain 来检查是否有变化，如果 DNS 记录没变（且之前没跑过），就不会触发无效的 commit。
第三步：如何使用
将上述文件保存并推送到 GitHub。
你可以点击仓库上方的 Actions 标签页。
找到 "DNS Monitor" workflow，点击右侧的 "Run workflow" 按钮进行第一次手动测试。
测试成功后，代码库根目录下会出现一个 history.log 文件。
之后，它会根据你设定的时间自动运行并更新这个文件。
进阶玩法：查询多条记录并发送通知
如果你需要监控多个网址，或者希望解析结果发生变化时发送邮件/钉钉通知，可以修改脚本逻辑：

<BASH>
# 示例：简单的变化检测
# 假设上次的结果存在 last_ip.txt
CURRENT_IP=$(dig +short example.com A +noall +answer | head -n 1)
if [ "$CURRENT_IP" != "$(cat last_ip.txt 2>/dev/null)" ]; then
  echo "IP Changed! New IP: $CURRENT_IP"
  echo "$CURRENT_IP" > last_ip.txt
  # 这里可以调用 curl 发送 Webhook 通知
  # curl -X POST "你的钉钉/Slack Webhook URL" -d '{"text":"DNS Changed"}'
fi
这个方案无需任何服务器，完全依托于 GitHub 的免费资源，非常适合个人开发者做长期监控。