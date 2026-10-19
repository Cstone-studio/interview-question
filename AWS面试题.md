1. **整体架构**：请画出 Saafin／日通的大致 AWS 架构（从 Route 53 → CloudFront／ALB → EC2 → RDS／ElastiCache），并说明各层职责。
2. **VPC／SG**：Security Group 你是怎么设计的？ALB、EC2、RDS、ElastiCache 之间分别开了哪些端口、为什么不能全开 0.0.0.0/0？
3. **ALB＋Auto Scaling**：日通的冗余构成是怎么做的？扩缩容触发条件是什么？健康检查失败后会发生什么？
4. **RDS マルチAZ**：主库故障时切换流程是怎样的？自动备份和快照有什么区别？RPO／RTO 大概怎么考虑？
5. **S3＋CloudFront 签名 URL**：请款单 PDF／证凭文件为什么用签名 URL？过期时间怎么设？和公开 Bucket 有什么区别？
6. **Lambda＋EventBridge＋Step Functions**：夜间主数据导入批处理的工作流怎么拆？某一步失败后如何重试／告警？
7. **ElastiCache（Redis）**：会话管理和主数据缓存分别怎么用？缓存失效（cache invalidation）怎么处理？
8. **Secrets Manager＋KMS**：DB 密码自动轮换具体怎么做？CMK 和 AWS 托管密钥有什么区别？谁有解密权限？
9. **IAM**：为什么要把 EC2 和 Lambda 的 Role 分开？举一个「最小权限」的具体例子。
10. **CloudWatch 运维**：你设过哪些告警？收到告警后的一次障害对应流程是怎样的？
11. RPO和RTO各是什么意思？
12. S3中的签名URL是什么？
13. 什么是CloudFront的OAC？
14. VPC里公有子网和私有子网通常怎么划分，ALB、EC2、RDS、ElastiCache 各放哪一层？
15. Route 53 做故障切换时，主备记录一般怎么配置？
16. CloudFront 的签名 URL 和签名 Cookie，分别适合什么使用场景？
17. S3 服务端加密里，SSE-S3、SSE-KMS、SSE-C 有什么区别？
18. Lambda 需要访问 VPC 内的 RDS 或 ElastiCache 时，为什么常常要把 Lambda 也放进 VPC？
19. Step Functions 某一步已经写入部分数据后失败并重试，怎样保证不会重复插入？
20. 用 SNS 发 SMS 审批通知时，发送失败后你怎么感知到？
21. 多台 EC2 通过 ALB 对外提供服务时，为什么用 Redis 存 Session 通常比开启粘性会话更好？
22. Secrets Manager 密码轮换过程中出现 AWSCURRENT 和 AWSPENDING 两个版本时，应用应该读哪一个？
23. 批处理告警里，怎样区分「业务校验失败」和「基础设施失败」？
24. 什么是粘性会话？粘性会话的缺点是什么？
25. Security Group 和 NACL 有什么区别？
26. 私有子网里的 EC2 如何访问互联网下载补丁或调用外部 API？
27. ALB 和 NLB 有什么主要区别？
28. RDS 的 Multi-AZ 和 Read Replica 有什么区别？
29. IAM 里的 User、Role、Policy 分别是什么？
30. 静态加密（at rest）和传输中加密（in transit）有什么区别？
31. Auto Scaling 缩容时，如何避免正在处理的请求被直接打断？
32. S3 的 Bucket Policy 和 IAM Policy 同时存在时，最终如何判断是否允许访问？
33. EventBridge、SQS、SNS 三者分别适合什么场景？
34. 什么是无状态应用，为什么云上更推荐无状态设计？
35. VPC Endpoint 是做什么用的？
36. KMS 的信封加密（envelope encryption）是什么意思？
37. CloudWatch 的 Metric 和 Log 有什么区别？
38. Lambda 并发突然升高时，可能对 RDS 造成什么影响？
39. Redis 内存满了之后会发生什么？
40. ALB 健康检查和 Route 53 健康检查，检查对象有什么不同？
41. EC2 上的应用程序如何获取实例角色的临时凭证？
42. Step Functions 的 Standard 工作流和 Express 工作流有什么区别？
43. 从 S3 误删文件后，怎样才能恢复？
44. 在 Auto Scaling 环境中做应用发版，常见的滚动更新流程是怎样的？
45. 可用区（AZ）是什么？
46. 水平扩展和垂直扩展有什么区别？
47. Systems Manager Parameter Store 和 Secrets Manager 有什么区别？
48. CloudTrail 和 CloudWatch 有什么区别？
49. 蓝绿部署和滚动更新有什么区别？
50. Redis 里设置 TTL 的主要作用是什么？
