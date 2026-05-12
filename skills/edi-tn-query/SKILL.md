## name: edi-db description: Linker EDI / webMethods TN 数据库查询 skill。用于只读查询交易、单据、Partner、处理规则、TPA、错误日志和原始报文；默认通过 JumpServer 获取临时凭证，仅区分测试/生产环境。

> **MySQL / TN 查询工具**
>
> 仅用于只读查询。默认只执行 `SELECT / SHOW / DESCRIBE / EXPLAIN`。
>
> **默认查询流程（必须遵循）：**
>
> 
> 1. 用户一旦提出 MySQL / 数据库 / SQL / EDI / AS2 / TN 查询请求，默认优先使用本 skill 处理。
> 2. 只区分 `测试` / `生产`，不再询问 UAT。
> 3. 测试环境、生产环境默认都走 **JumpServer**，不要按直连 MySQL 处理。
> 4. 默认应自主通过 JumpServer API 获取临时凭证，不要先向用户索要连接命令。
> 5. 仅当自动取 token 失败、asset 不对、或后端认证失败无法继续时，再向用户补充索要必要信息。
> 6. 查询前只确认环境；若上下文已明确环境，则可直接查询，不重复确认。
> 7. 默认展示业务可读信息；Sender / Receiver 优先显示 Partner 名称，只有用户明确要求时才显示原始 ID。
> 8. 输出结果时，先给结论，再给关键字段或 SQL 证据。

# 环境变量要求

在执行 JumpServer 临时凭证获取示例前，需先准备以下环境变量：

* `JMS_KEY_ID`
* `JMS_KEY_SECRET`
* `EDI_DB_USER`
* `EDI_DB_PASSWORD`

如果环境变量缺失，应直接指出缺少哪些变量，不要伪造凭证。

# 数据库连接信息

## 测试环境

* **JumpServer**: `jumpserver.item.com`
* **MySQL Proxy Port**: `33061`
* **Asset**: `160052ac-1132-489b-ab19-ca0928276140`
* **Account**: `@INPUT`
* **Connect Method**: `web_cli`
* **Database**: `wm1015`
* **Backend User**: `${EDI_DB_USER}`
* **Backend Password**: `${EDI_DB_PASSWORD}`
* **JumpServer API Key ID**: `${JMS_KEY_ID}`
* **JumpServer API Key Secret**: `${JMS_KEY_SECRET}`
* **Auth**: HTTP Signature (`hmac-sha256`)，签名头 `(request-target) accept date`
* **Org Header**: `X-JMS-ORG: 00000000-0000-0000-0000-000000000002`

### 测试环境获取临时 token 示例

```python
import json, urllib.request, ssl, hmac, hashlib, base64, os
from email.utils import formatdate

KEY_ID = os.getenv('JMS_KEY_ID')
KEY_SECRET = os.getenv('JMS_KEY_SECRET')
INPUT_USERNAME = os.getenv('EDI_DB_USER')
INPUT_SECRET = os.getenv('EDI_DB_PASSWORD')
BASE = 'https://jumpserver.item.com'

missing = [
    name for name, value in {
        'JMS_KEY_ID': KEY_ID,
        'JMS_KEY_SECRET': KEY_SECRET,
        'EDI_DB_USER': INPUT_USERNAME,
        'EDI_DB_PASSWORD': INPUT_SECRET,
    }.items() if not value
]
if missing:
    raise RuntimeError(f'Missing env vars: {", ".join(missing)}')

def jms_request(method, path, body=None):
    date_str = formatdate(timeval=None, localtime=False, usegmt=True)
    signing_string = f'(request-target): {method.lower()} {path}\naccept: application/json\ndate: {date_str}'
    sig = base64.b64encode(hmac.new(KEY_SECRET.encode(), signing_string.encode(), hashlib.sha256).digest()).decode()
    auth = f'Signature keyId="{KEY_ID}",algorithm="hmac-sha256",headers="(request-target) accept date",signature="{sig}"'
    req = urllib.request.Request(
        BASE + path,
        data=json.dumps(body).encode() if body else None,
        headers={
            'Authorization': auth,
            'Accept': 'application/json',
            'Date': date_str,
            'X-JMS-ORG': '00000000-0000-0000-0000-000000000002',
            'Content-Type': 'application/json'
        },
        method=method.upper()
    )
    with urllib.request.urlopen(req, context=ssl.create_default_context()) as resp:
        return json.loads(resp.read())

creds = jms_request('POST', '/api/v1/authentication/connection-token/', {
    'asset': '160052ac-1132-489b-ab19-ca0928276140',
    'account': '@INPUT',
    'protocol': 'mysql',
    'connect_method': 'web_cli',
    'input_username': INPUT_USERNAME,
    'input_secret': INPUT_SECRET,
})

print(creds)
```

