# 配置管理 - UnifiedConfig 模式

後端微服務配置管理的完整指南。

## 目錄

- [UnifiedConfig 概述](#unifiedconfig-概述)
- [絕對不要直接使用 process.env](#絕對不要直接使用-processenv)
- [配置結構](#配置結構)
- [環境特定配置](#環境特定配置)
- [密鑰管理](#密鑰管理)
- [遷移指南](#遷移指南)

---

## UnifiedConfig 概述

### 為什麼使用 UnifiedConfig？

**process.env 的問題：**
- ❌ 沒有型別安全
- ❌ 沒有驗證
- ❌ 難以測試
- ❌ 散落在程式碼各處
- ❌ 沒有預設值
- ❌ 拼寫錯誤會導致執行期錯誤

**unifiedConfig 的優點：**
- ✅ 型別安全的配置
- ✅ 單一真實來源
- ✅ 啟動時驗證
- ✅ 易於使用 mock 測試
- ✅ 清晰的結構
- ✅ 可回退到環境變數

---

## 絕對不要直接使用 process.env

### 規則

```typescript
// ❌ 絕對不要這樣做
const timeout = parseInt(process.env.TIMEOUT_MS || '5000');
const dbHost = process.env.DB_HOST || 'localhost';

// ✅ 永遠這樣做
import { config } from './config/unifiedConfig';
const timeout = config.timeouts.default;
const dbHost = config.database.host;
```

### 為什麼這很重要

**問題範例：**
```typescript
// 環境變數名稱拼錯
const host = process.env.DB_HSOT; // undefined！沒有錯誤！

// 型別安全
const port = process.env.PORT; // string！需要 parseInt
const timeout = parseInt(process.env.TIMEOUT); // 如果沒設定會是 NaN！
```

**使用 unifiedConfig：**
```typescript
const port = config.server.port; // number，保證有值
const timeout = config.timeouts.default; // number，有備用值
```

---

## 配置結構

### UnifiedConfig 介面

```typescript
export interface UnifiedConfig {
    database: {
        host: string;
        port: number;
        username: string;
        password: string;
        database: string;
    };
    server: {
        port: number;
        sessionSecret: string;
    };
    tokens: {
        jwt: string;
        inactivity: string;
        internal: string;
    };
    keycloak: {
        realm: string;
        client: string;
        baseUrl: string;
        secret: string;
    };
    aws: {
        region: string;
        emailQueueUrl: string;
        accessKeyId: string;
        secretAccessKey: string;
    };
    sentry: {
        dsn: string;
        environment: string;
        tracesSampleRate: number;
    };
    // ... 更多區塊
}
```

### 實作模式

**檔案：** `/blog-api/src/config/unifiedConfig.ts`

```typescript
import * as fs from 'fs';
import * as path from 'path';
import * as ini from 'ini';

const configPath = path.join(__dirname, '../../config.ini');
const iniConfig = ini.parse(fs.readFileSync(configPath, 'utf-8'));

export const config: UnifiedConfig = {
    database: {
        host: iniConfig.database?.host || process.env.DB_HOST || 'localhost',
        port: parseInt(iniConfig.database?.port || process.env.DB_PORT || '3306'),
        username: iniConfig.database?.username || process.env.DB_USER || 'root',
        password: iniConfig.database?.password || process.env.DB_PASSWORD || '',
        database: iniConfig.database?.database || process.env.DB_NAME || 'blog_dev',
    },
    server: {
        port: parseInt(iniConfig.server?.port || process.env.PORT || '3002'),
        sessionSecret: iniConfig.server?.sessionSecret || process.env.SESSION_SECRET || 'dev-secret',
    },
    // ... 更多配置
};

// 驗證關鍵配置
if (!config.tokens.jwt) {
    throw new Error('JWT secret not configured!');
}
```

**重點：**
- 優先從 config.ini 讀取
- 回退到 process.env
- 開發環境的預設值
- 啟動時驗證
- 型別安全的存取

---

## 環境特定配置

### config.ini 結構

```ini
[database]
host = localhost
port = 3306
username = root
password = password1
database = blog_dev

[server]
port = 3002
sessionSecret = your-secret-here

[tokens]
jwt = your-jwt-secret
inactivity = 30m
internal = internal-api-token

[keycloak]
realm = myapp
client = myapp-client
baseUrl = http://localhost:8080
secret = keycloak-client-secret

[sentry]
dsn = https://your-sentry-dsn
environment = development
tracesSampleRate = 0.1
```

### 環境變數覆寫

```bash
# .env 檔案（選擇性覆寫）
DB_HOST=production-db.example.com
DB_PASSWORD=secure-password
PORT=80
```

**優先順序：**
1. config.ini（最高優先）
2. process.env 變數
3. 硬編碼預設值（最低優先）

---

## 密鑰管理

### 不要提交密鑰

```gitignore
# .gitignore
config.ini
.env
sentry.ini
*.pem
*.key
```

### 在正式環境使用環境變數

```typescript
// 開發環境：config.ini
// 正式環境：環境變數

export const config: UnifiedConfig = {
    database: {
        password: process.env.DB_PASSWORD || iniConfig.database?.password || '',
    },
    tokens: {
        jwt: process.env.JWT_SECRET || iniConfig.tokens?.jwt || '',
    },
};
```

---

## 遷移指南

### 找出所有 process.env 使用

```bash
grep -r "process.env" blog-api/src/ --include="*.ts" | wc -l
```

### 遷移範例

**之前：**
```typescript
// 散落在程式碼各處
const timeout = parseInt(process.env.OPENID_HTTP_TIMEOUT_MS || '15000');
const keycloakUrl = process.env.KEYCLOAK_BASE_URL;
const jwtSecret = process.env.JWT_SECRET;
```

**之後：**
```typescript
import { config } from './config/unifiedConfig';

const timeout = config.keycloak.timeout;
const keycloakUrl = config.keycloak.baseUrl;
const jwtSecret = config.tokens.jwt;
```

**優點：**
- 型別安全
- 集中管理
- 易於測試
- 啟動時驗證

---

**相關檔案：**
- [SKILL.md](SKILL.md)
- [testing-guide.md](testing-guide.md)
