## 架构基础与高可用

### 1. 可用区（AZ）是什么？

可用区是同一 AWS 区域内、物理上隔离的数据中心集合。  
同一区域里有多个 AZ，把 ALB、EC2、RDS 跨 AZ 部署，可以降低单个机房故障带来的影响。

---

### 2. 水平扩展和垂直扩展有什么区别？

- **垂直扩展**：把单台机器配得更强，例如升配 CPU／内存
- **水平扩展**：增加机器数量，例如 Auto Scaling 加 EC2

云上更常用水平扩展，因为更灵活，也更适合搭配负载均衡。

---

### 3. 什么是无状态应用，为什么云上更推荐无状态设计？

无状态是指服务器本地不保存必须依赖的会话／业务状态，状态放在 Redis、RDS、S3 等外部存储。  
这样任意一台 EC2 挂了或扩容出新机器，请求都能被接手，更适合 ALB＋Auto Scaling。

---

### 4. RPO和RTO各是什么意思？

RPO 和 RTO 是衡量容灾能力的两个指标：RPO 关注**丢多少数据**，RTO 关注**停多长时间**。

- **RPO（Recovery Point Objective，恢复点目标）**：故障发生时，最多允许丢失多长时间的数据。它衡量的是数据丢失量。
  - RPO = 0 表示一条已提交的数据都不能丢。
  - RPO = 1 小时表示最多可以丢掉故障前 1 小时内写入的数据，通常对应每小时备份一次的方案。
- **RTO（Recovery Time Objective，恢复时间目标）**：从故障发生到服务恢复可用，最多允许花多长时间。它衡量的是服务中断时长。
  - RTO = 5 分钟表示 5 分钟内必须恢复对外服务。

---

### 5. 整体架构：请画出 Saafin／日通的大致 AWS 架构（从 Route 53 → CloudFront／ALB → EC2 → RDS／ElastiCache），并说明各层职责。

典型路径是：

**Route 53** 做域名解析 → **CloudFront** 分发静态资源／带签名的文件 → **ALB** 做入口负载均衡 → **EC2** 跑 Spring Boot 应用 → **RDS(MySQL)** 存业务数据 → **ElastiCache(Redis)** 做会话／缓存。

另外：**S3** 存证凭／请款单 PDF，**Lambda＋EventBridge（＋Step Functions）** 跑批处理，**SNS** 发通知，**Secrets Manager／KMS／IAM／CloudWatch／SG** 分别管密钥、加密、权限、监控和网络隔离。

一句话：入口层负责解析和分流，应用层处理业务，数据层做持久化和缓存，周边服务管安全、通知和运维。


## VPC 与网络安全

### 6. VPC里公有子网和私有子网通常怎么划分，ALB、EC2、RDS、ElastiCache 各放哪一层？

一般是：

- **公有子网**：放 ALB（需要被外网访问）
- **私有子网**：放 EC2、RDS、ElastiCache（不直接对公网开放）

ALB 在公有子网收流量，再转发到私有子网的 EC2；RDS 和 Redis 只允许 EC2／相关应用访问。

---

### 7. Security Group 和 NACL 有什么区别？

Security Group 是实例／网卡级防火墙，**有状态**，允许入站后回程自动放行。  
NACL 是子网级防火墙，**无状态**，入站和出站都要分别配置。  
实际中通常以 SG 为主做精细控制，NACL 做子网级兜底。

---

### 8. 私有子网里的 EC2 如何访问互联网下载补丁或调用外部 API？

常见两种方式：

- 走 **NAT Gateway**：私有子网路由指向 NAT，由 NAT 代为出网
- 访问 AWS 服务时用 **VPC Endpoint**：不走公网，更安全也更稳

下载补丁、调外部 API 多用 NAT；访问 S3／Secrets Manager 等 AWS 服务更推荐 Endpoint。

---

### 9. VPC Endpoint 是做什么用的？

让 VPC 内资源**不经过公网**访问 AWS 服务或私有服务。  
例如私有子网的 EC2／Lambda 通过 Endpoint 访问 S3、Secrets Manager，既更安全，也减少对 NAT 的依赖。

---

### 10. VPC Peering 和 Transit Gateway 有什么区别？

- **VPC Peering**：两个 VPC 之间点对点直连，**不可传递**（A 连 B、B 连 C，A 不能经过 B 访问 C），VPC 多了以后连接数会急剧增加
- **Transit Gateway**：中心辐射型的路由枢纽，所有 VPC 和本地网络都连到它，**可以传递路由**，适合多 VPC、多账号的大规模网络

VPC 少时用 Peering 简单省钱；VPC 多或需要集中管理时用 Transit Gateway。

---

### 11. AWS WAF 和 AWS Shield 有什么区别？

