# 合作夥伴 API

**Deepfens AI**

> 版本 1.5 · 2026-11-17
> 機密文件 · 僅供合作夥伴使用

本文件說明合作夥伴如何讀取 Deepfens 雷達裝置的資料：

- **狀態時間軸（Timeline）**：裝置狀態的變化紀錄，例如活動中、靜止、跌倒、無人。

所有 API 皆為**唯讀**。

---

## 1. 快速開始

```bash
# 1. 列出貴公司的裝置
curl https://www.deepfens.com/api/partner/v1/devices \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Accept: application/json"

# 2. 查詢單一裝置
curl "https://www.deepfens.com/api/partner/v1/devices/2442E357CC11" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Accept: application/json"
```

---

## 2. 基本資訊

| 項目 | 說明 |
| --- | --- |
| Base URL | `https://www.deepfens.com/api/partner/v1` |
| 通訊協定 | 僅支援 HTTPS |
| 資料格式 | JSON（UTF-8） |
| 時間格式 | ISO 8601，需包含時區，例如 `2026-10-30T09:15:00+08:00` |
| 裝置識別碼 | `identifier`：裝置 MAC 位址，共 12 個十六進位字元，例如 `2442E357CC11`（不區分大小寫） |

### 身分驗證

Deepfens 會提供貴公司一組 **API Key**。每次呼叫 API 時，都必須帶入以下 Header：

```http
Authorization: Bearer YOUR_API_KEY
Accept: application/json
```

- API Key **只能存取已分配給貴公司的裝置**。
- 請妥善保管 API Key，不要放在手機 App 或網頁前端程式碼中。
- 如需更換或停用 API Key，請聯絡 Deepfens。

### 呼叫頻率限制

- 每組 API Key **每分鐘最多 60 次請求**。
- 超過上限時，API 會回傳 `429 Too Many Requests`。請依照回應 Header `Retry-After` 指定的秒數等待後再重試。

---

## 3. API 一覽

| # | Method | 路徑 | 說明 |
| --- | --- | --- | --- |
| 1 | GET | `/devices` | 列出裝置及目前狀態 |
| 2 | GET | `/devices/{identifier}` | 查詢單一裝置的目前狀態 |
| 3 | GET | `/devices/{identifier}/timeline` | 查詢狀態時間軸 |

### 3.1 列出裝置

`GET /devices`

**查詢參數**

