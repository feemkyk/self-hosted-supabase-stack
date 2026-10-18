# self hosted supabase：Docker 堆栈的实际需求、如何在 VPS 上部署，以及如何选择服务器配置

自托管 Supabase 在你控制的服务器上运行完整的开源后端——包括 Postgres、身份验证、自动 REST API、实时广播、文件存储、边缘函数和 Studio 仪表板。它作为一个包含 11 个容器的 Docker Compose 部署，获得 Apache 2.0 许可，没有单项目或单席位费用，数据保留在你的硬件上。权衡之处在于你继承了原本由 Supabase 团队处理的运维工作——更新、安全、备份、正常运行时间——而且该堆栈需要的内存比大多数人预期的要多：**官方最低 4 GB，建议 8 GB**。

> **简要推荐：** 如果预算允许，在 **High Performance 8 GB** 实例（约 48 美元/月）上运行完整的堆栈；4 GB（约 24 美元/月）满足最低要求。如果只是试验精简后的堆栈，可以从 6 美元/月的 1 GB 方案开始。👉 [查看当前的 Vultr 方案定价](https://www.vultr.com/pricing/?ref=7268283)

## 自托管 Supabase 堆栈中实际包含什么

Docker 部署会启动 11 个容器，每个容器都有特定的用途。了解映射关系有助于在后续排查问题时理清思路：

- **PostgreSQL** — 其他所有组件都围绕的核心数据库
- **Auth (GoTrue)** — 基于 JWT 的注册、登录、魔术链接、OAuth
- **PostgREST** — 自动将你的架构转换为 REST API
- **Realtime** — 通过 WebSockets 广播 Postgres 行变更的 Elixir 服务
- **Storage** — 文件上传和检索，权限存储在 Postgres 中，文件存储在本地磁盘或 S3 上
- **Studio** — 浏览器仪表板
- **API 网关 (Envoy)** — 将 `/auth`、`/rest` 和 `/storage` 路由到正确的服务；可以覆盖为 Kong
- **Supavisor** — 连接池管理器，防止你的应用耗尽数据库连接
- **imgproxy** — 为存储的文件进行即时图像调整大小
- **postgres-meta** — 用于管理 Postgres（表、角色、查询）的 REST API
- **Edge Functions** — 用于无服务器函数的 Deno 运行时

你不需要运行所有这些。Realtime、Storage、imgproxy、Edge Functions 和分析部分可以从 `docker-compose.yml` 中移除，空闲内存占用会相应下降。一个勉强能用的后端需要 PostgreSQL、Auth、PostgREST 和 API 网关——四个容器。作为交换，你失去了仪表板和一些便利功能，但一个精简的堆栈可以很好地放入小型服务器。

还有一点值得注意：大多数设置都通过环境变量进行配置，因此 `.env` 文件实际上就是部署的大脑。妥善备份它，以后可以省去很多麻烦。

## 自托管 Supabase 与 Supabase Cloud 的取舍

自托管保留了核心产品——Auth、Storage、Realtime、REST API、Edge Functions——但平台端的便利功能消失了：分支 (branching)、日志之外的高级指标、托管备份和时间点恢复 (PITR)、分析和向量存储桶 (vector buckets)、ETL 工具以及管理 API。每次部署也仅限一个项目，因此 Studio 无法像云控制平面那样让你在组织之间切换。

备份需要自己处理。使用 `cron` 定时任务执行 `pg_dump` 到服务器之外，如果你需要仪表板级的监控和指标，可以使用 Supabase 社区维护的单独可观测性项目。支持也是基于社区的——GitHub discussions、Discord 和 subreddit——这对有耐心的业余爱好者来说没问题，也是企业在采用此方案前需要三思的原因。

这里有一个经常让用户感到惊喜的细节：自托管堆栈本身不会发送遥测数据，也不会向任何地方汇报。只有 Supabase CLI（一个单独的开发工具）会收集使用遥测数据，并且有相应的退出选项。

## 服务器需求：内存是首先会遇到瓶颈的资源

官方需求很明确，内存是首先耗尽的资源：

| 资源 | 最低要求 | 推荐配置 |
| --- | --- | --- |
| 内存 | 4 GB | 8 GB+ |
| CPU | 2 核 | 4 核+ |
| 磁盘 | 40 GB SSD | 80 GB SSD+ |

原因是 Studio。它是空闲时最重的单个容器，而且 Postgres 会随着数据的增加而持续占用内存。在 4 GB 的机器上，一旦 Postgres、Supavisor 和其他所有服务都驻留内存，就没有多少余量了；8 GB 则是真正的舒适区。磁盘同样如此——矢量数据、存储文件和日志增长得比你预期的要快，因此 80 GB 起步可以省去以后的迁移。

用最低的四个容器进行测试是另一回事。这可以装入 1 GB 的机器，虽然很紧凑。👉 [浏览 Vultr 的云计算方案配置](https://www.vultr.com/products/cloud-compute/?ref=7268283)

## Vultr 方案对应 Supabase 的工作负载

Vultr 的云计算产品线有多个版本——Regular Performance、High Frequency 和 High Performance——它们具有相同的入门规格，但采用不同的 CPU 代数和存储类型。以下是最适合自托管 Supabase 的方案：

| Vultr 方案系列 | 配置 | 价格 | Supabase 兼容性 | 链接 |
| --- | --- | --- | --- | --- |
| Cloud Compute – Regular Performance | 512 MB / 1 vCPU / 10 GB SSD | $2.50/mo | 低于内存限制；仅用于 Postgres 试验 | [查看定价](https://www.vultr.com/pricing/?ref=7268283) |
| Cloud Compute – Regular Performance | 1 GB / 1 vCPU / 25 GB SSD | $5/mo | 最多支持精简的四容器测试堆栈 | [查看定价](https://www.vultr.com/pricing/?ref=7268283) |
| Cloud Compute – High Frequency | 1 GB / 1 vCPU / 25 GB NVMe | $6/mo | 适合开发/预发环境的精简堆栈 | [对比方案](https://www.vultr.com/products/cloud-compute/?ref=7268283) |
| Cloud Compute – High Performance | 4 GB / 2 vCPU | ~$24/mo | 处于官方最低要求的完整 11 容器堆栈 | [查看当前定价](https://www.vultr.com/pricing/?ref=7268283) |
| Cloud Compute – High Performance | 8 GB / 4 vCPU | ~$48/mo | 推荐配置；为数据和流量增长留有余地 | [查看当前定价](https://www.vultr.com/pricing/?ref=7268283) |
| Bare Metal | 独占硬件 | 更高 | 多项目设置、持续高负载 | [查看裸金属方案](https://www.vultr.com/products/bare-metal/?ref=7268283) |

定价说明有两个：显示的价格是该系列长期的起步价格，请务必在定价页面上确认当前数字。同时请注意 1 GB 套餐自带 25 GB 存储——低于 40 GB 的磁盘底线——这足以运行精简的测试堆栈，但不足以容纳完整堆栈。Vultr 按小时计费，设有月度上限，因此部署一个周末进行实验，成本只需几美分，销毁实例即可停止计费。已托管数据库行可以忽略；Supabase 捆绑了它自己的 Postgres，因此你不会用到它。

## 在 Vultr VPS 上部署堆栈：两种途径

快速路径是 Linux 下的一个脚本。它适用于 Debian/Ubuntu 和 RHEL 系列发行版，如果缺失会自动安装 Docker，获取 Compose 配置，生成所有密钥（包括较新的非对称 JWT 密钥对），并提示你输入 URL：

bash
curl -fsSL https://supabase.link/setup.sh | sh
cd supabase-project
sh run.sh start


稍后使用 `sh run.sh secrets` 查看生成的凭据。Studio 在实例的 8000 端口上监听——使用安装程序生成的仪表板用户名和密码登录。

如果你希望控制每一个步骤，或者使用不支持的发行版，请手动克隆官方仓库，并固定来自 Supabase 发布页面的最新 `self-hosted` 标签：

bash
git clone --depth 1 --branch self-hosted/v0.8.1 https://github.com/supabase/supabase
mkdir supabase-project
cp -rf supabase/docker/. supabase-project
cd supabase-project && cp .env.example .env
docker compose pull


首次启动**之前**，生成密钥并设置面向外部的 URL：

bash
sh utils/generate-keys.sh
sh utils/add-new-auth-keys.sh


bash
SUPABASE_PUBLIC_URL=https://your-domain.com:8000
API_EXTERNAL_URL=https://your-domain.com:8000/auth/v1
SITE_URL=https://your-domain.com


然后运行 `sh run.sh start`。

> ⚠️ 请勿使用示例密钥启动堆栈。使用默认设置启动然后再轮换密钥会导致连接池挂起，且没有任何有用的错误提示——这是最难排查的故障之一。在首次启动前生成一次密钥，可以完全避免这个问题。

在公开流量之前，请在其前置一个 TLS 反向代理——使用 Let's Encrypt 自动证书的 Caddy 或 Nginx 是常见的选择——并将实例放置在靠近用户的区域，因为繁重的 REST API 使用意味着对 Postgres 的频繁往返。

## 自托管 Supabase 的实际成本

软件费用为零：Apache 2.0 许可证意味着没有单项目费用，也没有席位费。你的账单是实例费用——最低要求配置约 24 美元/月，推荐配置约 48 美元/月。

真正的成本是时间。你会负责服务器维护、安全加固、操作系统和服务更新、Postgres 维护、高可用性、备份以及监控。这个堆栈很稳定，但它不会自我更新。预算中最重要的项目是备份：对数据库进行 cron 定时的 `pg_dump` 到单独的存储中，Vultr 可以添加自动服务器级备份作为低成本附加项——这是方便的保险，但逻辑数据库转储在真正的恢复中仍然胜出。👉 [查看按小时计费的方案](https://www.vultr.com/pricing/?ref=7268283)

## 运行多个项目

自托管部署即一个项目。如果你需要更多，有三个选项。在具有唯一 `COMPOSE_PROJECT_NAME` 值的同一服务器上运行单独的 Compose 堆栈，这会为每个实例的容器、卷和网络名称添加前缀。或者使用像 supabase-multitenant 这样的控制平面项目，它可以在你自己的基础设施上（通过 Coolify、Dokku 或直接在 VPS 上）提供面板，为每个项目生成隔离的 Supabase 堆栈。或者，坦率地说，如果你的团队重视云平台的每个项目仪表板，那么自托管节省的钱可能无法覆盖运维开销——这是一个完全合理的结论。

## FAQ

**自托管 Supabase 可以用于生产环境吗？**
可以——Docker 部署是官方支持的自托管路径，且需要真正的运维投入（备份、更新、监控）。Supabase CLI 启动的本地堆栈是另一回事：它没有针对生产环境进行加固，绝不能暴露给外部流量。

**自托管堆栈会收集遥测数据吗？**
不会。Docker 部署的 Supabase 不会发送遥测数据。只有 CLI 开发工具会收集使用数据，且有已记录的退出方法。

**我可以获得 PITR 或托管备份吗？**
不能，这些仍然是平台端的独有功能。你可以通过定期 `pg_dump` 来覆盖大部分需求；时间点恢复则需要你自己使用 Postgres 归档来构建。

**以后想离开有多难？**
你的数据保存在你拥有的普通 Postgres 中，因此 `pg_dump` 导出始终有效。Auth 用户和存储文件需要它们自己的导出步骤，但没有任何东西被锁定在专有格式中。

## 按场景选择配置

- **学习或评估**：6 美元/月的 1 GB High Frequency 套餐运行精简的四容器堆栈，甚至 2.50 美元的方案也足以探索原始 Postgres。
- **一个实际应用**：4 GB High Performance 满足最低要求；8 GB 是能让你安然入睡的配置。
- **多个项目**：8 GB 或更多，然后选择多堆栈或控制平面；只有当服务器规模变得庞大时才考虑裸金属。

首次启动前，确认三件事：密钥已生成、公开 URL 已在 `.env` 中设置、备份 cron 任务已存在。这三件事涵盖了绝大多数糟糕的第一周经历。当你准备好了，👉 [启动 Vultr 实例并部署堆栈](https://bit.ly/VulTR)。