- **WAF**：7 层 Web 应用防火墙，挂在 CloudFront、ALB、API Gateway 上，按规则拦截 SQL 注入、XSS、恶意 IP，并可用基于速率的规则限流
- **Shield**：防 DDoS。**Shield Standard** 免费且默认开启，防护 3／4 层常见攻击；**Shield Advanced** 收费，提供更强的防护、攻击期间的费用保护和 AWS 专家团队支持

简单记：WAF 防"恶意请求内容"，Shield 防"流量洪水"。

---

### 12. VPC／SG：Security Group 你是怎么设计的？ALB、EC2、RDS、ElastiCache 之间分别开了哪些端口、为什么不能全开 0.0.0.0/0？

原则是「**只开必要端口，只允许上游访问**」：

- **ALB SG**：入站允许 80／443（来自互联网或 CloudFront）
- **EC2 SG**：入站只允许来自 ALB SG 的 8080（或应用端口），不直接对公网开放
- **RDS SG**：入站只允许来自 EC2 SG 的 3306
- **ElastiCache SG**：入站只允许来自 EC2 SG 的 6379

出站一般先放开，再按需要收紧。绝不把 RDS／Redis 的 0.0.0.0/0 打开。SG 是有状态的，回程流量不用单独配。


## 负载均衡、Route 53 与 Auto Scaling

### 13. ALB 和 NLB 有什么主要区别？

- **ALB**：7 层，按 HTTP／HTTPS 路径、Host 转发，适合 Web／API
- **NLB**：4 层，按 TCP／UDP 转发，延迟更低，适合高并发、极致性能或非 HTTP 协议

你简历里的 Web 系统，用 ALB 更合适。

---

### 14. 什么是粘性会话？粘性会话的缺点是什么？

粘性会话（Sticky Session，也叫会话亲和性）是负载均衡器的一种功能：把同一个用户的请求固定转发到同一台后端服务器上。

为什么需要它： 如果 Session 存在 EC2 本机内存里，用户第一次登录落在 A 机器上，下一次请求被 ALB 分到 B 机器，B 上没有这个用户的 Session，用户就会被要求重新登录。粘性会话让这个用户一直访问 A，问题就避开了。

- 缺点（也就是 Redis 方案更好的原因）
    - 负载不均衡：用户被绑定在某台机器上，可能导致一台很忙、另一台很闲。
    - 单点丢失：那台 EC2 一旦宕机或缩容，上面所有用户的 Session 都会丢失，需要重新登录。
    - 扩缩容效果差：新加的机器分不到老用户，扩容后不能马上分担压力。

把 Session 放到 Redis（如 ElastiCache）里，所有 EC2 共享同一份 Session，服务器本身就变成无状态的。请求落到任何一台机器都能正常处理，上面这些问题也就不存在了

---

### 15. 多台 EC2 通过 ALB 对外提供服务时，为什么用 Redis 存 Session 通常比开启粘性会话更好？

粘性会话会把用户固定到某台 EC2，扩缩容或某台宕机时容易丢会话、也不均匀。  
Session 放 Redis 后，任意 EC2 都能读到同一份登录态，更适合 ALB＋多机＋Auto Scaling。

---

### 16. ALB 健康检查和 Route 53 健康检查，检查对象有什么不同？

- **ALB 健康检查**：检查 Target Group 里的 EC2／目标是否健康，决定流量打到哪台机器
- **Route 53 健康检查**：检查整个端点／域名是否可用，决定 DNS 是否切到备用记录

一个管“集群内部哪台活着”，一个管“对外域名指到哪套环境”。

---

### 17. Route 53 做故障切换时，主备记录一般怎么配置？

常用 **Failover 路由**：

- 主记录（Primary）：指向正常环境，例如主 ALB
- 副记录（Secondary）：指向备用环境

并给主记录配健康检查。主记录不健康时，Route 53 把流量切到副记录。

---

### 18. Route 53 有哪些路由策略？Alias 记录和 CNAME 有什么区别？

路由策略：
- **简单路由**：一个记录对应一组值
- **加权路由**：按比例分流，适合灰度发布
- **延迟路由**：把用户导向延迟最低的区域
- **地理位置路由**：按用户所在国家或地区返回不同结果
- **故障切换路由**：主记录不健康时切到备用记录

Alias 和 CNAME 的区别：
- **Alias** 是 Route 53 特有的，可以指向 ALB、CloudFront、S3 等 AWS 资源，**可以用在根域名（zone apex）上**，指向 AWS 资源的查询不收费
- **CNAME** 不能用在根域名上，查询收费

---

### 19. Auto Scaling 缩容时，如何避免正在处理的请求被直接打断？

依赖 **deregistration delay（连接排空）**：

先把实例从 Target Group 摘掉，不再接收新请求，等存量请求处理完或超时，再终止实例。  
这样缩容时正在处理的请求不容易被直接切断。

---

