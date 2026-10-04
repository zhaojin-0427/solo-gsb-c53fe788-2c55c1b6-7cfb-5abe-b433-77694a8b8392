# DICOM 研究数据匿名化 API

基于 **Python + FastAPI + SQLite + pydicom + Pillow** 的 DICOM 研究数据匿名化服务。
客户端提交含多序列的 DICOM ZIP、版本化匿名化策略和可选像素遮挡框，服务输出
**确定性、可复验**的匿名化结果。

## 特性

- **确定性 UID 映射**：按 Study/Series/Instance 关系，用 HMAC-SHA256 将原始 UID
  映射为 `2.25.<decimal>` 形式的新 UID；同一 UID 在所有实例及引用处
  （`ReferencedSOPInstanceUID` 等）一致替换，保持引用完整性。
- **PHI 清除**：内置类 DICOM PS3.15 基本档案（姓名、出生日期、机构、医师、
  入组号等置空，`PatientID` 哈希化），默认删除全部私有标签；策略可增删改。
  **默认动作与显式动作统一递归覆盖所有保留下来的数据集层级**——顶层数据集及
  任意深度 SQ 序列项（`OriginalAttributesSequence`、`ContentSequence` 等）
  都按同一策略清洗；策略若移除（`remove`）或清空（`empty`）整个序列，则不再
  遍历/处理/计数其子项。
- **日期重写且保持时间差**：按患者派生确定性偏移（天数 + 秒级抖动），
  所有 DA/TM/DT 标签同步平移，跨实例时间差逐秒保持。
- **像素遮挡**：按遮挡框（instance/series/study 三级作用域）真正覆写像素
  （Pillow），支持 8/16 位灰度与 8 位交错 RGB，覆写后统一以
  Explicit VR Little Endian 重新编码。
- **字节级确定性**：相同输入 + 相同策略 ⇒ 输出 ZIP 字节完全一致
  （固定 ZIP 时间戳、确定性文件命名、确定性 DICOM 编码、HMAC 派生映射）。
- **全有或全无**：缺失引用、非法 VR、压缩像素、策略冲突等逐项报告，
  任一错误即整体失败，绝不产出半成品。
- **审计与可复验哈希**：结果附审计摘要（统计、UID 映射、警告）及
  输出 ZIP 与逐文件 SHA-256；审计计数按实际处理的标签出现次数累计
  （每个数据集层级各计一次，被移除序列的子项不计）；ZIP 内含
  `manifest.json` 供离线复验。
- **幂等提交**：相同输入 + 策略的重复提交返回同一作业（HTTP 200）。

## 快速开始（Docker Compose）

```bash
docker compose up --build
```

服务监听 <http://localhost:8000>：

- 交互式 API 文档：<http://localhost:8000/docs>
- 健康检查：`GET /healthz`
- 策略说明：`GET /policies`

> 生产部署请设置稳定的 `ANON_SECRET`（环境变量或 `.env` 文件），
> 它是 UID 映射 / 日期偏移 / 哈希的 HMAC 根密钥；变更后映射结果会改变。

## 本地开发

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
pytest tests/ -q          # 33 项测试
```

## 示例请求

### 1. 生成示例数据（1 个检查 × 2 个序列 × 2 个实例，含模拟 PHI）

```bash
python scripts/make_sample.py          # 生成 sample_input.zip
```

示例数据在每个实例的 `OriginalAttributesSequence` 内还挂了两层
`ContentSequence`，每层都藏有姓名、机构、病历号等 PHI，用于验证多层 SQ 的
递归匿名化（参见下节审计计数 `tags_emptied: 48` / `tags_hashed: 12`）。

### 2. 提交匿名化作业

```bash
curl -s -X POST http://localhost:8000/jobs \
  -F "file=@sample_input.zip" \
  -F "policy=<examples/policy.json" \
  -F "redactions=<examples/redactions.json"
