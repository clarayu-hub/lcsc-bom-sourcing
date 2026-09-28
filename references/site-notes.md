# 站点与抓取笔记

开始抓取前先读这一份，可以省掉大量试错。

## URL 格式（关键）

**国内站详情页必须用数字 ID，用 C 编码拼链接会 404：**

```
正确：https://item.szlcsc.com/<数字ID>.html
错误：https://item.szlcsc.com/detail/<C编码>.html     → 404
```

**定位数字 ID 用搜索页：**

```
https://so.szlcsc.com/global.html?k=<C编码或型号>
```

搜索页返回 `item.szlcsc.com/<数字ID>.html` 链接，以及型号/品牌/封装/C编码/参数。

> 注意：搜索页的「价格梯度」「库存交期」列不渲染，只有表头。价格与库存必须进详情页才有。

搜索关键词可同时带规格词（如 `0603 10K 1%`），一次返回完整候选列表，是批量筛选的首选入口。

国际站详情页：`https://www.lcsc.com/product-detail/<C编码>.html`（可直接用 C 编码；**仅在国内站有货但参数不全时使用**）。

## 国际站不能用于采购（重要）

- LCSC 国际站面向海外渠道：美元结算、DHL 国际件，官方条款明确不支持收货地址为中国大陆（与 JLCPCB 合并订单条款原文 "cannot ship to Mainland China"）。
- 被问「国际站有货国内能不能买」→ 答案是**不能**。改走国内站路径：① 预售/预定（平台向原厂调货入国内仓，人民币含税）；② 到货通知等补货；③ 联系平台销售询货；④ 放宽规格。
- 贴片扩展库有量只对贴片服务有效，**买不了散料**。

## 跨站库存差异

同一料号在国内外两站的现货数量可能差几个数量级（不同仓、不同渠道配额）。国际站库存只能当「有货/没货」的粗筛信号，**不能当国内现货报给使用者**，更不能拿国际站「有货」推翻国内站「缺货」的结论。

## 源码级抓取（反爬限频时的兜底）

连续请求后可能出现：搜索页跳登录墙、详情页动态购买区不渲染（拿不到库存与价格）。此时**不要硬重试**，改用源码级抓取：

- 用宿主自带的 HTTP 能力（如 `Invoke-WebRequest -UseBasicParsing`、Python `urllib.request`、`curl`）带 Chrome UA 直接拉 `item.szlcsc.com/<id>.html` 的 HTML，存文件后检索内嵌 JSON。
- 页面为服务端渲染，内嵌 JSON 中有权威字段：

| 字段 | 含义 |
|---|---|
| `stockNumber` | 库存总量（注意可能含贴片扩展库量） |
| `hasStockNow` | `"no"` 表示当前无现货 |
| `totalStockNumber` | `0` 表示不可直接购买 |
| `gdWarehouseStockNumber` / `jsWarehouseStockNumber` | 分仓现货 |
| `smtStockNumber` | 仅贴片服务可用，**不能当现货** |
| `minBuyNumber` / `minWholeNumber` | 最小起订量 / 每盘数量 |
| `"price":x` | 单一最低档价；完整阶梯价需登录后查看 |

- **判定以 `hasStockNow` / `totalStockNumber` 为准。** 实测有料号显示贴片库 65 个却不可购买——「贴片库有量 ≠ 有现货」。
- 旁路：某些贴片平台（jlcpcb.com）的 partdetail URL 中数字 ID 与详情页数字 ID 相同，可在搜索页被墙时用于定位数字 ID；但这类站点的快照库存与价格属贴片通道口径，**仅参考不引用**。

## 官方 API

`ips.lcsc.com/rest/wmsc2agent/*` 需申请 key 并有签名校验，无 key 不可用。

## 每次都必须做

给结果标注**抓取时间**；价格与库存随时变动。