### 20. ALB＋Auto Scaling：日通的冗余构成是怎么做的？扩缩容触发条件是什么？健康检查失败后会发生什么？

ALB 后面挂多个 EC2，用 Target Group 做健康检查（例如访问 `/health`）。某台不健康会被摘掉，流量只打到健康实例。

Auto Scaling 按 CPU、请求数或自定义指标扩缩：例如 CPU 持续超过 70% 加机器，低于 30% 减机器。扩容时新实例自动注册到 Target Group；缩容前先从 ALB 摘除，避免打断进行中的请求。

目的是：**单点故障不影响服务，流量高峰能自动扛住**。

---

### 21. 通过Route53把域名绑定到ALB上面，具体怎么操作？

1. **准备托管区域（Hosted Zone）**
   - 如果域名是在 Route 53 注册的，系统已经自动建好了托管区域。
   - 如果域名在其他注册商那里，先在 Route 53 创建一个公有托管区域，再到注册商后台把域名的 NS 改成 Route 53 分配的 4 个 NS 地址。

2. **创建记录**
   - 进入 Route 53 → Hosted zones → 选中你的域名 → Create record。
   - Record name 填子域名（比如 `www`）；要绑定根域名就留空。
   - Record type 选 `A`。
   - 打开 Alias，Route traffic to 选 "Alias to Application and Classic Load Balancer"，再选 ALB 所在的区域和对应的 ALB。
   - 如果 ALB 是双栈（同时支持 IPv4 和 IPv6），再建一条同样设置的 `AAAA` 记录。

3. **（推荐）配置 HTTPS**
   - 在 ACM 里给这个域名申请证书，验证方式选 DNS 验证。点 "Create records in Route 53" 会自动写入验证记录。
   - 证书签发后，给 ALB 加一个 443 监听器并关联证书。
   - 把 80 端口的监听器设成重定向到 443。

4. **检查**
   - 确认 ALB 的安全组放行了 80 和 443 端口。
   - 用 `dig www.example.com` 或 `nslookup` 查看解析结果，然后用浏览器访问测试。


## EC2 与存储

### 22. EC2 的按需实例、预留实例（RI）、Savings Plans、Spot 实例有什么区别？

- **按需实例**：按秒／小时计费，无需承诺，最灵活，单价最高
- **预留实例（RI）**：承诺 1 年或 3 年使用特定实例类型，换取较大折扣
- **Savings Plans**：承诺 1 年或 3 年每小时的消费金额，折扣类似 RI，但实例类型、区域等更灵活
- **Spot 实例**：使用 AWS 的闲置容量，最高可便宜约 90%，但可能被回收（提前 2 分钟通知）

常驻的基础负载用 Savings Plans／RI，突发部分用按需，可中断的批处理、CI 用 Spot。

---

### 23. EBS、EFS、Instance Store 有什么区别？

- **EBS(Elastic Block Store)**：块存储，挂载到单台 EC2，限定在一个 AZ 内，实例停止后数据仍在，可做快照
- **EFS(Elastic File System)**：NFS(Network File System) 文件存储，可跨 AZ 被多台 EC2 同时挂载，容量自动伸缩
- **Instance Store**：物理机本地盘，性能高，但实例停止或终止后数据丢失，只适合放临时数据、缓存

---

### 24. EBS和EFS各有什么利弊？

**EBS（弹性块存储）**
- **优点**：
  - 延迟低，IOPS 高，性能稳定，适合跑数据库和系统盘
  - 每 GB 单价比 EFS 便宜
  - 可以做快照，用来备份和恢复
- **缺点**：
  - 一般只能挂在一台 EC2 上，不能共享
  - 只存在于一个可用区（AZ），这个 AZ 出故障就访问不了
  - 容量要事先指定，不够了得手动扩容；申请了没用完的部分也要付费

**EFS（弹性文件系统）**

- **优点**：
  - 多台 EC2 可以同时挂载，共享同一份文件
  - 数据跨多个可用区保存，可用性高
  - 容量自动伸缩，不用规划大小，按实际用量计费
- **缺点**：
  - 走网络访问，延迟比 EBS 高；处理大量小文件时性能较差
  - 每 GB 单价比 EBS 贵好几倍
  - 不能做系统盘，也不适合跑数据库
  - 基于 NFS 协议，主要给 Linux 用；Windows 一般用 FSx


## 发布部署

### 25. 在 Auto Scaling 环境中做应用发版，常见的滚动更新流程是怎样的？

常见做法：

1. 启动带新版本的新实例
2. 新实例通过健康检查后挂到 ALB
3. 逐步把旧实例从 Target Group 摘掉并终止
4. 全程保持一部分健康实例继续服务

这样发版时服务不中断，也比一次性全切更稳。

---

### 26. 蓝绿部署和滚动更新有什么区别？