```

响应（HTTP 202；重复提交返回 200 与同一 `job_id`）。下列数值用仓库自带的
`examples/policy.json` 与 `examples/redactions.json`、`ANON_SECRET` 为
compose 默认值时实际运行得到，可复验：

```json
{
  "job_id": "3c362ba1541397d69baf3a4291e5c8ba",
  "status": "queued",
  "status_url": "/jobs/3c362ba1541397d69baf3a4291e5c8ba",
  "detail": null
}
```

### 3. 查询状态与审计摘要

```bash
curl -s http://localhost:8000/jobs/<job_id>
```

成功时（节选；完整响应还含 `created_at` / `updated_at` / `input_sha256`）：

```json
{
  "job_id": "3c362ba1541397d69baf3a4291e5c8ba",
  "status": "done",
  "policy_version": "1.0",
  "policy_sha256": "692118d41bb079cf38e2a94c5cea806faf06a6a643fcfdacc0479df47cb0155b",
  "audit": {
    "summary": {
      "studies": 1, "series": 2, "instances": 4,
      "uids_mapped": 7,
      "tags_removed": 0,
      "tags_emptied": 48,
      "tags_replaced": 4,
      "tags_hashed": 12,
      "private_tags_removed": 8,
      "date_tags_shifted": 16,
      "redaction_boxes_applied": 2
    },
    "date_shift": { "enabled": true, "max_days": 365, "offset_groups": 1 },
    "uid_mapping": {
      "1.2.826.0.1.3680043.9.7433.1.1":
        "2.25.78141620074571268104463839734170551707",
      "1.2.826.0.1.3680043.9.7433.1.1.1":
        "2.25.134517156987503762300503873690750715363"
    },
    "output": {
      "zip_sha256": "19d8518165657f0bc0d05fbbc230fc89a4bf0f9f10c840734ae0ddc41b2fa009",
      "files": [
        {
          "path": "STUDY_0001/SERIES_0001/INSTANCE_00001.dcm",
          "sha256": "be27c2691fcd5ff88a5b8e0eccff383ff86cbe92d63ba34c1c4865e702fea06d",
          "study_instance_uid": "2.25.78141620074571268104463839734170551707",
          "series_instance_uid": "2.25.134517156987503762300503873690750715363",
          "sop_instance_uid": "2.25.144478445824475544209090090708132039840"
        }
      ]
    }
  },
  "result": {
    "sha256": "19d8518165657f0bc0d05fbbc230fc89a4bf0f9f10c840734ae0ddc41b2fa009",
    "size_bytes": 15785,
    "download_url": "/jobs/3c362ba1541397d69baf3a4291e5c8ba/download"
  }
}
```

计数说明（与上面的数字对应）：4 个实例各自在 4 个数据集层级
（顶层 + 3 层 SQ 项）携带默认档案中的应清洗标签，`tags_emptied=48`、
`tags_hashed=12` 均为跨层级的实际标签出现次数；序列层级本身被保留，
故其内部引用 UID 与 `…Sequence → ContentSequence → ContentSequence`
结构在输出中仍然存在，只是每个子数据集的 PHI 已被清洗。

失败时（逐项错误，无半成品）：

```json
{
  "status": "failed",
  "errors": [ { "file": "compressed.dcm", "code": "COMPRESSED_PIXELS",
                "message": "compressed transfer syntax 1.2.840.10008.1.2.4.50 is not supported; …" } ] }
```

策略冲突（如同一标签同时 `keep` 与 `remove`、对 SQ/二进制 VR 标签做
`hash`/`replace`）在 `GET /jobs/{id}` 中以 `POLICY_CONFLICT` 逐项返回，
此时下载接口返回 409。

### 4. 下载并复验

```bash
curl -s -OJ http://localhost:8000/jobs/<job_id>/download
sha256sum anonymized-<job_id>.zip        # 与 result.sha256 / audit.output.zip_sha256 一致
unzip -p anonymized-<job_id>.zip manifest.json   # 内含逐文件 SHA-256 与映射后 UID
```

以上述示例作业为例：

```text
$ sha256sum anonymized-3c362ba1541397d69baf3a4291e5c8ba.zip
19d8518165657f0bc0d05fbbc230fc89a4bf0f9f10c840734ae0ddc41b2fa009  anonymized-….zip
```

逐文件哈希、映射后的 Study/Series/SOP UID 均可与 `manifest.json` 对照复验；
同一输入 + 同一策略重复处理得到逐字节相同的 ZIP（重复提交还会直接命中
同一作业，HTTP 200）。

## API 一览

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/jobs` | 提交作业（multipart：`file` ZIP、`policy` JSON 字符串、`redactions` 可选 JSON 数组）。幂等：相同输入返回既有作业 |
| `GET` | `/jobs/{id}` | 查询状态 / 逐项错误 / 审计摘要 / 结果哈希 |
| `GET` | `/jobs/{id}/download` | 下载匿名化结果 ZIP（未完成返回 409） |
| `GET` | `/policies` | 支持的策略版本、默认档案、遮挡框格式 |
| `GET` | `/healthz` | 健康检查 |

## 匿名化策略（版本化）

```json
{
  "version": "1.0",
  "salt": "demo-research-project",
  "remove_private_tags": true,
  "strict_references": true,
  "date_shift": { "enabled": true, "max_days": 365 },
  "tags": {
    "keep":    ["PatientSex", "PatientAge", "StudyDescription"],
    "empty":   ["InstitutionName"],
    "remove":  ["PatientAddress"],
    "hash":    ["PatientID"],
    "replace": { "ReferringPhysicianName": "ANON^MD" }
  }
}
```

- `version`：必填，当前支持 `"1.0"`；不支持的版本报 `POLICY_INVALID`。
- `salt`：可选项目级盐，并入策略哈希，隔离不同项目的映射结果。
- `remove_private_tags`（默认 `true`）：删除全部奇数组号私有标签（含序列内）。
- `strict_references`（默认 `true`）：实例引用的 SOPInstanceUID 必须都在
  提交包内，否则报 `MISSING_REFERENCE`。