## 生产环境

* **JumpServer**: `jumpserver.item.com`
* **MySQL Proxy Port**: `33061`
* **Asset**: `cc3d9644-938b-48f8-a6fd-93083af7e862`
* **Account**: `@INPUT`
* **Connect Method**: `db_guide`
* **Database**: `linker_edi`
* **Backend User**: `${EDI_DB_USER}`
* **Backend Password**: `${EDI_DB_PASSWORD}`
* **JumpServer API Key ID**: `${JMS_KEY_ID}`
* **JumpServer API Key Secret**: `${JMS_KEY_SECRET}`
* **Auth**: HTTP Signature (`hmac-sha256`)，签名头 `(request-target) accept date`
* **Org Header**: `X-JMS-ORG: 00000000-0000-0000-0000-000000000002`

### 生产环境获取临时 token 示例

```python
import json, urllib.request, ssl, hmac, hashlib, base64, os
from email.utils import formatdate

KEY_ID = os.getenv('JMS_KEY_ID')
KEY_SECRET = os.getenv('JMS_KEY_SECRET')
INPUT_USERNAME = os.getenv('EDI_DB_USER')
INPUT_SECRET = os.getenv('EDI_DB_PASSWORD')
BASE = 'https://jumpserver.item.com'

missing = [
    name for name, value in {
        'JMS_KEY_ID': KEY_ID,
        'JMS_KEY_SECRET': KEY_SECRET,
        'EDI_DB_USER': INPUT_USERNAME,
        'EDI_DB_PASSWORD': INPUT_SECRET,
    }.items() if not value
]
if missing:
    raise RuntimeError(f'Missing env vars: {", ".join(missing)}')

def jms_request(method, path, body=None):
    date_str = formatdate(timeval=None, localtime=False, usegmt=True)
    signing_string = f'(request-target): {method.lower()} {path}\naccept: application/json\ndate: {date_str}'
    sig = base64.b64encode(hmac.new(KEY_SECRET.encode(), signing_string.encode(), hashlib.sha256).digest()).decode()
    auth = f'Signature keyId="{KEY_ID}",algorithm="hmac-sha256",headers="(request-target) accept date",signature="{sig}"'
    req = urllib.request.Request(
        BASE + path,
        data=json.dumps(body).encode() if body else None,
        headers={
            'Authorization': auth,
            'Accept': 'application/json',
            'Date': date_str,
            'X-JMS-ORG': '00000000-0000-0000-0000-000000000002',
            'Content-Type': 'application/json'
        },
        method=method.upper()
    )
    with urllib.request.urlopen(req, context=ssl.create_default_context()) as resp:
        return json.loads(resp.read())

creds = jms_request('POST', '/api/v1/authentication/connection-token/', {
    'asset': 'cc3d9644-938b-48f8-a6fd-93083af7e862',
    'account': '@INPUT',
    'protocol': 'mysql',
    'connect_method': 'db_guide',
    'input_username': INPUT_USERNAME,
    'input_secret': INPUT_SECRET,
})

print(creds)
```

# 查询优先级与定位方法

## 1. 用户给了业务标识时

如果用户提供以下任一信息，应先定位主记录，再继续补查轨迹与错误：

