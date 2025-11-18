---
name: frontend-dev-guidelines
description: Frontend development guidelines for React/TypeScript applications. Modern patterns including Suspense, lazy loading, useSuspenseQuery, file organization with features directory, MUI v7 styling, TanStack Router, performance optimization, and TypeScript best practices. Use when creating components, pages, features, fetching data, styling, routing, or working with frontend code.
---

# Frontend 開發指南

## 目的

現代 React 開發的完整指南，著重於基於 Suspense 的資料抓取、延遲載入、適當的檔案組織以及效能優化。

## 何時使用此技能

- 建立新元件或頁面
- 開發新功能
- 使用 TanStack Query 抓取資料
- 使用 TanStack Router 設定路由
- 使用 MUI v7 為元件設定樣式
- 效能優化
- 組織前端程式碼
- TypeScript 最佳實踐

---

## 快速開始

### 新元件檢查清單

建立元件時，請遵循此檢查清單：

- [ ] 使用 `React.FC<Props>` 模式搭配 TypeScript
- [ ] 如果是大型元件，使用延遲載入：`React.lazy(() => import())`
- [ ] 使用 `<SuspenseLoader>` 包裝以處理載入狀態
- [ ] 使用 `useSuspenseQuery` 抓取資料
- [ ] 使用 import 別名：`@/`、`~types`、`~components`、`~features`
- [ ] 樣式：少於 100 行使用 inline，超過 100 行使用獨立檔案
- [ ] 對傳遞給子元件的事件處理器使用 `useCallback`
- [ ] 在檔案底部使用 default export
- [ ] 不要使用 early return 搭配 loading spinner
- [ ] 使用 `useMuiSnackbar` 顯示使用者通知

### 新功能檢查清單

建立功能時，請設定以下結構：

- [ ] 建立 `features/{feature-name}/` 目錄
- [ ] 建立子目錄：`api/`、`components/`、`hooks/`、`helpers/`、`types/`
- [ ] 建立 API 服務檔案：`api/{feature}Api.ts`
- [ ] 在 `types/` 中設定 TypeScript 型別
- [ ] 在 `routes/{feature-name}/index.tsx` 建立路由
- [ ] 延遲載入功能元件
- [ ] 使用 Suspense 邊界
- [ ] 從功能的 `index.ts` 匯出公開 API

---

## Import 別名快速參考

| 別名 | 解析至 | 範例 |
|-------|-------------|---------|
| `@/` | `src/` | `import { apiClient } from '@/lib/apiClient'` |
| `~types` | `src/types` | `import type { User } from '~types/user'` |
| `~components` | `src/components` | `import { SuspenseLoader } from '~components/SuspenseLoader'` |
| `~features` | `src/features` | `import { authApi } from '~features/auth'` |

定義於：[vite.config.ts](../../vite.config.ts) 第 180-185 行

---

## 常用 Import 速查表

```typescript
// React & Lazy Loading
import React, { useState, useCallback, useMemo } from 'react';
const Heavy = React.lazy(() => import('./Heavy'));

// MUI Components
import { Box, Paper, Typography, Button, Grid } from '@mui/material';
import type { SxProps, Theme } from '@mui/material';

// TanStack Query (Suspense)
import { useSuspenseQuery, useQueryClient } from '@tanstack/react-query';

// TanStack Router
import { createFileRoute } from '@tanstack/react-router';

// Project Components
import { SuspenseLoader } from '~components/SuspenseLoader';

// Hooks
import { useAuth } from '@/hooks/useAuth';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

// Types
import type { Post } from '~types/post';
```

---

## 主題指南

### 🎨 元件模式

**現代 React 元件使用：**
- `React.FC<Props>` 提供型別安全
- `React.lazy()` 進行程式碼分割
- `SuspenseLoader` 處理載入狀態
- 具名 const + default export 模式

**關鍵概念：**
- 延遲載入大型元件（DataGrid、圖表、編輯器）
- 總是使用 Suspense 包裝延遲載入的元件
- 使用 SuspenseLoader 元件（帶淡入動畫）
- 元件結構：Props → Hooks → Handlers → Render → Export