- **蓝绿部署**：先准备一整套新环境，验证通过后一次性切流量
- **滚动更新**：在同一套环境里逐步替换旧实例为新实例

蓝绿回滚更快、隔离更干净；滚动更新资源占用通常更少，但过程中会短暂新旧版本并存。


## 容器与 IaC

### 27. ECS 的 EC2 启动类型和 Fargate 启动类型有什么区别？

- **EC2 启动类型**：容器跑在自己管理的 EC2 集群上，需要自己负责扩容、打补丁；可以选机型（如 GPU），大规模稳定负载时成本更低
- **Fargate**：无服务器容器，只需定义每个任务的 CPU 和内存，不用管服务器，按任务资源计费

团队运维人力少、负载波动大时选 Fargate；需要特殊机型或追求极致成本时选 EC2。

---

### 28. 什么是 IaC？CloudFormation 有哪些关键概念？

IaC（Infrastructure as Code，基础设施即代码）是用代码定义 VPC、EC2、RDS 等资源，做到可版本管理、可评审、可重复部署，避免在控制台手工操作导致环境不一致。

CloudFormation 的关键概念：
- **Template**：用 YAML／JSON 写的资源定义
- **Stack**：按模板创建出来的一组资源，作为一个整体创建、更新、删除
- **Change Set**：更新前预览哪些资源会被修改或替换
- **Drift Detection**：检测实际资源是否被人手工改过
- 更新失败时自动回滚

CDK 可以用 TypeScript、Java 等语言编写，最终生成 CloudFormation 模板；Terraform 是第三方工具，可以跨云使用。


## RDS

### 29. RDS 的 Multi-AZ 和 Read Replica 有什么区别？

- **Multi-AZ**：高可用，同步复制到备库，主库故障可自动切换，备库一般不承担读流量
- **Read Replica**：读扩展，异步复制，可分担查询压力，不能当成自动故障切换的高可用方案

简单记：Multi-AZ 保活，Read Replica 抗读。

---

### 30. RDS マルチAZ：主库故障时切换流程是怎样的？自动备份和快照有什么区别？RPO／RTO 大概怎么考虑？

マルチAZ 是同步复制到另一个可用区的备库。主库故障时，RDS 自动故障切换，DNS 端点不变，应用通常只需短暂重连，不用改连接串。

- **自动备份**：按保留期做连续备份＋事务日志，可做时间点恢复（PITR）
- **快照**：某时刻的手动／自动完整备份，适合发版前、大变更前留档

常见目标：RPO 接近 0（同步复制），RTO 大约几分钟（故障切换时间）。面试时可以说：业务可接受短暂中断，但不能丢已提交的请款／审批数据。


## ElastiCache（Redis）

### 31. Redis 里设置 TTL 的主要作用是什么？

让 key 到期自动删除，避免缓存长期占满内存，也能降低读到过期数据的风险。  
会话、主数据缓存都常设 TTL，作为缓存失效的兜底机制。

---

### 32. Redis 内存满了之后会发生什么？

取决于淘汰策略。常见是按 LRU 等规则删旧 key；如果策略不允许淘汰且内存满了，写操作会失败。  
所以要设合理 `maxmemory-policy`，并监控内存使用率，避免会话或缓存雪崩式失效。

---

### 33. ElastiCache（Redis）：会话管理和主数据缓存分别怎么用？缓存失效（cache invalidation）怎么处理？

两种典型用法：

- **会话管理**：登录态放 Redis，多台 EC2 可共享会话，方便水平扩展
- **主数据缓存**：客户／商品等读多写少的数据缓存起来，减轻 RDS 压力，加快搜索

缓存更新策略常见两种：
- 更新 DB 后删除对应 key（下次读时回源）
- 更新 DB 后同时写缓存

主数据变更后一定要失效旧缓存，否则会出现「库已更新、页面还是旧数据」。还要设 TTL，防止冷数据长期占内存。


## S3 与 CloudFront

### 34. S3中的签名URL是什么？

签名 URL（Signed URL）是一个带有签名和过期时间的临时访问链接。拿到这个链接的人，在有效期内不需要登录、也不需要 AWS 凭证，就能访问一个私有文件。

- 两种签名 URL
    - S3 预签名 URL（Presigned URL）：用 IAM 凭证签名，直接访问 S3。
    - CloudFront 签名 URL：用 CloudFront 的密钥对签名，通过 CloudFront 访问；S3 Bucket 本身保持私有，只允许 CloudFront 读取（通过 OAC 配置）。

---

### 35. 从 S3 误删文件后，怎样才能恢复？

前提是提前开了 **版本控制（Versioning）**，必要时再加 MFA Delete／回收机制。  
开启版本控制后，删除往往只是打删除标记，可把旧版本恢复回来；如果没开版本控制，误删通常很难恢复。

---

### 36. 什么是CloudFront的OAC？