* 单据号 / 订单号
* Internal ID / BizDoc ID / DocID
* Partner 名称
* Document Type
* Rule 名称
* Sender / Receiver

## 2. 用户问“为什么报错”时

优先查：

* `activitylog`
* `bizdoc`
* `deliveryjob`
* `editracking`
* 相关错误消息 / 处理服务调用记录

## 3. 用户问“原始报文”时

优先查 `bizdoccontent`，必要时只返回关键片段，不默认整段回传超长内容。

## 4. 用户问 Partner / Rule / TPA 时

优先查：

* `partner`
* `partnerid`
* `processingrule`
* `tpa`
* `bizdoctypedef`

## 5. 用户给的是扩展属性 / 业务字段时

优先查：

* `bizdocattribute`
* `bizdocattributedef`
* 必要时配合 `bizdoc` 关联

# 已验证的 TN 重点表

> 以下表更贴近当前实际 TN 查询场景，优先于通用 `edi_*` 假设表。

| 表名  | 用途  |
|-----|-----|
| `edi_transaction_log` | 交易检索入口，按单号、方向、partner、消息类型等定位 |
| `bizdoc` | TN 主单据记录，含状态、DocID、SenderID/ReceiverID、文档类型 |
| `activitylog` | 活动日志，查看处理过程、路由、映射、状态变化 |
| `deliveryjob` | 投递记录，查看是否发送、是否重试、失败原因 |
| `editracking` | EDI 跟踪信息 / 回执状态 |
| `bizdoccontent` | 原始报文 / 内容片段 |
| `partner` | Partner 主数据，取 `CorporationName` 展示业务名称 |
| `partnerid` | Partner 外部标识与内部 PartnerID 映射 |
| `processingrule` | TN 处理规则定义 |
| `tpa` | TPA/Agreement 配置，按 SenderID / ReceiverID / AgreementID 定位 |
| `bizdoctypedef` | 文档类型定义，`TypeID` 到类型名称映射 |
| `bizdocattribute` | 单据扩展属性值 |
| `bizdocattributedef` | 单据扩展属性定义，属性名到 AttributeID 映射 |
| `bizdocrelationship` | 单据关联关系，查看父子/关联单据 |
| `deliveryservice` | 投递服务配置 |
| `deliveryqueue` | 投递队列状态 |

# 常用只读 SQL 模板

## 1. 看 TN 核心表是否存在

```sql
SHOW TABLES LIKE 'bizdoc';
SHOW TABLES LIKE 'activitylog';
SHOW TABLES LIKE 'deliveryjob';
SHOW TABLES LIKE 'edi_transaction_log';
SHOW TABLES LIKE 'editracking';
SHOW TABLES LIKE 'bizdoccontent';
SHOW TABLES LIKE 'partner';
SHOW TABLES LIKE 'partnerid';
SHOW TABLES LIKE 'processingrule';
SHOW TABLES LIKE 'tpa';
SHOW TABLES LIKE 'bizdoctypedef';
SHOW TABLES LIKE 'bizdocattribute';
SHOW TABLES LIKE 'bizdocattributedef';
SHOW TABLES LIKE 'bizdocrelationship';
SHOW TABLES LIKE 'deliveryservice';
SHOW TABLES LIKE 'deliveryqueue';
```

## 2. 查表结构

```sql
DESCRIBE edi_transaction_log;
DESCRIBE bizdoc;
DESCRIBE activitylog;
DESCRIBE deliveryjob;
DESCRIBE editracking;
DESCRIBE bizdoccontent;
DESCRIBE partner;
DESCRIBE partnerid;
DESCRIBE processingrule;
DESCRIBE tpa;
DESCRIBE bizdoctypedef;
DESCRIBE bizdocattribute;
DESCRIBE bizdocattributedef;
DESCRIBE bizdocrelationship;
DESCRIBE deliveryservice;
DESCRIBE deliveryqueue;
```

## 3. 按单号 / trans_key 查交易入口

