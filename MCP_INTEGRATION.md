# MCP_INTEGRATION — 麦当劳 MCP 集成说明

## 1. 使用的 MCP Server

| 项 | 内容 |
|:---|:---|
| 服务方 | 麦当劳中国 MCP Server（官方开放平台：https://open.mcd.cn/mcp） |
| 接入地址 | `https://mcp.mcd.cn` |
| 传输协议 | Streamable HTTP |
| 认证方式 | 请求头 `Authorization: Bearer <MCD_MCP_TOKEN>` |
| 配置示例 | 见仓库 `mcp-config.example.json`（仅环境变量占位符） |

本 Skill 的全部业务数据（派对商品、门店、日期场次、菜单、价格、优惠券、订单）均实时调用上述 MCP 获取，**流程内不写死任何场景参数**。

## 2. 实际使用的 Tool 清单

按派对策划全链路列出的实际调用工具：

| 阶段 | Tool | 用途 |
|:---|:---|:---|
| 时间校准 | `now-time-info` | 获取当前时间，把「下周末」「万圣节前后」等口语换算成具体可约日期 |
| 选派对商品 | `mall-points-products`（catRuleIds=`1>6>20`） | 拉取生日类派对商品列表（主题/价格/spuId） |
| 选派对商品 | `mall-product-detail` | 取商品 skuId、partyType（包场/拼团）、shopId 等落单必需参数 |
| 选址 | `query-party-city` | 确认商品在目标城市是否可参与 |
| 选址 | `query-party-store` | 列出该城市可办该派对的门店清单（店名/地址/距离/营业状态），由主办人逐家选 |
| 排期 | `query-party-store-date` | 查门店可预约日期 |
| 排期 | `query-party-store-session` | 查指定日期的可约场次（时间段/成团下限/余位/价格） |
| 定制餐品 | `query-meals` | 拉门店真实菜单：派对内含开心乐园餐选项 + 加购品（薯条/麦乐鸡/冰淇淋等）及单价 |
| 加购省钱 | `query-store-coupons` / `query-my-coupons` / `available-coupons` | 下单前查当前门店可用券与卡包券，有券直接推荐并写明省多少 |
| 核价 | `calculate-price` | 按选购清单（含券）计算应付总价 |
| 落单 | `party-order-create` | 创建派对主题订单（真实付费，复述+确认后执行） |
| 落单 | `create-order` | 创建额外加购餐品订单（真实付费，到店自取） |

## 3. 调用流程

```
主办人想法（城市/区域、时间想法、主题预算倾向、规模感）
   │
   ▼
now-time-info ──► 时间校准（口语日期 → 真实可约日期）
   │
   ▼
mall-points-products(1>6>20) ──► 主办人选派对商品（spu）
   │
   ▼
mall-product-detail ──► 取 skuId / partyType / shopId
   │
   ▼
query-party-city ──► query-party-store ──► 主办人从门店清单逐家选定
   │
   ▼
query-party-store-date ──► query-party-store-session ──► 主办人选日期+场次
   │
   ▼
收集名单（腾讯问卷 / 聊天框，发起时主办人自选）
   │
   ▼
query-meals ──► 按「年龄+想吃的+忌口」定制内含餐选项 + 加购清单
   │
   ▼
query-store-coupons + query-my-coupons + calculate-price ──► 有券省钱、核价
   │
   ▼
完整复述 ──► 主办人明说「创建订单」──► party-order-create + create-order（真实扣款）
   │
   ▼
落单成功 ──► 自动生成当日纪念（徽章+相框，纯本地绘制）
```

## 4. 业务价值

办一场小朋友的麦当劳派对，难点不是订场地，而是琐碎的协调：谁家娃来、几岁、有没有过敏、想吃啥、加购什么、人数够不够成团——靠家长在微信群接龙、手动统计，又慢又容易错，过敏信息还容易漏。

本 Skill 把这段协调整体接管：

- **数据驱动**：人数/忌口/年龄/想吃的全部来自一次收集，不回头在对话里逐项问；
- **过敏安全**：按过敏原剔除冲突餐项（如海鲜过敏避开鱼排堡），派对订单无主食变体参数这一接口限制也通过「现场告知卡」兜底；
- **省钱**：加购前查门店券+卡包券，同品多规格按性价比推荐；
- **真实可用**：全流程用真实 MCP 数据跑通并真实落单（2026-10-25 北京朝阳实跑），不是演示脚本。
