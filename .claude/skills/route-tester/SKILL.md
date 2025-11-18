---
name: route-tester
description: 在 your project 專案中測試已認證的路由，使用 Cookie 認證方式。當您需要測試 API 端點、驗證路由功能或除錯認證問題時使用此技能。包含 test-auth-route.js 的使用模式和模擬認證。
---

# your project 路由測試技能

## 目的
此技能提供在 your project 中測試已認證路由的模式，使用基於 Cookie 的 JWT 認證。

## 何時使用此技能
- 測試新的 API 端點
- 驗證修改後的路由功能
- 除錯認證問題
- 測試 POST/PUT/DELETE 操作
- 驗證請求/回應資料

## your project 認證概覽

your project 使用：
- **Keycloak** 進行 SSO（realm: yourRealm）
- **基於 Cookie 的 JWT** token（非 Bearer headers）
- **Cookie 名稱**：`refresh_token`
- **JWT 簽署**：使用 `config.ini` 中的 secret

## 測試方法

### 方法 1：test-auth-route.js（建議使用）

`test-auth-route.js` 腳本會自動處理所有認證複雜性。

**位置**：`/root/git/your project_pre/scripts/test-auth-route.js`

#### 基本 GET 請求

```bash
node scripts/test-auth-route.js http://localhost:3000/blog-api/api/endpoint
```

#### POST 請求並帶 JSON 資料

```bash
node scripts/test-auth-route.js \
    http://localhost:3000/blog-api/777/submit \
    POST \
    '{"responses":{"4577":"13295"},"submissionID":5,"stepInstanceId":"11"}'
```

#### 腳本功能說明

1. 從 Keycloak 取得 refresh token
   - 使用者名稱：`testuser`
   - 密碼：`testpassword`
2. 使用 `config.ini` 中的 JWT secret 簽署 token
3. 建立 cookie header：`refresh_token=<signed-token>`
4. 發送已認證的請求
5. 顯示可手動重現的完整 curl 指令

#### 腳本輸出

腳本會輸出：
- 請求詳細資訊
- 回應狀態和內容
- 用於手動重現的 curl 指令

**注意**：腳本輸出較為詳細，請留意實際的回應內容。

### 方法 2：使用 Token 的手動 curl

使用 test-auth-route.js 輸出的 curl 指令：

```bash
# 腳本會輸出類似這樣的內容：
# 💡 To test manually with curl:
# curl -b "refresh_token=eyJhbGci..." http://localhost:3000/blog-api/api/endpoint

# 複製並修改該 curl 指令：
curl -X POST http://localhost:3000/blog-api/777/submit \
  -H "Content-Type: application/json" \
  -b "refresh_token=<從腳本輸出複製的_TOKEN>" \
  -d '{"your": "data"}'
```

### 方法 3：模擬認證（僅限開發環境 - 最簡單）

在開發環境中，可完全繞過 Keycloak 使用模擬認證。

#### 設定

```bash
# 在服務的 .env 檔案中新增（例如 blog-api/.env）
MOCK_AUTH=true
MOCK_USER_ID=test-user
MOCK_USER_ROLES=admin,operations
```

#### 使用方式

```bash
curl -H "X-Mock-Auth: true" \
     -H "X-Mock-User: test-user" \
     -H "X-Mock-Roles: admin,operations" \
     http://localhost:3002/api/protected
```

#### 模擬認證要求

模擬認證僅在以下情況有效：
- `NODE_ENV` 為 `development` 或 `test`
- 路由已加入 `mockAuth` 中介軟體
- 在正式環境中絕對不會運作（安全機制）

## 常見測試模式

### 測試表單提交

```bash
node scripts/test-auth-route.js \
    http://localhost:3000/blog-api/777/submit \
    POST \
    '{"responses":{"4577":"13295"},"submissionID":5,"stepInstanceId":"11"}'
```

### 測試工作流程啟動

```bash
node scripts/test-auth-route.js \
    http://localhost:3002/api/workflow/start \
    POST \
    '{"workflowCode":"DHS_CLOSEOUT","entityType":"Submission","entityID":123}'
```

### 測試工作流程步驟完成

```bash
node scripts/test-auth-route.js \
    http://localhost:3002/api/workflow/step/complete \
    POST \
    '{"stepInstanceID":789,"answers":{"decision":"approved","comments":"Looks good"}}'
```

### 測試帶查詢參數的 GET 請求

```bash
node scripts/test-auth-route.js \
    "http://localhost:3002/api/workflows?status=active&limit=10"
```

### 測試檔案上傳

```bash
# 先從 test-auth-route.js 取得 token，然後：
curl -X POST http://localhost:5000/upload \
  -H "Content-Type: multipart/form-data" \
  -b "refresh_token=<TOKEN>" \
  -F "file=@/path/to/file.pdf" \
  -F "metadata={\"description\":\"Test file\"}"
```

## 硬編碼的測試憑證

`test-auth-route.js` 腳本使用以下憑證：

- **使用者名稱**：`testuser`
- **密碼**：`testpassword`
- **Keycloak URL**：從 `config.ini` 取得（通常是 `http://localhost:8081`）
- **Realm**：`yourRealm`
- **Client ID**：從 `config.ini` 取得

## 服務埠號

| 服務 | 埠號 | Base URL |
|---------|------|----------|
| Users   | 3000 | http://localhost:3000 |
| Projects| 3001 | http://localhost:3001 |
| Form    | 3002 | http://localhost:3002 |
| Email   | 3003 | http://localhost:3003 |
| Uploads | 5000 | http://localhost:5000 |

## 路由前綴

檢查每個服務中的 `/src/app.ts` 以確認路由前綴：