```sql
SELECT *
FROM edi_transaction_log
WHERE trans_key = 'YOUR_DOC_NO'
   OR trans_key LIKE 'YOUR_DOC_NO%'
ORDER BY `timestamp` DESC
LIMIT 20;
```

## 4. 按 Internal ID / DocID 查 bizdoc

```sql
SELECT *
FROM bizdoc
WHERE DocID = 'YOUR_DOC_ID'
LIMIT 20;
```

## 5. 查单据状态

```sql
SELECT DocID, SenderID, ReceiverID, DocTypeID, RoutingStatus, UserStatus, LastModified
FROM bizdoc
WHERE DocID = 'YOUR_DOC_ID'
LIMIT 20;
```

## 6. 查 activitylog 处理轨迹

```sql
SELECT *
FROM activitylog
WHERE RelatedDocID = 'YOUR_DOC_ID'
ORDER BY EntryTimestamp DESC
LIMIT 100;
```

## 7. 查 deliveryjob 投递记录

```sql
SELECT *
FROM deliveryjob
WHERE DocID = 'YOUR_DOC_ID'
ORDER BY TimeUpdated DESC
LIMIT 100;
```

## 8. 查 editracking

```sql
SELECT *
FROM editracking
WHERE DocID = 'YOUR_DOC_ID'
ORDER BY DocTimestamp DESC
LIMIT 100;
```

## 9. 查原始报文

```sql
SELECT *
FROM bizdoccontent
WHERE DocID = 'YOUR_DOC_ID'
LIMIT 20;
```

## 10. 查原始报文关键片段

```sql
SELECT DocID, LEFT(Content, 2000) AS payload_preview
FROM bizdoccontent
WHERE DocID = 'YOUR_DOC_ID'
LIMIT 20;
```

## 11. 按 Partner / 文档类型筛最近单据

```sql
SELECT DocID, SenderID, ReceiverID, DocTypeID, RoutingStatus, UserStatus, LastModified
FROM bizdoc
WHERE SenderID = 'YOUR_SENDER_ID'
   OR ReceiverID = 'YOUR_RECEIVER_ID'
   OR DocTypeID = 'YOUR_DOC_TYPE_ID'
ORDER BY LastModified DESC
LIMIT 50;
```

## 12. DocID 关联 Partner 名称与文档类型名称

```sql
SELECT b.DocID,
       sp.CorporationName AS SenderName,
       rp.CorporationName AS ReceiverName,
       dt.TypeName AS DocumentTypeName,
       b.RoutingStatus,
       b.UserStatus,
       b.LastModified
FROM bizdoc b
LEFT JOIN partner sp ON sp.PartnerID = b.SenderID
LEFT JOIN partner rp ON rp.PartnerID = b.ReceiverID
LEFT JOIN bizdoctypedef dt ON dt.TypeID = b.DocTypeID
WHERE b.DocID = 'YOUR_DOC_ID'
LIMIT 20;
```

## 13. 按 Partner 名称查 PartnerID

```sql
SELECT PartnerID, CorporationName, OrgUnitName, Status, Type, LastModified
FROM partner
WHERE CorporationName = 'YOUR_PARTNER_NAME'
   OR CorporationName LIKE '%YOUR_PARTNER_NAME%'
ORDER BY CorporationName
LIMIT 50;
```

## 14. 按外部 Sender/Receiver ID 反查 Partner

```sql
SELECT p.PartnerID, p.CorporationName, p.OrgUnitName, pi.ExternalID, pi.IDType, pi.SequenceNumber
FROM partnerid pi
JOIN partner p ON p.PartnerID = pi.InternalID
WHERE pi.ExternalID = 'YOUR_EXTERNAL_ID'
   OR pi.ExternalID LIKE 'YOUR_EXTERNAL_ID%'
ORDER BY p.CorporationName, pi.SequenceNumber
LIMIT 50;
```

## 15. 按 Rule 名称查 processingrule

