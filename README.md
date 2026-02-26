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
读取旧日志：在开始查询前，先读取上一次生成的日志文件（如果存在），把里面的 IP 提取出来暂存。
生成新日志：不再使用追加模式（>>），而是创建一个新的临时文件，将本次结果写入其中。
失败回退逻辑：如果查询失败，去暂存的旧数据里找有没有这个域名的上一次 IP，有就拿来用，没有就标记为未知。
覆盖原文件：最后用新文件覆盖旧文件
代码逻辑解析
日志文件 (dns_current.log)：

文件里永远只保留最新一次的数据。
格式严格按照你的要求：IP 域名 # 注释。
get_old_ip 函数：

这是核心。在生成新日志前，它利用 awk 命令去读取现有的 dns_current.log。
它会比较每一行的第2列（$2），如果匹配当前的域名，就返回第1列（$1，即旧的IP）。
失败处理逻辑：

如果 dig 查不到 IP：
先调用 get_old_ip。
如果拿到了旧 IP，就输出 1.2.3.4 example.com # Timeout。
如果从来没查到过这个域名（旧日志里也没有），就输出 Unknown example.com # Timeout，保证日志格式的一致性。
临时文件 (dns_current.tmp)：

我们先往 .tmp 文件里写，写完了再把 .tmp 改名为 .log。这样做是为了防止万一脚本中途出错，导致原来的日志文件被清空或损坏。
预期效果
运行后，你的仓库根目录下的 dns_current.log 内容将会是这样的：

情况 A：正常解析

<TEXT>
142.250.183.36 www.google.com
140.82.113.4 github.com
13.107.21.200 bing.com
情况 B：某次查询 bing.com 超时
(它保留了上一次解析到的 IP 13.107.21.200 并添加了注释)

<TEXT>
142.250.183.36 www.google.com
140.82.113.4 github.com
13.107.21.200 bing.com # Timeout
情况 C：从未解析成功过的新域名超时

<TEXT>
Unknown fake-site.com # Timeout (No History)

希望解析结果发生变化时发送邮件/钉钉通知，可以修改脚本逻辑：

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