```typescript
// 範例來自 blog-api/src/app.ts
app.use('/blog-api/api', formRoutes);          // 前綴：/blog-api/api
app.use('/api/workflow', workflowRoutes);  // 前綴：/api/workflow
```

**完整路由** = Base URL + 前綴 + 路由路徑

範例：
- Base：`http://localhost:3002`
- 前綴：`/form`
- 路由：`/777/submit`
- **完整 URL**：`http://localhost:3000/blog-api/777/submit`

## 測試檢查清單

測試路由前：

- [ ] 識別服務（form、email、users 等）
- [ ] 找到正確的埠號
- [ ] 檢查 `app.ts` 中的路由前綴
- [ ] 建構完整的 URL
- [ ] 準備請求內容（若為 POST/PUT）
- [ ] 決定認證方法
- [ ] 執行測試
- [ ] 驗證回應狀態和資料
- [ ] 檢查資料庫變更（如適用）

## 驗證資料庫變更

測試會修改資料的路由後：

```bash
# 連線到 MySQL
docker exec -i local-mysql mysql -u root -ppassword1 blog_dev

# 檢查特定資料表
mysql> SELECT * FROM WorkflowInstance WHERE id = 123;
mysql> SELECT * FROM WorkflowStepInstance WHERE instanceId = 123;
mysql> SELECT * FROM WorkflowNotification WHERE recipientUserId = 'user-123';
```

## 測試失敗的除錯

### 401 Unauthorized

**可能原因**：
1. Token 已過期（使用 test-auth-route.js 重新產生）
2. Cookie 格式不正確
3. JWT secret 不符
4. Keycloak 未執行

**解決方法**：
```bash
# 檢查 Keycloak 是否執行中
docker ps | grep keycloak

# 重新產生 token
node scripts/test-auth-route.js http://localhost:3002/api/health

# 驗證 config.ini 中的 jwtSecret 是否正確
```

### 403 Forbidden

**可能原因**：
1. 使用者缺少必要的角色
2. 資源權限不正確
3. 路由需要特定權限

**解決方法**：
```bash
# 使用具有 admin 角色的模擬認證
curl -H "X-Mock-Auth: true" \
     -H "X-Mock-User: test-admin" \
     -H "X-Mock-Roles: admin" \
     http://localhost:3002/api/protected
```

### 404 Not Found

**可能原因**：
1. URL 不正確
2. 缺少路由前綴
3. 路由未註冊

**解決方法**：
1. 檢查 `app.ts` 中的路由前綴
2. 驗證路由註冊
3. 檢查服務是否執行中（`pm2 list`）

### 500 Internal Server Error

**可能原因**：
1. 資料庫連線問題
2. 缺少必要欄位
3. 驗證錯誤
4. 應用程式錯誤

**解決方法**：
1. 檢查服務日誌（`pm2 logs <service>`）
2. 檢查 Sentry 的錯誤詳細資訊
3. 驗證請求內容是否符合預期的 schema
4. 檢查資料庫連線狀態

## 使用 auth-route-tester Agent

在修改後進行完整的路由測試：

1. **識別受影響的路由**
2. **收集路由資訊**：
   - 完整的路由路徑（含前綴）
   - 預期的 POST 資料
   - 需要驗證的資料表
3. **呼叫 auth-route-tester agent**

此 agent 會：
- 使用正確的認證測試路由
- 驗證資料庫變更
- 檢查回應格式
- 回報任何問題

## 測試情境範例

### 建立新路由後

```bash
# 1. 使用有效資料測試
node scripts/test-auth-route.js \
    http://localhost:3002/api/my-new-route \
    POST \
    '{"field1":"value1","field2":"value2"}'

# 2. 驗證資料庫
docker exec -i local-mysql mysql -u root -ppassword1 blog_dev \
    -e "SELECT * FROM MyTable ORDER BY createdAt DESC LIMIT 1;"

# 3. 使用無效資料測試
node scripts/test-auth-route.js \
    http://localhost:3002/api/my-new-route \
    POST \
    '{"field1":"invalid"}'

# 4. 不帶認證測試
curl http://localhost:3002/api/my-new-route
# 應該回傳 401
```

### 修改路由後

```bash
# 1. 測試現有功能是否仍正常運作
node scripts/test-auth-route.js \
    http://localhost:3002/api/existing-route \
    POST \
    '{"existing":"data"}'

# 2. 測試新功能
node scripts/test-auth-route.js \
    http://localhost:3002/api/existing-route \
    POST \
    '{"new":"field","existing":"data"}'

# 3. 驗證向後相容性
# 使用舊的請求格式測試（如適用）
```

## 設定檔

### config.ini（每個服務）

```ini
[keycloak]
url = http://localhost:8081
realm = yourRealm
clientId = app-client

[jwt]
jwtSecret = your-jwt-secret-here
```

### .env（每個服務）

```bash
NODE_ENV=development
MOCK_AUTH=true           # 選用：啟用模擬認證
MOCK_USER_ID=test-user   # 選用：預設模擬使用者
MOCK_USER_ROLES=admin    # 選用：預設模擬角色
```

## 關鍵檔案

- `/root/git/your project_pre/scripts/test-auth-route.js` - 主要測試腳本
- `/blog-api/src/app.ts` - Form 服務路由
- `/notifications/src/app.ts` - Email 服務路由
- `/auth/src/app.ts` - Users 服務路由
- `/config.ini` - 服務設定檔
- `/.env` - 環境變數

## 相關技能

- 使用 **database-verification** 驗證資料庫變更
- 使用 **error-tracking** 檢查已捕捉的錯誤
- 使用 **workflow-builder** 進行工作流程路由測試
- 使用 **notification-sender** 驗證已傳送的通知