OAC（Origin Access Control，源访问控制）是 CloudFront 的一种配置。它的作用是让 S3 Bucket 保持私有，只允许指定的 CloudFront 分发读取，用户无法绕过 CloudFront 直接访问 S3。

- 工作原理
    - CloudFront 向 S3 取文件时，会用 SigV4 对请求签名，表明请求来自哪个 CloudFront 分发。S3 根据 Bucket 策略检查这个来源。不是指定分发发来的请求，一律拒绝。

---

### 37. CloudFront 的签名 URL 和签名 Cookie，分别适合什么使用场景？

- **签名 URL**：适合单文件、临时下载，例如证凭、请款单 PDF
- **签名 Cookie**：适合一次授权后连续访问一批资源，例如整站静态资源、一组页面／文件

你简历里的证凭／PDF 下载，更适合签名 URL。

---

### 38. S3 的 Bucket Policy 和 IAM Policy 同时存在时，最终如何判断是否允许访问？

AWS 权限是综合评估的，核心规则是：

- 显式 **Deny** 优先
- 没有 Deny 时，要有至少一方 **Allow** 才会放行
- 两边都不 Allow，就是拒绝

所以既要配好 IAM，也要检查 Bucket Policy，避免互相打架。

---

### 39. S3 有哪些存储类型？怎样用生命周期规则降低成本？

- **Standard**：频繁访问
- **Intelligent-Tiering**：访问模式不确定时，自动在不同访问层之间移动
- **Standard-IA／One Zone-IA**：不常访问但需要快速读取；有最短存储 30 天的限制，读取要收取回费；One Zone-IA 只存一个 AZ，更便宜
- **Glacier Instant Retrieval／Flexible Retrieval／Deep Archive**：归档用，越往后越便宜，取回越慢（Deep Archive 取回需要数小时）

生命周期规则可按前缀或标签设置，例如 30 天转 IA、180 天转 Glacier、7 年后删除。证凭类需要长期留存的文件很适合这种方式。

---

### 40. CloudFront 的缓存 TTL 和失效（Invalidation）是怎么回事？发版后用户看到旧文件怎么办？

- **TTL(Time To Live)**：决定对象在边缘节点缓存多久，可以由源站的 `Cache-Control` 头或 CloudFront 的缓存策略控制
- **Invalidation**：手动让指定路径的缓存失效，下次请求时回源拉取；每月有一定免费路径数，超出后收费

发版的推荐做法是给静态文件名加版本号或哈希（如 `app.3f2a.js`），新版本就是新 URL，不依赖失效；`index.html` 设置较短 TTL，必要时再做 Invalidation。

---

### 41. S3＋CloudFront 签名 URL：请款单 PDF／证凭文件为什么用签名 URL？过期时间怎么设？和公开 Bucket 有什么区别？

证凭、请款单 PDF 属于敏感文件，Bucket 保持私有，不设公共读。用户下载时由后端生成**限时签名 URL**（例如 5～15 分钟），过期后链接失效。

CloudFront 放在前面，可加速分发，并用 Origin Access 控制，只允许 CloudFront 访问 S3，避免绕过 CDN 直连 Bucket。

和公开 Bucket 的区别：公开 Bucket 任何人有链接就能长期访问；签名 URL 是「临时、可追溯、可失效」的访问方式。

---

### 42. 如何使用CloudFront静态分发S3的内容

1. **创建 S3 桶**
   - 新建一个桶，保持“阻止所有公有访问”为开启。
   - 不需要开启“静态网站托管”。
   - 上传静态文件，比如 `index.html`、CSS、JS 和图片。

2. **申请 ACM 证书**（要用自定义域名时才需要）
   - 证书必须在 **us-east-1（弗吉尼亚北部）** 区域申请，CloudFront 只认这个区域的证书。
   - 验证方式选 DNS 验证，点 "Create records in Route 53" 自动完成验证。

3. **创建 CloudFront 分发**
   - Origin（源）选刚才的 S3 桶（选 REST 端点，不要选网站端点）。
   - Origin access 选 "Origin access control settings"，新建一个 OAC。
   - Viewer protocol policy 选 "Redirect HTTP to HTTPS"。
   - 缓存策略可以先用 `CachingOptimized`。
   - Alternate domain name 填你的域名（比如 `static.example.com`），再选上 ACM 证书。
   - Default root object 填 `index.html`。

4. **更新 S3 桶策略**
   - 分发创建好后，控制台会提示复制一段桶策略。把它粘贴到 S3 的桶策略里，这样只有这个分发能读取桶里的对象。

5. **配置 Route 53**
   - 在托管区域里建一条 `A` 记录（需要 IPv6 的话再建 `AAAA`），打开 Alias，目标选 "Alias to CloudFront distribution"，再选你的分发。

6. **检查**
   - 等分发状态变成 Deployed（通常几分钟），然后访问域名测试。

