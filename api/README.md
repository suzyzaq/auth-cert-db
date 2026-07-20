# 授权资质查询 API · 接口说明（静态 JSON，免鉴权）

> 数据来源：欧菲斯 SCS 平台授权列表 + 资质池商标注册证，每日 17:00 自动更新。
> 适用：Dify / Coze / 任意 HTTP 客户端，按**品牌**查询授权书与商标注册证，只返回命中数据（KB 级），无需下载整页。

## 接口清单（全部 GET，返回 JSON）

| 接口 | 说明 | 大小 |
|---|---|---|
| `https://suzyzaq.github.io/auth-cert-db/api/index.json` | 品牌索引：品牌名 → 分片号 | ~60 KB |
| `https://suzyzaq.github.io/auth-cert-db/api/shard/{分片号}.json` | 品牌数据分片（含授权+商标明细） | 每片 ≤300 KB |

index.json 结构：
```json
{
  "today": "2026-07-20",
  "generated": "2026-07-20 18:00:00",
  "shardCount": 12,
  "brandCount": 4212,
  "brands": { "惠普/HP": 0, "惠普": 9, "惠普/hp": 9, "...": 3 }
}
```

shard/N.json 结构：
```json
{
  "惠普/HP": {
    "auth": [ {授权记录}, ... ],
    "cert": [ {商标注册证记录}, ... ]
  },
  "其他品牌": { "auth": [], "cert": [] }
}
```

## 字段字典

**auth（授权书）**：`code` 授权编号 / `name` 授权名称 / `type` 授权类型 / `grant` 授权方 / `authed` 被授权方 / `category` 授权品类 / `region` 授权区域 / `enable` 生效日期 / `disable` 失效日期 / `months` 剩余有效期(月，负数=已过期) / `flag` valid|expiring|expired / `flagText` 有效|临期|失效 / `files` 附件链接数组（图片或PDF）

**cert（商标注册证）**：`code` 证书编号 / `name` 证书名称 / `type` 类型 / `subject` 注册人 / `model` 型号 / `enable` 注册日期 / `disable` 有效期至 / `months` 剩余有效期(月) / `flag` / `flagText` / `files` 附件链接数组

## Dify 工作流配置（3 个节点）

**节点1 · HTTP 请求（GET 索引）**
- 方法：GET
- URL：`https://suzyzaq.github.io/auth-cert-db/api/index.json`

**节点2 · 代码执行（品牌模糊匹配 → 待查分片）**（Python3）
```python
def main(index: dict, keyword: str) -> dict:
    brands = index.get("brands", {})
    hits = [{"brand": b, "shard": s} for b, s in brands.items() if keyword in b]
    shards = sorted({h["shard"] for h in hits})
    return {"hits": hits, "shards": shards, "found": len(hits) > 0}
```
- 输入：`index` = 节点1 body 转 JSON，`keyword` = 用户品牌关键词（如"惠普"）

**节点3 · HTTP 请求（GET 分片，多个分片用迭代）**
- 方法：GET
- URL：`https://suzyzaq.github.io/auth-cert-db/api/shard/{{shard}}.json`
- 然后代码节点提取：
```python
def main(shard: dict, hits: list) -> dict:
    auth, cert = [], []
    for h in hits:
        g = shard.get(h["brand"])
        if g:
            auth += g.get("auth", [])
            cert += g.get("cert", [])
    return {"auth": auth, "cert": cert}
```

## 直接调用示例（curl）

```bash
curl -s https://suzyzaq.github.io/auth-cert-db/api/index.json     # 查"惠普"在分片 0 和 9
curl -s https://suzyzaq.github.io/auth-cert-db/api/shard/0.json   # 取"惠普/HP"的 11 条授权+10 条商标证
```

## 备注

- 同名品牌可能存在多个键（如 惠普 / 惠普/HP / 惠普/hp），建议按"包含关键词"模糊匹配后合并结果。
- 附件链接为欧菲斯 OSS 公网地址，可直接打开预览。
- 数据每日 17:00 自动更新，index.json 中 `today` 字段可校验数据日期。
