static_configs:
    - targets: ['localhost:9093']

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
  
  - job_name: 'spring-boot'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['localhost:8080']
```

**Grafana告警配置**：
```json
{
  "alert": {
    "name": "High CPU Usage",
    "conditions": [
      {
        "type": "query",
        "query": {
          "params": ["A", "5m", "now"]
        },
        "reducer": {
          "type": "avg",
          "params": []
        },
        "evaluator": {
          "type": "gt",
          "params": [80]
        }
      }
    ],
    "frequency": "1m",
    "handler": 1,
    "notifications": [
      {
        "uid": "telegram-notification"
      }
    ]
  }
}
```

## 3. 总结

### 3.1 性能指标总结

| 指标 | 用途 | 关键考虑点 |
|------|------|------------|
| OPS | 系统整体处理能力 | 硬件资源、系统架构、代码优化 |
| QPS | 查询处理能力 | 缓存策略、数据库优化、并发控制 |
| TPS | 事务处理能力 | 数据一致性、锁机制、事务隔离级别 |

### 3.2 高可用架构总结

| 层级 | 关键技术 | 实现方案 |
|------|---------|----------|
| 应用层 | 负载均衡、服务集群 | Nginx、Kubernetes |
| 数据层 | 主从复制、分片集群 | MySQL InnoDB Cluster、Redis Cluster |
| 存储层 | 分布式存储、数据备份 | Ceph、HDFS |
| 监控层 | 实时监控、智能告警 | Prometheus、Grafana |

### 3.3 最佳实践建议

1. **性能优化**：
   - 合理使用缓存
   - 优化数据库查询
   - 实施代码优化
   - 进行性能测试

2. **高可用保障**：
   - 实施冗余部署
   - 自动故障转移
   - 定期灾备演练
   - 完善监控告警

3. **运维管理**：
   - 自动化部署
   - 持续监控
   - 及时预警
   - 快速响应

## 4. 参考资源

### 4.1 技术文档

- [Prometheus官方文档](https://prometheus.io/docs/introduction/overview/)
- [Grafana官方文档](https://grafana.com/docs/)
- [MySQL高可用方案](https://dev.mysql.com/doc/mysql-ha-scalability/en/)
- [Redis高可用方案](https://redis.io/topics/sentinel)

### 4.2 工具推荐

- **性能测试工具**：
  - Apache JMeter
  - Gatling
  - wrk

- **监控工具**：
  - Prometheus
  - Grafana
  - Zabbix

- **高可用工具**：
  - Keepalived
  - HAProxy
  - Consul

### 4.3 相关书籍

- 《高性能MySQL》
- 《Redis设计与实现》
- 《SRE: Google运维解密》
- 《大型网站技术架构：核心原理与案例分析》

---

本文详细介绍了OPS/QPS性能指标和高可用架构的核心概念、实现方案和最佳实践。通过理解这些概念和技术，开发者和运维人员可以更好地设计、实现和维护高性能、高可用的系统。