**[📖 完整指南：resources/component-patterns.md](resources/component-patterns.md)**

---

### 📊 資料抓取

**主要模式：useSuspenseQuery**
- 搭配 Suspense 邊界使用
- 快取優先策略（在呼叫 API 前先檢查 grid 快取）
- 取代 `isLoading` 檢查
- 使用泛型提供型別安全

**API 服務層：**
- 建立 `features/{feature}/api/{feature}Api.ts`
- 使用 `apiClient` axios 實例
- 每個功能的集中化方法
- 路由格式：`/form/route`（不是 `/api/form/route`）

**[📖 完整指南：resources/data-fetching.md](resources/data-fetching.md)**

---

### 📁 檔案組織

**features/ vs components/：**
- `features/`：領域特定的（posts、comments、auth）
- `components/`：真正可重用的（SuspenseLoader、CustomAppBar）

**功能子目錄：**
```
features/
  my-feature/
    api/          # API 服務層
    components/   # 功能元件
    hooks/        # 自訂 hooks
    helpers/      # 工具函式
    types/        # TypeScript 型別
```

**[📖 完整指南：resources/file-organization.md](resources/file-organization.md)**

---

### 🎨 樣式設定

**Inline vs 獨立檔案：**
- 少於 100 行：Inline `const styles: Record<string, SxProps<Theme>>`
- 超過 100 行：獨立的 `.styles.ts` 檔案

**主要方法：**
- 對 MUI 元件使用 `sx` prop
- 使用 `SxProps<Theme>` 提供型別安全
- 存取主題：`(theme) => theme.palette.primary.main`

**MUI v7 Grid：**
```typescript
<Grid size={{ xs: 12, md: 6 }}>  // ✅ v7 語法
<Grid xs={12} md={6}>             // ❌ 舊語法
```

**[📖 完整指南：resources/styling-guide.md](resources/styling-guide.md)**

---

### 🛣️ 路由

**TanStack Router - 資料夾架構：**
- 目錄：`routes/my-route/index.tsx`
- 延遲載入元件
- 使用 `createFileRoute`
- 在 loader 中設定麵包屑資料

**範例：**
```typescript
import { createFileRoute } from '@tanstack/react-router';
import { lazy } from 'react';

const MyPage = lazy(() => import('@/features/my-feature/components/MyPage'));

export const Route = createFileRoute('/my-route/')({
    component: MyPage,
    loader: () => ({ crumb: 'My Route' }),
});
```

**[📖 完整指南：resources/routing-guide.md](resources/routing-guide.md)**

---

### ⏳ 載入與錯誤狀態

**重要規則：不要使用 Early Return**

```typescript
// ❌ 絕對不要 - 會造成版面位移
if (isLoading) {
    return <LoadingSpinner />;
}

// ✅ 永遠使用 - 保持版面一致
<SuspenseLoader>
    <Content />
</SuspenseLoader>
```

**原因：**防止累積版面位移（CLS），提供更好的使用者體驗

**錯誤處理：**
- 使用 `useMuiSnackbar` 提供使用者回饋
- 絕對不要使用 `react-toastify`
- 使用 TanStack Query 的 `onError` 回呼

**[📖 完整指南：resources/loading-and-error-states.md](resources/loading-and-error-states.md)**

---

### ⚡ 效能

**優化模式：**
- `useMemo`：昂貴的計算（filter、sort、map）
- `useCallback`：傳遞給子元件的事件處理器
- `React.memo`：昂貴的元件
- 防抖搜尋（300-500ms）
- 記憶體洩漏預防（在 useEffect 中清理）

**[📖 完整指南：resources/performance.md](resources/performance.md)**

---

### 📘 TypeScript

**標準：**
- 嚴格模式，不使用 `any` 型別
- 函式要明確宣告回傳型別
- 型別 import：`import type { User } from '~types/user'`
- 元件 prop 介面要加上 JSDoc

**[📖 完整指南：resources/typescript-standards.md](resources/typescript-standards.md)**

---

### 🔧 常見模式

**涵蓋主題：**
- React Hook Form 搭配 Zod 驗證
- DataGrid wrapper 契約
- Dialog 元件標準
- 使用 `useAuth` hook 取得當前使用者
- 搭配快取失效的 Mutation 模式