| 名稱 | 必填 | 說明 |
| --- | --- | --- |
| `per_page` | 否 | 每頁筆數，範圍 1–100，預設 50 |
| `cursor` | 否 | 下一頁的游標（請參考[分頁](#4-分頁)） |

**回應 `200 OK`**

```json
{
  "data": [
    {
      "identifier": "2442E357CC11",
      "name": "301 房浴室",
      "location": "3 樓",
      "is_online": true,
      "status": {
        "code": 9,
        "name": "somebody",
        "label": "有人"
      },
      "rssi": -71
    }
  ],
  "next_cursor": null
}
```

| 欄位 | 說明 |
| --- | --- |
| `identifier` | 裝置識別碼（MAC） |
| `name` | 裝置名稱 |
| `location` | 裝置所在位置 |
| `is_online` | 是否在線 |
| `status` | 目前狀態（請參考[狀態代碼](#5-裝置狀態代碼)） |
| `rssi` | Wi-Fi 訊號強度（dBm），數值越接近 0，訊號越好 |

### 3.2 查詢單一裝置

`GET /devices/{identifier}`

回傳內容與 `/devices` 中的單一裝置相同，資料放在 `data` 欄位內。

```json
{
  "data": {
    "identifier": "2442E357CC11",
    "name": "301 房浴室",
    "location": "3 樓",
    "is_online": true,
    "status": { "code": 9, "name": "somebody", "label": "有人" },
    "rssi": -71
  }
}
```

### 3.3 查詢狀態時間軸

`GET /devices/{identifier}/timeline`

裝置**每次狀態改變**時會產生一筆紀錄。回傳資料依時間排序，**最新的資料在最前面**。

**查詢參數**

| 名稱 | 必填 | 說明 |
| --- | --- | --- |
| `from` | 是 | 開始時間（包含） |
| `to` | 是 | 結束時間（不包含）。查詢區間**最長 31 天** |
| `status` | 否 | 依狀態名稱篩選；多個值以逗號分隔，例如 `fall,distress_call` |
| `per_page` | 否 | 每頁筆數，範圍 1–500，預設 100 |
| `cursor` | 否 | 下一頁的游標 |

**回應 `200 OK`**

```json
{
  "data": [
    {
      "id": 98231,
      "status": { "code": 1, "name": "fall", "label": "跌倒" },
      "occurred_at": "2026-10-29T22:41:07+08:00"
    },
    {
      "id": 98230,
      "status": { "code": 3, "name": "still", "label": "靜止" },
      "occurred_at": "2026-10-29T22:38:55+08:00"
    }
  ],
  "next_cursor": "eyJpZCI6OTgyMzB9"
}
```

**如何解讀：** 每個狀態會持續到下一筆紀錄出現為止。以上例來說，22:38:55 起偵測到人員**靜止**，到 22:41:07 偵測到**跌倒**。

---

## 4. 分頁

列表類 API 使用游標（cursor）分頁：

1. 第一次查詢時不帶 `cursor`。
2. 如果 `next_cursor` 不是 `null`，請使用相同參數再查詢一次，並加上 `cursor=<next_cursor>`。
3. 當 `next_cursor` 為 `null` 時，表示已沒有下一頁。

查詢各頁時，請保持 `from`、`to`、`status` 等其他參數相同。

---

## 5. 裝置狀態代碼

`status.code` 和 `status.name` 不會變動，可安心儲存或用於程式判斷；`label` 僅供畫面顯示，日後可能調整。

| code | name | 說明 |
| --- | --- | --- |
| 1 | `fall` | 偵測到跌倒 |
| 11 | `distress_call` | 人員透過語音呼救 |
| 10 | `resolved` | 警報解除，例如人員已起身或回覆沒事 |
| 2 | `moving` | 有人且正在活動 |
| 3 | `still` | 有人但長時間靜止 |
| 9 | `somebody` | 有人 |
| 4 | `vacant` | 偵測範圍內無人 |
| 5 | `bed_exit` | 離床 |
| 6 | `bed_entry` | 上床 |
| 7 | `bed_occupied` | 床上有人 |
| 8 | `bed_vacant` | 床上無人 |

不是每種型號都會回報所有狀態，例如床相關狀態只會由床墊感測器回報。

> **警報：** `fall`（1）與 `distress_call`（11）需要立即處理。收到 `resolved`（10）或其他狀態前，警報都會持續。

---

## 6. 錯誤處理

錯誤會使用標準 HTTP 狀態碼，回應格式如下：

```json
{
  "message": "查詢區間不可超過 31 天。",
  "errors": {
    "to": ["查詢區間不可超過 31 天。"]
  }
}
```

| HTTP | 發生情況 |
| --- | --- |
| `400` | 請求格式錯誤 |
| `401` | 未提供 API Key，或 API Key 無效 |
| `403` | 此 API Key 無權存取該裝置 |
| `404` | 找不到裝置 |
| `422` | 參數錯誤，例如時間格式錯誤或查詢區間過長；詳細原因請查看 `errors` |
| `429` | 請求次數過多；請依照 `Retry-After` 指定的時間等待後重試 |
| `500` / `503` | 伺服器錯誤；請稍後重試，並逐次拉長重試間隔 |

---

## 7. 資料保存期限

| 資料 | 保存期限 |
| --- | --- |
| 狀態時間軸 | 12 個月 |

---

## 8. 串接建議

- **定期查詢：** 查詢時間軸時，將 `from` 設為上次取得的最後一筆 `occurred_at`。一般儀表板**每 30 秒以上查詢一次**即可。
- **避免重複資料：** 使用紀錄的 `id` 判斷資料是否已處理過。
- **時區：** `from` 與 `to` 一定要包含時區，例如 `+08:00`。
- **商業邏輯使用 `status`：** 請依據穩定的 `status.code` 或 `status.name` 進行程式判斷。

---

## 9. 聯絡窗口

如需申請 API Key、設定裝置權限或詢問技術問題，請聯絡 Deepfens：`support@deepfens.com`