```sql
SELECT RuleID, RuleName, RuleDescription, Disabled, LastModified, LastChangeUser
FROM processingrule
WHERE RuleName = 'YOUR_RULE_NAME'
   OR RuleName LIKE '%YOUR_RULE_NAME%'
ORDER BY LastModified DESC
LIMIT 50;
```

## 16. 查 activitylog 涉及的处理规则

```sql
SELECT al.ActivityLogID, al.RelatedDocID, al.RelatedProcRuleID, pr.RuleName, al.EntryClass, al.BriefMessage, al.EntryTimestamp
FROM activitylog al
LEFT JOIN processingrule pr ON pr.RuleID = al.RelatedProcRuleID
WHERE al.RelatedDocID = 'YOUR_DOC_ID'
ORDER BY al.EntryTimestamp DESC
LIMIT 100;
```

## 17. 按 AgreementID / SenderID / ReceiverID 查 TPA

```sql
SELECT TpaID, AgreementID, SenderID, ReceiverID, Status, ExportService, InitService, ValidationSvc, LastModified
FROM tpa
WHERE AgreementID = 'YOUR_AGREEMENT_ID'
   OR SenderID = 'YOUR_SENDER_ID'
   OR ReceiverID = 'YOUR_RECEIVER_ID'
ORDER BY LastModified DESC
LIMIT 50;
```

## 18. DocTypeID 反查文档类型名称

```sql
SELECT TypeID, TypeName, TypeDescription, LastModified
FROM bizdoctypedef
WHERE TypeID = 'YOUR_DOC_TYPE_ID'
   OR TypeName = 'YOUR_DOC_TYPE_NAME'
   OR TypeName LIKE '%YOUR_DOC_TYPE_NAME%'
LIMIT 50;
```

## 19. 查单据扩展属性值

```sql
SELECT a.DocID, d.AttributeName, d.AttributeType, a.StringValue, a.NumberValue, a.DateValue
FROM bizdocattribute a
JOIN bizdocattributedef d ON d.AttributeID = a.AttributeID
WHERE a.DocID = 'YOUR_DOC_ID'
ORDER BY d.AttributeName
LIMIT 200;
```

## 20. 按属性名/属性值反查单据

```sql
SELECT a.DocID, d.AttributeName, a.StringValue, a.NumberValue, a.DateValue
FROM bizdocattribute a
JOIN bizdocattributedef d ON d.AttributeID = a.AttributeID
WHERE d.AttributeName = 'YOUR_ATTRIBUTE_NAME'
  AND a.StringValue = 'YOUR_ATTRIBUTE_VALUE'
LIMIT 100;
```

## 21. 查关联单据

```sql
SELECT r.DocID, r.RelatedDocID, r.Relationship
FROM bizdocrelationship r
WHERE r.DocID = 'YOUR_DOC_ID'
   OR r.RelatedDocID = 'YOUR_DOC_ID'
LIMIT 100;
```

## 22. 查投递服务配置

```sql
SELECT ServiceName, B2BInterface, B2BService, RemoteLocation, Scheduled
FROM deliveryservice
WHERE ServiceName = 'YOUR_SERVICE_NAME'
   OR ServiceName LIKE '%YOUR_SERVICE_NAME%'
LIMIT 50;
```

## 23. 查投递队列状态

```sql
SELECT QueueName, QueueType, QueueState
FROM deliveryqueue
WHERE QueueName = 'YOUR_QUEUE_NAME'
   OR QueueName LIKE '%YOUR_QUEUE_NAME%'
LIMIT 50;
```

## 24. 推荐的高频联查模板

### 24.1 DocID 一次看全貌

```sql
SELECT b.DocID,
       b.NativeID,
       sp.CorporationName AS SenderName,
       rp.CorporationName AS ReceiverName,
       dt.TypeName AS DocumentTypeName,
       b.RoutingStatus,
       b.UserStatus,
       b.DocTimestamp,
       b.LastModified
FROM bizdoc b
LEFT JOIN partner sp ON sp.PartnerID = b.SenderID
LEFT JOIN partner rp ON rp.PartnerID = b.ReceiverID
LEFT JOIN bizdoctypedef dt ON dt.TypeID = b.DocTypeID
WHERE b.DocID = 'YOUR_DOC_ID'
LIMIT 20;
```