---

### 43. S3を利用した証憑ファイルの保存設計、KMSによるデータ暗号化具体做了什么？

1. **存储结构**：按“公司 / 年 / 月 / 证凭种类”划分前缀（比如 `invoice/2026/10/xxx.pdf`）。日期、金额、交易对象这些元数据存在数据库里，方便检索（这也满足日本《电子账簿保存法》的检索要求）。
2. **访问控制**：开启“阻止所有公有访问”，用 IAM 角色控制最小权限。用户上传和下载通过预签名 URL（Presigned URL）进行，并强制只允许 HTTPS 访问。
3. **加密**：用 SSE-KMS 加密，密钥是客户托管密钥（CMK）。在密钥策略里限制谁能用这个密钥，开启自动轮换，再启用 S3 Bucket Key 降低 KMS 调用费用。
4. **防篡改和保存期限**：开启版本控制和 Object Lock（合规模式），保存期限按法定年限设置，比如 7 到 10 年。配合生命周期规则，把旧文件转到 Glacier 降低成本。
5. **审计**：用 CloudTrail 记录 S3 和 KMS 的操作日志，必要时再开启 S3 访问日志。

---

### 44. 为什么用 KMS 而不用 SSE-S3？

因为可以自己控制密钥的使用权限，还能在 CloudTrail 里查到每次密钥是谁用的。


## Lambda、事件驱动与批处理

### 45. EventBridge、SQS、SNS 三者分别适合什么场景？

- **EventBridge**：定时任务、事件路由，适合“到点触发／按事件分发”
- **SQS**：队列缓冲，适合削峰、异步解耦、保证消息被消费
- **SNS**：扇出通知，适合一条消息同时发给短信、邮件、多个下游

简历里的夜间批处理偏 EventBridge；审批通知偏 SNS；若要可靠排队处理可用 SQS。

---

### 46. SQS 的标准队列和 FIFO 队列有什么区别？

- **标准队列**：吞吐量几乎无限，**至少投递一次**（可能重复），顺序尽力保证但不严格
- **FIFO 队列**：同一消息组（Message Group）内**严格有序**，支持去重，实现**恰好处理一次**，但吞吐量有上限

对顺序有要求的场景（如同一张请款单的状态变更）用 FIFO；其他场景用标准队列，同时让消费端做到幂等。

---

### 47. SQS 的可见性超时（Visibility Timeout）和死信队列（DLQ）是什么？

- **可见性超时**：消息被消费者取走后，在这段时间内对其他消费者不可见；如果超时前没有删除消息，它会重新出现在队列里被再次消费。默认 30 秒，应大于消息的处理时间
- **死信队列**：一条消息被接收的次数超过 `maxReceiveCount` 后，转进 DLQ，避免"毒消息"无限重试

实践中要对 DLQ 的消息数设置告警，排查原因后再重新投递（redrive）。用 Lambda 消费 SQS 时，AWS 建议可见性超时至少设为函数超时的 6 倍。

---

### 48. Lambda 需要访问 VPC 内的 RDS 或 ElastiCache 时，为什么常常要把 Lambda 也放进 VPC？

因为 RDS、ElastiCache 通常在私有子网，只允许 VPC 内访问。  
Lambda 默认不在你的 VPC 里，所以要访问这些资源时，需要把 Lambda 配到同一 VPC／可连通的子网。

---

### 49. 什么是 Lambda 冷启动？怎样缓解？

冷启动是指 Lambda 没有可复用的执行环境时，需要新建环境、加载代码、初始化运行时，导致首次请求变慢。Java 这类运行时尤其明显。

缓解方法：
- 开启 **Provisioned Concurrency(预置并发)**，预先准备好执行环境
- Java 可以开启 **SnapStart(冷启动快照)**
- 减小部署包、减少依赖，精简初始化逻辑
- 把 DB 连接、SDK 客户端放在 handler 外面初始化，以便后续调用复用

夜间批处理对冷启动不敏感；面向用户的同步接口才需要重点优化。

---

### 50. Lambda 有哪些主要限制？超过 15 分钟的任务怎么处理？

主要限制：
- 单次执行最长 **15 分钟**
- 内存最大约 10 GB，CPU 按内存比例分配
- `/tmp` 临时空间有上限
- 同步调用的请求／响应负载约 6 MB
- 账号级的并发配额

超过 15 分钟的任务：
- 用 **Step Functions** 拆成多个短步骤，或用 Map 状态并行分片处理
- 改用 **ECS Fargate** 或 **AWS Batch** 跑长时间任务

---

### 51. Step Functions 的 Standard 工作流和 Express 工作流有什么区别？

- **Standard**：适合长时间、关键流程，执行历史保留更久，按状态转换计费
- **Express**：适合高吞吐、短时间流程，成本模型不同，审计／历史能力较弱

