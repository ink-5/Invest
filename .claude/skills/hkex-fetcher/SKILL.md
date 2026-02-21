---
name: hkex-fetcher
description: 港交所披露易(HKEXnews)公告自动获取。用于搜索和下载港股上市公司的年报、中报、季报、公告等PDF文件。触发条件：(1) 用户要求获取/下载港股财报或公告PDF，(2) 用户提到"披露易"或"HKEXnews"，(3) 用户要求查找某港股公司的年报/中报/业绩公告，(4) 烟蒂股分析流程中需要获取年报原文进行Fact Check验证。
---

# 港交所披露易公告获取工具

## 概述

通过港交所披露易(HKEXnews)未公开但稳定可用的JSON REST API，自动搜索和下载港股上市公司的公告PDF文件。无需浏览器渲染、无需登录、无需Session。

## API端点

### 1. 股票代码转内部ID

```
GET https://www1.hkexnews.hk/search/prefix.do?callback=callback&lang=EN&type=A&name={股票代码}&market=SEHK
```

返回JSONP格式，需解析：
```javascript
callback({"more":"1","stockInfo":[{"stockId":97773,"code":"01522","name":"BII TRANS TECH"}]});
```

### 2. 搜索公告

```
GET https://www1.hkexnews.hk/search/titleSearchServlet.do
```

#### 完整参数表

| 参数 | 说明 | 常用值 |
|:-----|:-----|:------|
| `sortDir` | 排序方向 | `0`（降序，最新在前） |
| `sortByOptions` | 排序字段 | `DateTime` |
| `category` | 分类 | `0` |
| `market` | 市场 | `SEHK`（主板）、`GEM`（创业板） |
| `stockId` | 股票内部ID | 通过prefix API获取 |
| `documentType` | 文档类型 | `-1`（全部） |
| `t1code` | 一级分类 | 见下方分类表 |
| `t2Gcode` | 二级分组 | `-2`（全部） |
| `t2code` | 二级分类 | 见下方分类表 |
| `searchType` | 搜索类型 | `0`=按分类，`1`=按标题关键词 |
| `title` | 标题关键词 | 仅`searchType=1`时有效 |
| `lang` | 语言 | `ZH`=中文，`EN`=英文 |
| `rowRange` | 返回记录数 | `100` |
| `fromDate` | 起始日期 | `yyyyMMdd` |
| `toDate` | 结束日期 | `yyyyMMdd` |

#### 分类代码(t1code / t2code)

| t1code | 含义 | t2code | 含义 |
|:-------|:-----|:-------|:-----|
| `-2` | 全部 | `-2` | 全部 |
| `40000` | 财务报表/ESG | `40100` | 年报 |
| `40000` | 财务报表/ESG | `40200` | 中期报告 |
| `40000` | 财务报表/ESG | `40300` | 季报 |
| `40000` | 财务报表/ESG | `40400` | ESG报告 |
| `10000` | 公告及通告 | `-2` | 全部公告 |

### 3. PDF下载

返回JSON中每条记录的 `FILE_LINK` 字段拼接即可直接下载，无需认证：

```
https://www1.hkexnews.hk{FILE_LINK}
```

中文版PDF的 `FILE_LINK` 以 `_c.pdf` 结尾，英文版以 `.pdf` 结尾。

### 4. 返回JSON结构

```json
{
  "result": "[JSON数组字符串]",
  "hasNextRow": false,
  "rowRange": 100,
  "loadedRecord": 6,
  "recordCnt": 6
}
```

`result` 解析后每条记录：

```json
{
  "FILE_INFO": "4MB",
  "NEWS_ID": "11650832",
  "STOCK_NAME": "京投交通科技",
  "TITLE": "2024年報",
  "FILE_TYPE": "PDF",
  "DATE_TIME": "29/04/2025 17:00",
  "STOCK_CODE": "01522",
  "FILE_LINK": "/listedco/listconews/sehk/2025/0429/2025042901727_c.pdf"
}
```

## 标准工作流程

当用户要求获取某港股公司的财报或公告时，按以下步骤执行：

### Step 1: 获取stockId

```bash
curl -s "https://www1.hkexnews.hk/search/prefix.do?callback=callback&lang=EN&type=A&name={代码数字部分}&market=SEHK"
```

从返回的JSONP中提取 `stockId`。注意 `name` 参数传入不带前导零的数字（如1522而非01522）。精确匹配时用5位补零的 `code` 字段（如01522）。

### Step 2: 搜索公告

根据用户需求选择合适的 `t1code` 和 `t2code`：

```bash
# 搜索年报（中文版）
curl -s "https://www1.hkexnews.hk/search/titleSearchServlet.do?sortDir=0&sortByOptions=DateTime&category=0&market=SEHK&stockId={stockId}&documentType=-1&t1code=40000&t2Gcode=-2&t2code=40100&searchType=0&title=&lang=ZH&rowRange=100&fromDate={起始日期}&toDate={结束日期}"
```

常用搜索组合：
- 年报：`t1code=40000&t2code=40100`
- 中报：`t1code=40000&t2code=40200`
- 全部公告：`t1code=-2&t2code=-2`
- 按标题搜索：`searchType=1&title={关键词}`

### Step 3: 解析结果并下载

从返回JSON的 `result` 字段（需JSON.parse）中提取 `FILE_LINK`，拼接完整URL后用curl下载：

```bash
curl -o "{保存路径}" "https://www1.hkexnews.hk{FILE_LINK}"
```

建议保存路径格式：`F:/Invest/{股票代码}_{报告类型}.pdf`

## 注意事项

1. `lang=ZH` 返回中文版公告，`lang=EN` 返回英文版。中文版PDF链接以 `_c.pdf` 结尾
2. 请求频率建议控制在每秒1次以内，避免被限流
3. `rowRange` 控制返回记录数，先用小值获取 `recordCnt` 总数再按需调整
4. 数据最早可追溯到约2007年
5. 创业板公司需将 `market` 改为 `GEM`