### 24.2 NativeID / 业务单号 反查 DocID

```sql
SELECT b.DocID,
       b.NativeID,
       sp.CorporationName AS SenderName,
       rp.CorporationName AS ReceiverName,
       dt.TypeName AS DocumentTypeName,
       b.RoutingStatus,
       b.UserStatus,
       b.LastModified
FROM bizdoc b
LEFT JOIN partner sp ON sp.PartnerID = b.SenderID
LEFT JOIN partner rp ON rp.PartnerID = b.ReceiverID
LEFT JOIN bizdoctypedef dt ON dt.TypeID = b.DocTypeID
WHERE b.NativeID = 'YOUR_NATIVE_ID'
   OR b.NativeID LIKE 'YOUR_NATIVE_ID%'
ORDER BY b.LastModified DESC
LIMIT 50;
```

### 24.3 DocID 一次看错误轨迹 + 投递状态

```sql
SELECT 'activitylog' AS source,
       al.EntryTimestamp AS event_time,
       al.EntryClass AS status_or_class,
       al.BriefMessage AS message,
       pr.RuleName AS related_rule,
       NULL AS transport_status
FROM activitylog al
LEFT JOIN processingrule pr ON pr.RuleID = al.RelatedProcRuleID
WHERE al.RelatedDocID = 'YOUR_DOC_ID'
UNION ALL
SELECT 'deliveryjob' AS source,
       dj.TimeUpdated AS event_time,
       dj.JobStatus AS status_or_class,
       dj.TransportStatusMessage AS message,
       dj.ServiceName AS related_rule,
       dj.TransportStatus AS transport_status
FROM deliveryjob dj
WHERE dj.DocID = 'YOUR_DOC_ID'
ORDER BY event_time DESC
LIMIT 200;
```

## 25. SQL 优化建议

* `bizdoc` 优先用 `DocID`、`NativeID`、`SenderID`、`ReceiverID`、`DocTypeID` 过滤；这些字段已有索引。
* `activitylog` 优先用 `RelatedDocID` 过滤，再按 `EntryTimestamp` 排序；避免全表扫日志。
* `deliveryjob` 优先用 `DocID` 过滤，再按 `TimeUpdated` 排序；该组合有可用索引。
* `edi_transaction_log` 优先按 `trans_key` 精确匹配；模糊匹配仅作为补充。
* `partner` 优先按 `CorporationName` 精确匹配；仅在名称不完整时再用 `%LIKE%`。
* `partnerid` 按 `ExternalID` 反查 Partner 时优先带上 `IDType`，结果更准。
* `tpa` 优先按 `AgreementID` 精确定位，其次 `SenderID / ReceiverID`。
* `bizdocattribute` 反查时先用 `AttributeName` 锁定，再匹配 `StringValue`；不要无条件扫全部属性。
* 查询原始报文默认先取 `LEFT(Content, N)` 预览，只有用户明确要全文时再返回更长内容。
* 除非 SQL 报错或用户明确要求，不要每次都先 `DESCRIBE` 全表，以减少往返和延迟。

# 错误分析输出格式

当用户问“为什么失败 / 为什么报错 / 为什么没发出去”时，按以下结构输出：

* **现象**：当前看到的报错、状态、异常点
* **原因**：最可能根因
* **证据**：来自 `activitylog` / `bizdoc` / `deliveryjob` / `editracking` 的关键字段或日志
* **建议排查点**：下一步该看哪个规则、服务、partner、payload 或外部系统

# 输出要求

