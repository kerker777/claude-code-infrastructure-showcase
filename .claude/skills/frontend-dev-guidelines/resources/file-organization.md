# 檔案組織

在應用程式中維護可維護性和可擴展性的前端程式碼，需要適當的檔案和目錄結構。

---

## features/ 與 components/ 的區別

### features/ 目錄

**用途**：具有自己的邏輯、API 和元件的領域特定功能

**何時使用：**
- 功能有多個相關元件
- 功能有自己的 API endpoints
- 功能有領域特定的邏輯
- 功能有自定義的 hooks/工具函式

**範例：**
- `features/posts/` - 專案目錄/文章管理
- `features/blogs/` - 部落格建構和渲染
- `features/auth/` - 驗證流程

**結構：**
```
features/
  my-feature/
    api/
      myFeatureApi.ts         # API service layer
    components/
      MyFeatureMain.tsx       # Main component
      SubComponents/          # Related components
    hooks/
      useMyFeature.ts         # Custom hooks
      useSuspenseMyFeature.ts # Suspense hooks
    helpers/
      myFeatureHelpers.ts     # Utility functions
    types/
      index.ts                # TypeScript types
    index.ts                  # Public exports
```

### components/ 目錄

**用途**：可跨多個功能真正重複使用的元件

**何時使用：**
- 元件在 3 個以上的地方使用
- 元件是通用的（沒有功能特定的邏輯）
- 元件是 UI 原件或模式

**範例：**
- `components/SuspenseLoader/` - Loading wrapper
- `components/CustomAppBar/` - Application header
- `components/ErrorBoundary/` - Error handling
- `components/LoadingOverlay/` - Loading overlay

**結構：**
```
components/
  SuspenseLoader/
    SuspenseLoader.tsx
    SuspenseLoader.test.tsx
  CustomAppBar/
    CustomAppBar.tsx
    CustomAppBar.test.tsx
```

---

## 功能目錄結構（詳細）

### 完整功能範例

基於 `features/posts/` 結構：

```
features/
  posts/
    api/
      postApi.ts              # API service layer (GET, POST, PUT, DELETE)

    components/
      PostTable.tsx           # Main container component
      grids/
        PostDataGrid/
          PostDataGrid.tsx
      drawers/
        ProjectPostDrawer/
          ProjectPostDrawer.tsx
      cells/
        editors/
          TextEditCell.tsx
        renderers/
          DateCell.tsx
      toolbar/
        CustomToolbar.tsx

    hooks/
      usePostQueries.ts       # Regular queries
      useSuspensePost.ts      # Suspense queries
      usePostMutations.ts     # Mutations
      useGridLayout.ts              # Feature-specific hooks

    helpers/
      postHelpers.ts          # Utility functions
      validation.ts                 # Validation logic

    types/
      index.ts                      # TypeScript types/interfaces

    queries/
      postQueries.ts          # Query key factories (optional)

    context/
      PostContext.tsx         # React context (if needed)

    index.ts                        # Public API exports
```

### 子目錄指南

#### api/ 目錄

**用途**：功能的集中式 API 呼叫

**檔案：**
- `{feature}Api.ts` - 主要 API service

**模式：**
```typescript
// features/my-feature/api/myFeatureApi.ts
import apiClient from '@/lib/apiClient';

export const myFeatureApi = {
    getItem: async (id: number) => {
        const { data } = await apiClient.get(`/blog/items/${id}`);
        return data;
    },
    createItem: async (payload) => {
        const { data } = await apiClient.post('/blog/items', payload);
        return data;
    },
};
```

#### components/ 目錄

**用途**：功能特定的元件

**組織方式：**
- 如果少於 5 個元件使用扁平結構
- 如果多於 5 個元件依職責分類使用子目錄

**範例：**
```
components/
  MyFeatureMain.tsx           # Main component
  MyFeatureHeader.tsx         # Supporting components
  MyFeatureFooter.tsx

  # OR with subdirectories:
  containers/
    MyFeatureContainer.tsx
  presentational/
    MyFeatureDisplay.tsx
  blogs/
    MyFeatureBlog.tsx
```

#### hooks/ 目錄