- `date_shift.max_days`：日期偏移上限（0–3650 天），偏移量由
  HMAC(患者键) 确定性派生，同一患者所有实例一致。
- `tags`：在默认档案之上叠加。`keep` 将标签从默认清除动作中豁免；
  `empty`/`remove`/`hash`/`replace` 追加或覆盖默认动作。**所有动作（含
  `keep` 豁免）都统一作用于顶层及任意深度 SQ 序列项内保留下来的每个
  数据集**，因此同名标签在所有层级得到一致处理。
- `remove` 作用于 SQ 标签时整个序列连同子项一起删除；`empty` 作用于 SQ
  标签时序列被清空为零个条目。两种情况下都不会再遍历子项，审计也不
  计数其子项的标签。对 SQ 或二进制 VR（`OB`/`OW`/`UN` 等）执行
  `hash`/`replace`、对二进制 VR 执行 `empty` 均报 `POLICY_CONFLICT`
  （这类值无法用标量策略值表示）。
- 同一标签被赋予两个动作、`replace` 值违反标签 VR、或试图修改
  受管标签（`StudyInstanceUID`、`PixelData`、行列像素描述符等）
  均报 `POLICY_CONFLICT`。

默认档案：约 30 个 PHI 标签置空（姓名、生日、机构、医师、入组号、
检查/序列描述等），`PatientID` 哈希（HMAC-SHA256 前 16 位十六进制），
`PatientSex/Age/Size/Weight` 默认保留。完整清单见 `GET /policies`。

## 像素遮挡框

```json
[
  { "scope": "series", "uid": "1.2.826.0.1.3680043.9.7433.1.1.1",
    "boxes": [[0, 0, 128, 30]], "fill": 0 }
]
```

- `scope`：`instance`（SOPInstanceUID）/ `series` / `study`。
- `boxes`：`[x, y, width, height]` 像素坐标（原点左上），多帧图像逐帧应用。
- `fill`：填充值 0–65535，默认 0（黑）。
- 越界（`REDACTION_OUT_OF_BOUNDS`）、目标不存在
  （`REDACTION_TARGET_NOT_FOUND`）、布局不支持
  （`UNSUPPORTED_PIXEL_LAYOUT`）都会使作业整体失败。

## 错误代码

| 代码 | 含义 |
| --- | --- |
| `ZIP_INVALID` | 非法/损坏的 ZIP 或条目 |
| `READ_ERROR` | 文件形似 DICOM 但无法解析 |
| `NOT_DICOM`（警告） | 非 DICOM 文件被跳过并记入 `warnings` |
| `MISSING_UID` | 缺少 SOP/Study/Series Instance UID 或 SOP Class UID |
| `DUPLICATE_INSTANCE` | 同一 SOPInstanceUID 出现多次 |
| `MISSING_REFERENCE` | 引用的 SOPInstanceUID 不在提交包内 |
| `INVALID_VR` / `INVALID_VALUE` | 未知 VR 或 DA/TM/DT/UI 值非法（附标签号；逐层递归检查，含 SQ 序列项） |
| `INVALID_DICOM` | 元素无法解码或数据集无法遍历（损坏的 DICOM 结构） |
| `COMPRESSED_PIXELS` | JPEG/RLE/Deflated 等压缩传输语法（需先解压） |
| `UNSUPPORTED_TRANSFER_SYNTAX` | 非隐式/显式 VR Little Endian（如 Big Endian） |
| `UNSUPPORTED_PIXEL_LAYOUT` | 遮挡目标像素布局不支持 |
| `REDACTION_*` | 遮挡目标缺失 / 遮挡框越界 |
| `POLICY_INVALID` / `POLICY_CONFLICT` | 策略结构非法 / 策略冲突 |
| `NO_INSTANCES` | 包内无可用 DICOM 实例 |

## 确定性说明

输出字节由以下因素唯一决定：输入 ZIP 字节、策略（规范化 JSON）、
遮挡规则、服务密钥 `ANON_SECRET`。具体措施：UID 映射 / 日期偏移 /
值哈希均用 HMAC-SHA256 派生；输出统一编码为 Explicit VR Little Endian、
固定文件元与 128 字节零前导；ZIP 条目按 `STUDY_xxxx/SERIES_xxxx/
INSTANCE_xxxxx.dcm` 确定性命名、固定 1980-01-01 时间戳与压缩级别；
`manifest.json` 键序固定。重复处理同一输入将得到逐字节相同的结果。

## 安全与合规提示

- 本服务面向研究数据去标识化场景；输出是否满足具体法规（如 HIPAA
  Safe Harbor、GDPR）需由数据负责人评估， burned-in 文本建议配合遮挡框。
- `ANON_SECRET` 决定映射可复现性，请妥善保管并仅在受信环境部署。
- UID 映射表（审计中返回）本身属于敏感关联信息，请按研究数据治理要求保存。