* 先给结论，再给关键字段。
* 默认用中文回答。
* 优先展示 `transaction id / 单据号 / 状态 / SenderName / ReceiverName / DocumentTypeName / 更新时间`。
* 如果字段或表名与预期不一致，先 `SHOW COLUMNS` / `DESCRIBE` 验证，再调整 SQL。
* 不要编造数据库结构。
* 不要执行任何写操作。

# 何时读取参考文件

* 需要看 EDI/AS2 表字段详情时，优先读取 `references/tables-edi.md`
* 需要看日志类表字段时，读取 `references/tables-log-mcp.md`
* 如果参考文件和真实库结构不一致，以 `SHOW COLUMNS` / `DESCRIBE` 实际结果为准

# 报文重发能力（SubmitTest）

本 Skill 支持将已查询到的 X12 原始报文重新提交到测试接口，用于测试环境报文重发。

---

# 重发接口

- URL:
  `https://edi-staging.item.com:5555/invoke/ATEST_WLG.TEST:SubmitTest`

- Method:
  `POST`

- Content-Type:
  `application/x-www-form-urlencoded`

- Auth:
  Anonymous（无需认证头）

- Form Field:
  - `X12`: 原始 X12 报文内容

---

# 重发触发条件

只有当用户明确要求以下动作时，才允许调用重发接口：

- 重发报文
- resend
- reprocess
- 重新推送
- 重新提交
- resubmit

如果用户只是查询、分析、查看报文，不允许自动重发。

---

# 重发前必须执行

重发前必须确认：

1. 已成功查询到原始 X12 报文
2. 当前环境是测试环境
3. 用户明确要求执行重发
4. 原始报文完整有效
5. X12 内容放入表单字段 `X12`

---

# 重发实现示例

```python
import urllib.request
import urllib.parse

submit_url = "https://edi-staging.item.com:5555/invoke/ATEST_WLG.TEST:SubmitTest"

payload = urllib.parse.urlencode({
    "X12": x12_payload
}).encode("utf-8")

request = urllib.request.Request(
    submit_url,
    data=payload,
    headers={
        "Content-Type": "application/x-www-form-urlencoded"
    },
    method="POST"
)

try:
    with urllib.request.urlopen(request, timeout=60) as response:
        response_body = response.read().decode("utf-8", errors="replace")

        result = {
            "success": True,
            "http_status": response.status,
            "response": response_body
        }

except Exception as e:
    result = {
        "success": False,
        "error": str(e)
    }
````

---

# 重发后输出格式

重发完成后，必须返回：

* 是否提交成功
* HTTP 状态码
* 接口名称
* 服务器响应内容
* 下一步建议

推荐输出格式：

```text
重发结果：
- 接口：ATEST_WLG.TEST:SubmitTest
- HTTP 状态码：
- 是否成功：
- 响应内容：

建议：
- 可继续查询 TN 确认是否生成新的 BizDoc / ActivityLog。
```

---

# 重发后自动验证（推荐）

重发成功后，建议自动：

1. 再次查询 `edi_transaction_log`
2. 查询新的 `bizdoc`
3. 查询 `activitylog`
4. 确认是否生成新的交易记录
5. 输出新的 DocID / Internal ID

---

# 安全限制

* 当前重发接口仅允许测试环境使用
* 禁止用于生产环境
* 禁止自动循环重发
* 禁止在未查询到原始报文时执行重发
* 禁止自动修改 X12 内容
* 除非用户明确要求，否则禁止编辑报文内容

---

# 错误处理

如果 SubmitTest 返回：

* HTTP 4xx
* HTTP 5xx
* timeout
* connection refused

必须：

1. 返回 HTTP 状态码
2. 返回错误信息
3. 返回接口响应内容
4. 提示用户后续排查方向

不要伪造成功结果。

---

# 示例用户请求

用户：

```text
查订单 5502240467，并重发 945
```

你应该：

1. 查询订单对应交易
2. 找到最新 945 BizDoc
3. 提取原始 X12 报文
4. 调用 SubmitTest
5. 返回 HTTP 状态码和响应
6. 再查询 TN 验证是否生成新的 BizDoc
7. 总结结果

```
```
