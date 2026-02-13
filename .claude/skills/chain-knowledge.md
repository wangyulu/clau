
## 用户下单链路
用户端App → API Gateway (SLA: 5s) → OrderService.createOrder() (SLA: 2s) → InventoryService.reserve() (SLA: 500ms) → PaymentService.preAuth() (SLA: 1s) → NotificationService.push() (async, 不阻塞主链路) → 返回订单确认

## 关键 SLA 约束

| 链路 | 端到端 SLA | 备注 |
|------|-----------|------|
| 用户下单 | 5s | 超时则展示"处理中"兜底页 |
| 商品搜索 | 1s | P99 要求 |
| 支付回调 | 30s | 异步可容忍 |

## 最近事故记录

- 2024-12: OrderService 增加风控调用，链路增加 800ms，触发 5s SLA 告警
    - 根因：风控服务未配置超时，极端情况 3s+
    - 修复：增加 500ms 超时 + 降级策略