**用途**：功能的自定義 hooks

**命名：**
- `use` 前綴（camelCase）
- 描述性地說明它們的作用

**範例：**
```
hooks/
  useMyFeature.ts               # Main hook
  useSuspenseMyFeature.ts       # Suspense version
  useMyFeatureMutations.ts      # Mutations
  useMyFeatureFilters.ts        # Filters/search
```

#### helpers/ 目錄

**用途**：功能特定的工具函式

**範例：**
```
helpers/
  myFeatureHelpers.ts           # General utilities
  validation.ts                 # Validation logic
  transblogers.ts               # Data transblogations
  constants.ts                  # Constants
```

#### types/ 目錄

**用途**：TypeScript 型別和介面

**檔案：**
```
types/
  index.ts                      # Main types, exported
  internal.ts                   # Internal types (not exported)
```

---

## Import 別名（Vite 設定）

### 可用的別名

來自 `vite.config.ts` 第 180-185 行：

| 別名 | 解析至 | 用途 |
|-------|-------------|---------|
| `@/` | `src/` | 從 src 根目錄的絕對 imports |
| `~types` | `src/types` | 共用的 TypeScript 型別 |
| `~components` | `src/components` | 可重複使用的元件 |
| `~features` | `src/features` | 功能 imports |

### 使用範例

```typescript
// ✅ 推薦 - 使用別名進行絕對 imports
import { apiClient } from '@/lib/apiClient';
import { SuspenseLoader } from '~components/SuspenseLoader';
import { postApi } from '~features/posts/api/postApi';
import type { User } from '~types/user';

// ❌ 避免 - 從深層巢狀使用相對路徑
import { apiClient } from '../../../lib/apiClient';
import { SuspenseLoader } from '../../../components/SuspenseLoader';
```

### 何時使用哪個別名

**@/ (一般用途)**:
- Lib 工具函式：`@/lib/apiClient`
- Hooks：`@/hooks/useAuth`
- 設定檔：`@/config/theme`
- 共用服務：`@/services/authService`

**~types (型別 Imports)**:
```typescript
import type { Post } from '~types/post';
import type { User, UserRole } from '~types/user';
```

**~components (可重複使用的元件)**:
```typescript
import { SuspenseLoader } from '~components/SuspenseLoader';
import { CustomAppBar } from '~components/CustomAppBar';
import { ErrorBoundary } from '~components/ErrorBoundary';
```

**~features (功能 Imports)**:
```typescript
import { postApi } from '~features/posts/api/postApi';
import { useAuth } from '~features/auth/hooks/useAuth';
```

---

## 檔案命名慣例

### 元件

**模式**：PascalCase 搭配 `.tsx` 副檔名

```
MyComponent.tsx
PostDataGrid.tsx
CustomAppBar.tsx
```

**避免：**
- camelCase：`myComponent.tsx` ❌
- kebab-case：`my-component.tsx` ❌
- 全大寫：`MYCOMPONENT.tsx` ❌

### Hooks

**模式**：camelCase 搭配 `use` 前綴，`.ts` 副檔名

```
useMyFeature.ts
useSuspensePost.ts
useAuth.ts
useGridLayout.ts
```

### API Services

**模式**：camelCase 搭配 `Api` 後綴，`.ts` 副檔名

```
myFeatureApi.ts
postApi.ts
userApi.ts
```

### Helpers/工具函式

**模式**：camelCase 搭配描述性名稱，`.ts` 副檔名

```
myFeatureHelpers.ts
validation.ts
transblogers.ts
constants.ts
```

### 型別

**模式**：camelCase，`index.ts` 或描述性名稱

```
types/index.ts
types/post.ts
types/user.ts
```

---

## 何時建立新功能

### 建立新功能的時機：

- 多個相關元件（>3 個）
- 有自己的 API endpoints
- 領域特定的邏輯
- 隨時間會成長
- 跨多個路由重複使用

**範例：** `features/posts/`
- 20 個以上元件
- 自己的 API service
- 複雜的狀態管理
- 在多個路由中使用

### 加入現有功能的時機：

- 與現有功能相關
- 共用相同的 API
- 邏輯上分組
- 擴展現有功能