夜间主数据导入、要清晰排查和重试的流程，通常更适合 Standard。

---

### 52. Lambda 并发突然升高时，可能对 RDS 造成什么影响？

每个并发 Lambda 都可能占数据库连接，容易把 RDS **连接数打满**，导致新连接失败、业务超时。  
所以要控制并发、用连接池／代理（如 RDS Proxy），或把压力大的写入改成队列化。

---

### 53. 用 SNS 发 SMS 审批通知时，发送失败后你怎么感知到？

给 SNS 配 **失败反馈／订阅状态监控**，并结合 CloudWatch 看发送失败指标和日志。  
也可以同时落库发送结果；失败时再告警，或自动改发邮件做降级。

配置的关键是打开 SNS 的 SMS 投递状态日志（Delivery Status Logging）。打开后，每条短信的投递结果都会写进 CloudWatch Logs；再基于失败指标配一个告警，就能在发送失败时收到通知。

选择 SNS 自带的指标 NumberOfNotificationsFailed。设置条件，例如 5 分钟内失败数大于 0 就告警。告警动作通知到另一个 SNS 主题（发邮件），或者触发 Lambda 做降级处理，比如改发邮件。

---

### 54. Step Functions 某一步已经写入部分数据后失败并重试，怎样保证不会重复插入？

关键是让每一步 **幂等**：

- 用业务唯一键（如主数据 ID、文件名＋批次号）做去重
- 写入前先查是否已处理
- 或用「先写处理状态，再写业务数据」的方式

这样重试时发现已经处理过，就跳过，不会插第二遍。

---

### 55. 批处理告警里，怎样区分「业务校验失败」和「基础设施失败」？

看失败类型和日志内容：

- **业务校验失败**：文件格式不对、必填字段缺失、主数据重复等，通常是可预期的业务错误码
- **基础设施失败**：超时、连不上 RDS、权限不足、Lambda OOM、Step Functions 系统错误等

---

### 56. Lambda＋EventBridge＋Step Functions：夜间主数据导入批处理的工作流怎么拆？某一步失败后如何重试／告警？

日通的主数据导入可以拆成：校验文件 → 清洗转换 → 写入 RDS → 刷新缓存 → 发送结果通知。

- **EventBridge**：设定夜间 cron，到点触发
- **Step Functions**：编排上述步骤，控制顺序、重试、分支
- **Lambda**：执行每个具体步骤

某步失败时：按策略重试（例如最多 3 次，指数退避）；仍失败则标记失败状态，并通过 SNS／CloudWatch 告警，避免静默失败。成功／失败都要有日志可查。


## IAM 权限管理

### 57. IAM 里的 User、Role、Policy 分别是什么？

- **User**：给人用的长期身份
- **Role**：可被 EC2、Lambda、其他账号临时扮演的身份
- **Policy**：权限说明书，挂到 User／Role 上决定能做什么

云上应用更推荐用 Role，而不是把长期密钥写进服务器。

---

### 58. EC2 上的应用程序如何获取实例角色的临时凭证？

通过实例元数据服务获取，现代推荐 **IMDSv2**：

应用向元数据地址要临时 Access Key、Secret Key、Token，再拿这些凭证访问 S3、Secrets Manager 等。  
凭证会自动轮换，不用把长期密钥写进代码。

---

### 59. IAM：为什么要把 EC2 和 Lambda 的 Role 分开？举一个「最小权限」的具体例子。

EC2 和 Lambda 职责不同，所以 Role 分开：

- **EC2 Role**：读 Secrets、读写业务 S3、写 CloudWatch Logs、访问 ElastiCache／RDS 所需权限
- **Lambda Role**：只给批处理需要的权限，例如读指定 S3 前缀、写 RDS、发 SNS、写日志

例子：批处理 Lambda 只允许 `s3:GetObject` 到 `s3://xxx/master-import/*`，而不是整个账号的 `s3:*`。人用 IAM 用户／角色做管理，应用用 Role，避免把长期密钥打进服务器。

---

### 60. IAMによるアクセス権限設計具体是怎么设计的？

- **原则**：只给完成工作所需的最小权限。
- **按角色划分**：开发、运维、只读等角色分别建 IAM 组或角色，只允许各自需要的操作。
- **不用长期密钥**：EC2、Lambda 这类 AWS 资源通过 IAM 角色获取权限，代码里不写访问密钥（Access Key）。
- **安全和审计**：日常不使用 root 用户，所有用户强制开启 MFA（多因素认证），操作记录用 CloudTrail 保存。


## 加密与密钥管理

### 61. 静态加密（at rest）和传输中加密（in transit）有什么区别？

- **静态加密**：数据落盘时加密，如 S3、RDS 用 KMS 加密存储
- **传输中加密**：数据在网络传输时加密，如 HTTPS、TLS、数据库 SSL 连接