**[📖 完整指南：resources/common-patterns.md](resources/common-patterns.md)**

---

### 📚 完整範例

**完整的工作範例：**
- 包含所有模式的現代元件
- 完整的功能結構
- API 服務層
- 帶延遲載入的路由
- Suspense + useSuspenseQuery
- 帶驗證的表單

**[📖 完整指南：resources/complete-examples.md](resources/complete-examples.md)**

---

## 導覽指南

| 需要... | 閱讀此資源 |
|------------|-------------------|
| 建立元件 | [component-patterns.md](resources/component-patterns.md) |
| 抓取資料 | [data-fetching.md](resources/data-fetching.md) |
| 組織檔案/資料夾 | [file-organization.md](resources/file-organization.md) |
| 設定元件樣式 | [styling-guide.md](resources/styling-guide.md) |
| 設定路由 | [routing-guide.md](resources/routing-guide.md) |
| 處理載入/錯誤 | [loading-and-error-states.md](resources/loading-and-error-states.md) |
| 優化效能 | [performance.md](resources/performance.md) |
| TypeScript 型別 | [typescript-standards.md](resources/typescript-standards.md) |
| 表單/認證/DataGrid | [common-patterns.md](resources/common-patterns.md) |
| 查看完整範例 | [complete-examples.md](resources/complete-examples.md) |

---

## 核心原則

1. **延遲載入所有大型元件**：路由、DataGrid、圖表、編輯器
2. **使用 Suspense 處理載入**：使用 SuspenseLoader，不要用 early return
3. **useSuspenseQuery**：新程式碼的主要資料抓取模式
4. **功能要有組織**：包含 api/、components/、hooks/、helpers/ 子目錄
5. **依據大小決定樣式位置**：少於 100 行用 inline，超過 100 行用獨立檔案
6. **使用 Import 別名**：使用 @/、~types、~components、~features
7. **不要 Early Return**：防止版面位移
8. **useMuiSnackbar**：所有使用者通知都用這個

---

## 快速參考：檔案結構

```
src/
  features/
    my-feature/
      api/
        myFeatureApi.ts       # API 服務
      components/
        MyFeature.tsx         # 主元件
        SubComponent.tsx      # 相關元件
      hooks/
        useMyFeature.ts       # 自訂 hooks
        useSuspenseMyFeature.ts  # Suspense hooks
      helpers/
        myFeatureHelpers.ts   # 工具函式
      types/
        index.ts              # TypeScript 型別
      index.ts                # 公開匯出

  components/
    SuspenseLoader/
      SuspenseLoader.tsx      # 可重用的載入器
    CustomAppBar/
      CustomAppBar.tsx        # 可重用的應用程式列

  routes/
    my-route/
      index.tsx               # 路由元件
      create/
        index.tsx             # 巢狀路由
```

---

## 現代元件範本（快速複製）

```typescript
import React, { useState, useCallback } from 'react';
import { Box, Paper } from '@mui/material';
import { useSuspenseQuery } from '@tanstack/react-query';
import { featureApi } from '../api/featureApi';
import type { FeatureData } from '~types/feature';

interface MyComponentProps {
    id: number;
    onAction?: () => void;
}

export const MyComponent: React.FC<MyComponentProps> = ({ id, onAction }) => {
    const [state, setState] = useState<string>('');

    const { data } = useSuspenseQuery({
        queryKey: ['feature', id],
        queryFn: () => featureApi.getFeature(id),
    });

    const handleAction = useCallback(() => {
        setState('updated');
        onAction?.();
    }, [onAction]);

    return (
        <Box sx={{ p: 2 }}>
            <Paper sx={{ p: 3 }}>
                {/* Content */}
            </Paper>
        </Box>
    );
};

export default MyComponent;
```

完整範例請參見 [resources/complete-examples.md](resources/complete-examples.md)

---

## 相關技能

- **error-tracking**：使用 Sentry 進行錯誤追蹤（也適用於前端）
- **backend-dev-guidelines**：前端使用的後端 API 模式

---

**技能狀態**：模組化結構，漸進式載入以達到最佳的上下文管理