**範例：**在 posts 功能中新增匯出對話框

### 建立可重複使用元件的時機：

- 跨 3 個以上功能使用
- 通用的，沒有領域邏輯
- 純展示
- 共用模式

**範例：** `components/SuspenseLoader/`

---

## Import 組織

### Import 順序（建議）

```typescript
// 1. React 和 React 相關
import React, { useState, useCallback, useMemo } from 'react';
import { lazy } from 'react';

// 2. 第三方套件（依字母排序）
import { Box, Paper, Button, Grid } from '@mui/material';
import type { SxProps, Theme } from '@mui/material';
import { useSuspenseQuery, useQueryClient } from '@tanstack/react-query';
import { createFileRoute } from '@tanstack/react-router';

// 3. 別名 imports (@ 先，然後 ~)
import { apiClient } from '@/lib/apiClient';
import { useAuth } from '@/hooks/useAuth';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';
import { SuspenseLoader } from '~components/SuspenseLoader';
import { postApi } from '~features/posts/api/postApi';

// 4. 型別 imports（分組）
import type { Post } from '~types/post';
import type { User } from '~types/user';

// 5. 相對 imports（同一功能）
import { MySubComponent } from './MySubComponent';
import { useMyFeature } from '../hooks/useMyFeature';
import { myFeatureHelpers } from '../helpers/myFeatureHelpers';
```

**所有 imports 使用單引號**（專案標準）

---

## Public API 模式

### feature/index.ts

從功能匯出 public API 以實現簡潔的 imports：

```typescript
// features/my-feature/index.ts

// Export main components
export { MyFeatureMain } from './components/MyFeatureMain';
export { MyFeatureHeader } from './components/MyFeatureHeader';

// Export hooks
export { useMyFeature } from './hooks/useMyFeature';
export { useSuspenseMyFeature } from './hooks/useSuspenseMyFeature';

// Export API
export { myFeatureApi } from './api/myFeatureApi';

// Export types
export type { MyFeatureData, MyFeatureConfig } from './types';
```

**使用方式：**
```typescript
// ✅ 從功能 index 簡潔的 import
import { MyFeatureMain, useMyFeature } from '~features/my-feature';

// ❌ 避免深層 imports（但如有需要也可以）
import { MyFeatureMain } from '~features/my-feature/components/MyFeatureMain';
```

---

## 目錄結構視覺化

```
src/
├── features/                    # 領域特定功能
│   ├── posts/
│   │   ├── api/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── helpers/
│   │   ├── types/
│   │   └── index.ts
│   ├── blogs/
│   └── auth/
│
├── components/                  # 可重複使用的元件
│   ├── SuspenseLoader/
│   ├── CustomAppBar/
│   ├── ErrorBoundary/
│   └── LoadingOverlay/
│
├── routes/                      # TanStack Router routes
│   ├── __root.tsx
│   ├── index.tsx
│   ├── project-catalog/
│   │   ├── index.tsx
│   │   └── create/
│   └── blogs/
│
├── hooks/                       # 共用 hooks
│   ├── useAuth.ts
│   ├── useMuiSnackbar.ts
│   └── useDebounce.ts
│
├── lib/                         # 共用工具
│   ├── apiClient.ts
│   └── utils.ts
│
├── types/                       # 共用 TypeScript 型別
│   ├── user.ts
│   ├── post.ts
│   └── common.ts
│
├── config/                      # 設定檔
│   └── theme.ts
│
└── App.tsx                      # Root component
```

---

## 摘要

**關鍵原則：**
1. **features/** 用於領域特定程式碼
2. **components/** 用於真正可重複使用的 UI
3. 使用子目錄：api/、components/、hooks/、helpers/、types/
4. Import 別名用於簡潔的 imports（@/、~types、~components、~features）
5. 一致的命名：PascalCase 元件、camelCase 工具函式
6. 從功能 index.ts 匯出 public API

**另請參閱：**
- [component-patterns.md](component-patterns.md) - 元件結構
- [data-fetching.md](data-fetching.md) - API service 模式
- [complete-examples.md](complete-examples.md) - 完整功能範例