两者要一起做，不能只加密存储却明文传。

---

### 62. ACM 是什么？HTTPS 证书一般配置在哪里？

ACM（AWS Certificate Manager）可以免费申请公有 SSL／TLS 证书，并自动续期。

证书一般配置在 **ALB 或 CloudFront** 上，由它们做 HTTPS 终止，后端 EC2 不用管理证书。注意：给 CloudFront 使用的证书必须申请在 **us-east-1（弗吉尼亚北部）区域**。合规要求高时，ALB 到 EC2 之间也可以再走 HTTPS。

---

### 63. S3 服务端加密里，SSE-S3、SSE-KMS、SSE-C 有什么区别？

- **SSE-S3**：S3 托管密钥，配置最简单
- **SSE-KMS**：用 KMS 密钥，可细控权限和审计
- **SSE-C**：客户自己提供密钥，AWS 不帮你保管密钥

简历里强调 CMK，对应更接近 **SSE-KMS**。

---

### 64. Systems Manager Parameter Store 和 Secrets Manager 有什么区别？

- **Parameter Store**：适合普通配置项，如开关、非敏感参数；成本低
- **Secrets Manager**：适合密码、API Key 等机密，支持自动轮换

数据库密码、第三方密钥一般放 Secrets Manager；普通应用配置可用 Parameter Store。

---

### 65. KMS 的信封加密（envelope encryption）是什么意思？

不是每次都用主密钥直接加密大文件，而是：

1. 生成一把短暂的数据密钥（data key）
2. 用数据密钥加密真正的数据
3. 再用 KMS 主密钥加密这把数据密钥

这样性能更好，也便于密钥管理；你看到的 S3／RDS 很多加密底层都是这个思路。

---

### 66. Secrets Manager 密码轮换过程中出现 AWSCURRENT 和 AWSPENDING 两个版本时，应用应该读哪一个？

应用应读 **AWSCURRENT**。  
`AWSPENDING` 是轮换过程中的临时版本，只有轮换验证通过后才会变成当前版本。业务应用不要去读 `AWSPENDING`。

---

### 67. Secrets Manager＋KMS：DB 密码自动轮换具体怎么做？CMK 和 AWS 托管密钥有什么区别？谁有解密权限？

DB 密码、API Key 不写在代码或配置文件里，而是放在 **Secrets Manager**。应用通过 IAM 角色在运行时读取。

自动轮换大致是：Secrets Manager 定时触发 Lambda → 在 RDS 创建新密码 → 验证可用 → 标记为当前版本。应用下次读取会拿到新密码。

**KMS**：
- AWS 托管密钥：省事，权限随服务
- **CMK（客户托管密钥）**：可自管轮换、审计、授权，适合合规要求更高的数据（证凭、请款单、RDS 存储加密）

谁能解密，取决于 KMS Key Policy＋IAM：只有需要读密文的角色（如 EC2／Lambda 角色）才有 `kms:Decrypt`。


## 监控与审计

### 68. CloudWatch 的 Metric 和 Log 有什么区别？

- **Metric**：数值指标，如 CPU、5xx、延迟，适合做告警和看板
- **Log**：文本日志，适合排查具体报错原因和调用链细节

通常先看 Metric 发现问题，再靠 Log 定位根因。

---

### 69. CloudTrail 和 CloudWatch 有什么区别？

- **CloudTrail**：记录谁在什么时候对 AWS 做了什么 API 操作，偏审计
- **CloudWatch**：看指标、日志、告警，偏监控和运维排障

简单记：CloudTrail 查“谁做了什么”，CloudWatch 查“系统现在正不正常”。

---

### 70. CloudWatch 运维：你设过哪些告警？收到告警后的一次障害对应流程是怎样的？

常见告警：
- EC2／ALB：5xx、延迟、CPU
- RDS：CPU、连接数、存储空间、复制延迟
- 批处理：Lambda 错误数、Step Functions 失败、EventBridge 未触发
- 业务：队列堆积或自定义失败指标

一次障害对应流程：
1. 收到告警，先看 CloudWatch 看板／日志，确认影响范围
2. 判断是基础设施问题还是应用／批处理问题
3. 紧急处理（切流量、扩容、重跑批处理、回滚）
4. 恢复后复核数据是否完整
5. 记录原因和再发防止对策

面试时用一句收尾：**先止损，再定位，最后复盘**。

---

### 71. CloudWatchによるシステム監視・アラート設定具体做了什么？

- **监控对象**：EC2 和 RDS 的 CPU、内存、磁盘，ALB 的 5xx 错误数和响应时间。
- **报警方式**：指标超过阈值就触发告警。应用日志集中存到 CloudWatch Logs，用指标过滤器（Metric Filter）检测到 "ERROR" 等关键字时也会告警。
- **通知方式**：通过 SNS 发邮件，让故障能尽早被发现和处理。
