# 資料取得模式

使用 TanStack Query 搭配 Suspense 邊界、快取優先策略與集中式 API 服務的現代資料取得方式。

---

## 主要模式：useSuspenseQuery

### 為什麼使用 useSuspenseQuery？

對於**所有新元件**，請使用 `useSuspenseQuery` 而非一般的 `useQuery`：

**好處：**
- 不需要檢查 `isLoading`
- 與 Suspense 邊界整合
- 元件程式碼更簡潔
- 一致的載入使用者體驗
- 透過 error boundaries 有更好的錯誤處理

### 基本模式

```typescript
import { useSuspenseQuery } from '@tanstack/react-query';
import { myFeatureApi } from '../api/myFeatureApi';

export const MyComponent: React.FC<Props> = ({ id }) => {
    // 不需要 isLoading - Suspense 會處理！
    const { data } = useSuspenseQuery({
        queryKey: ['myEntity', id],
        queryFn: () => myFeatureApi.getEntity(id),
    });

    // data 在這裡永遠有定義（不是 undefined | Data）
    return <div>{data.name}</div>;
};

// 用 Suspense boundary 包裝
<SuspenseLoader>
    <MyComponent id={123} />
</SuspenseLoader>
```

### useSuspenseQuery vs useQuery

| 功能 | useSuspenseQuery | useQuery |
|---------|------------------|----------|
| 載入狀態 | 由 Suspense 處理 | 手動檢查 `isLoading` |
| 資料型別 | 永遠有定義 | `Data \| undefined` |
| 搭配使用 | Suspense 邊界 | 傳統元件 |
| 建議用於 | **新元件** | 僅限舊程式碼 |
| 錯誤處理 | Error boundaries | 手動錯誤狀態 |

**何時使用一般 useQuery：**
- 維護舊程式碼
- 非常簡單且不使用 Suspense 的情況
- 需要背景更新的輪詢

**對於新元件：永遠優先使用 useSuspenseQuery**

---

## 快取優先策略

### 快取優先模式範例

**智慧快取**透過先檢查 React Query 快取來減少 API 呼叫：

```typescript
import { useSuspenseQuery, useQueryClient } from '@tanstack/react-query';
import { postApi } from '../api/postApi';

export function useSuspensePost(postId: number) {
    const queryClient = useQueryClient();

    return useSuspenseQuery({
        queryKey: ['post', postId],
        queryFn: async () => {
            // 策略 1：先從列表快取取得
            const cachedListData = queryClient.getQueryData<{ posts: Post[] }>([
                'posts',
                'list'
            ]);

            if (cachedListData?.posts) {
                const cachedPost = cachedListData.posts.find(
                    (post) => post.id === postId
                );

                if (cachedPost) {
                    return cachedPost;  // 從快取回傳！
                }
            }

            // 策略 2：不在快取中，從 API 取得
            return postApi.getPost(postId);
        },
        staleTime: 5 * 60 * 1000,      // 視為新鮮資料 5 分鐘
        gcTime: 10 * 60 * 1000,         // 保留在快取 10 分鐘
        refetchOnWindowFocus: false,    // 聚焦時不重新取得
    });
}
```

**重點：**
- 在 API 呼叫之前先檢查網格/列表快取
- 避免重複請求
- `staleTime`：資料被視為新鮮的時間
- `gcTime`：未使用的資料保留在快取中的時間
- `refetchOnWindowFocus: false`：使用者偏好設定

---

## 平行資料取得

### useSuspenseQueries

當需要取得多個獨立資源時：

```typescript
import { useSuspenseQueries } from '@tanstack/react-query';

export const MyComponent: React.FC = () => {
    const [userQuery, settingsQuery, preferencesQuery] = useSuspenseQueries({
        queries: [
            {
                queryKey: ['user'],
                queryFn: () => userApi.getCurrentUser(),
            },
            {
                queryKey: ['settings'],
                queryFn: () => settingsApi.getSettings(),
            },
            {
                queryKey: ['preferences'],
                queryFn: () => preferencesApi.getPreferences(),
            },
        ],
    });

    // 所有資料都可用，Suspense 處理載入
    const user = userQuery.data;
    const settings = settingsQuery.data;
    const preferences = preferencesQuery.data;

    return <Display user={user} settings={settings} prefs={preferences} />;
};
```

**好處：**
- 所有查詢平行執行
- 單一 Suspense 邊界
- 型別安全的結果

---

## Query Keys 組織

### 命名慣例

```typescript
// 實體列表
['entities', blogId]
['entities', blogId, 'summary']    // 帶檢視模式
['entities', blogId, 'flat']

// 單一實體
['entity', blogId, entityId]

// 相關資料
['entity', entityId, 'history']
['entity', entityId, 'comments']

// 使用者相關
['user', userId, 'profile']
['user', userId, 'permissions']
```

**規則：**
- 以實體名稱開頭（列表用複數，單一用單數）
- 包含 ID 以提高特定性
- 在最後加上檢視模式/關聯
- 在整個應用程式中保持一致

### Query Key 範例

```typescript
// 來自 useSuspensePost.ts
queryKey: ['post', blogId, postId]
queryKey: ['posts-v2', blogId, 'summary']

// 失效模式
queryClient.invalidateQueries({ queryKey: ['post', blogId] });  // 表單的所有貼文
queryClient.invalidateQueries({ queryKey: ['post'] });          // 所有貼文
```

---

## API 服務層模式

### 檔案結構

為每個功能建立集中式 API 服務：

```
features/
  my-feature/
    api/
      myFeatureApi.ts    # 服務層
```

### 服務模式（來自 postApi.ts）

```typescript
/**
 * my-feature 操作的集中式 API 服務
 * 使用 apiClient 以保持一致的錯誤處理
 */
import apiClient from '@/lib/apiClient';
import type { MyEntity, UpdatePayload } from '../types';

export const myFeatureApi = {
    /**
     * 取得單一實體
     */
    getEntity: async (blogId: number, entityId: number): Promise<MyEntity> => {
        const { data } = await apiClient.get(
            `/blog/entities/${blogId}/${entityId}`
        );
        return data;
    },

    /**
     * 取得表單的所有實體
     */
    getEntities: async (blogId: number, view: 'summary' | 'flat'): Promise<MyEntity[]> => {
        const { data } = await apiClient.get(
            `/blog/entities/${blogId}`,
            { params: { view } }
        );
        return data.rows;
    },

    /**
     * 更新實體
     */
    updateEntity: async (
        blogId: number,
        entityId: number,
        payload: UpdatePayload
    ): Promise<MyEntity> => {
        const { data } = await apiClient.put(
            `/blog/entities/${blogId}/${entityId}`,
            payload
        );
        return data;
    },

    /**
     * 刪除實體
     */
    deleteEntity: async (blogId: number, entityId: number): Promise<void> => {
        await apiClient.delete(`/blog/entities/${blogId}/${entityId}`);
    },
};
```

**重點：**
- 匯出帶有方法的單一物件
- 使用 `apiClient`（來自 `@/lib/apiClient` 的 axios 實例）
- 參數和回傳值的型別安全
- 每個方法都有 JSDoc 註解
- 集中式錯誤處理（由 apiClient 處理）

---

## 路由格式規則（重要）

### 正確格式

```typescript
// ✅ 正確 - 直接服務路徑
await apiClient.get('/blog/posts/123');
await apiClient.post('/projects/create', data);
await apiClient.put('/users/update/456', updates);
await apiClient.get('/email/templates');

// ❌ 錯誤 - 不要加 /api/ 前綴
await apiClient.get('/api/blog/posts/123');  // 錯誤！
await apiClient.post('/api/projects/create', data); // 錯誤！
```

**微服務路由：**
- Form 服務：`/blog/*`
- Projects 服務：`/projects/*`
- Email 服務：`/email/*`
- Users 服務：`/users/*`

**原因：** API 路由由 proxy 設定處理，不需要 `/api/` 前綴。

---

## Mutations

### 基本 Mutation 模式

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { myFeatureApi } from '../api/myFeatureApi';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

export const MyComponent: React.FC = () => {
    const queryClient = useQueryClient();
    const { showSuccess, showError } = useMuiSnackbar();

    const updateMutation = useMutation({
        mutationFn: (payload: UpdatePayload) =>
            myFeatureApi.updateEntity(blogId, entityId, payload),

        onSuccess: () => {
            // 失效並重新取得
            queryClient.invalidateQueries({
                queryKey: ['entity', blogId, entityId]
            });
            showSuccess('Entity updated successfully');
        },

        onError: (error) => {
            showError('Failed to update entity');
            console.error('Update error:', error);
        },
    });

    const handleUpdate = () => {
        updateMutation.mutate({ name: 'New Name' });
    };

    return (
        <Button
            onClick={handleUpdate}
            disabled={updateMutation.isPending}
        >
            {updateMutation.isPending ? 'Updating...' : 'Update'}
        </Button>
    );
};
```

### 樂觀更新

```typescript
const updateMutation = useMutation({
    mutationFn: (payload) => myFeatureApi.update(id, payload),

    // 樂觀更新
    onMutate: async (newData) => {
        // 取消進行中的重新取得
        await queryClient.cancelQueries({ queryKey: ['entity', id] });

        // 快照目前值
        const previousData = queryClient.getQueryData(['entity', id]);

        // 樂觀更新
        queryClient.setQueryData(['entity', id], (old) => ({
            ...old,
            ...newData,
        }));

        // 回傳回滾函式
        return { previousData };
    },

    // 錯誤時回滾
    onError: (err, newData, context) => {
        queryClient.setQueryData(['entity', id], context.previousData);
        showError('Update failed');
    },

    // 成功或錯誤後重新取得
    onSettled: () => {
        queryClient.invalidateQueries({ queryKey: ['entity', id] });
    },
});
```

---

## 進階 Query 模式

### 預取

```typescript
export function usePrefetchEntity() {
    const queryClient = useQueryClient();

    return (blogId: number, entityId: number) => {
        return queryClient.prefetchQuery({
            queryKey: ['entity', blogId, entityId],
            queryFn: () => myFeatureApi.getEntity(blogId, entityId),
            staleTime: 5 * 60 * 1000,
        });
    };
}

// 使用方式：滑鼠移入時預取
<div onMouseEnter={() => prefetch(blogId, id)}>
    <Link to={`/entity/${id}`}>View</Link>
</div>
```

### 存取快取而不取得資料

```typescript
export function useEntityFromCache(blogId: number, entityId: number) {
    const queryClient = useQueryClient();

    // 從快取取得，若不存在則不取得
    const directCache = queryClient.getQueryData<MyEntity>(['entity', blogId, entityId]);

    if (directCache) return directCache;

    // 嘗試網格快取
    const gridCache = queryClient.getQueryData<{ rows: MyEntity[] }>(['entities-v2', blogId]);

    return gridCache?.rows.find(row => row.id === entityId);
}
```

### 相依查詢

```typescript
// 先取得使用者，再取得使用者的設定
const { data: user } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => userApi.getUser(userId),
});

const { data: settings } = useSuspenseQuery({
    queryKey: ['user', userId, 'settings'],
    queryFn: () => settingsApi.getUserSettings(user.id),
    // 由於 Suspense 會自動等待 user 載入
});
```

---

## API Client 設定

### 使用 apiClient

```typescript
import apiClient from '@/lib/apiClient';

// apiClient 是已設定的 axios 實例
// 自動包含：
// - Base URL 設定
// - 基於 Cookie 的認證
// - 錯誤攔截器
// - 回應轉換器
```

**不要建立新的 axios 實例** - 使用 apiClient 以保持一致性。

---

## 查詢中的錯誤處理

### onError 回呼

```typescript
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

const { showError } = useMuiSnackbar();

const { data } = useSuspenseQuery({
    queryKey: ['entity', id],
    queryFn: () => myFeatureApi.getEntity(id),

    // 處理錯誤
    onError: (error) => {
        showError('Failed to load entity');
        console.error('Load error:', error);
    },
});
```

### Error Boundaries

與 Error Boundaries 結合以進行完整的錯誤處理：

```typescript
import { ErrorBoundary } from 'react-error-boundary';

<ErrorBoundary
    fallback={<ErrorDisplay />}
    onError={(error) => console.error(error)}
>
    <SuspenseLoader>
        <ComponentWithSuspenseQuery />
    </SuspenseLoader>
</ErrorBoundary>
```

---

## 完整範例

### 範例 1：簡單實體取得

```typescript
import React from 'react';
import { useSuspenseQuery } from '@tanstack/react-query';
import { Box, Typography } from '@mui/material';
import { userApi } from '../api/userApi';

interface UserProfileProps {
    userId: string;
}

export const UserProfile: React.FC<UserProfileProps> = ({ userId }) => {
    const { data: user } = useSuspenseQuery({
        queryKey: ['user', userId],
        queryFn: () => userApi.getUser(userId),
        staleTime: 5 * 60 * 1000,
    });

    return (
        <Box>
            <Typography variant='h5'>{user.name}</Typography>
            <Typography>{user.email}</Typography>
        </Box>
    );
};

// 搭配 Suspense 使用
<SuspenseLoader>
    <UserProfile userId='123' />
</SuspenseLoader>
```

### 範例 2：快取優先策略

```typescript
import { useSuspenseQuery, useQueryClient } from '@tanstack/react-query';
import { postApi } from '../api/postApi';
import type { Post } from '../types';

/**
 * 具有快取優先策略的 Hook
 * 在 API 呼叫前先檢查網格快取
 */
export function useSuspensePost(blogId: number, postId: number) {
    const queryClient = useQueryClient();

    return useSuspenseQuery<Post, Error>({
        queryKey: ['post', blogId, postId],
        queryFn: async () => {
            // 1. 先檢查網格快取
            const gridCache = queryClient.getQueryData<{ rows: Post[] }>([
                'posts-v2',
                blogId,
                'summary'
            ]) || queryClient.getQueryData<{ rows: Post[] }>([
                'posts-v2',
                blogId,
                'flat'
            ]);

            if (gridCache?.rows) {
                const cached = gridCache.rows.find(row => row.S_ID === postId);
                if (cached) {
                    return cached;  // 重複使用網格資料
                }
            }

            // 2. 不在快取中，直接取得
            return postApi.getPost(blogId, postId);
        },
        staleTime: 5 * 60 * 1000,
        gcTime: 10 * 60 * 1000,
        refetchOnWindowFocus: false,
    });
}
```

**好處：**
- 避免重複的 API 呼叫
- 如果已載入則立即取得資料
- 若未快取則回退到 API

### 範例 3：平行取得

```typescript
import { useSuspenseQueries } from '@tanstack/react-query';

export const Dashboard: React.FC = () => {
    const [statsQuery, projectsQuery, notificationsQuery] = useSuspenseQueries({
        queries: [
            {
                queryKey: ['stats'],
                queryFn: () => statsApi.getStats(),
            },
            {
                queryKey: ['projects', 'active'],
                queryFn: () => projectsApi.getActiveProjects(),
            },
            {
                queryKey: ['notifications', 'unread'],
                queryFn: () => notificationsApi.getUnread(),
            },
        ],
    });

    return (
        <Box>
            <StatsCard data={statsQuery.data} />
            <ProjectsList projects={projectsQuery.data} />
            <Notifications items={notificationsQuery.data} />
        </Box>
    );
};
```

---

## 帶快取失效的 Mutations

### 更新 Mutation

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { postApi } from '../api/postApi';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

export const useUpdatePost = () => {
    const queryClient = useQueryClient();
    const { showSuccess, showError } = useMuiSnackbar();

    return useMutation({
        mutationFn: ({ blogId, postId, data }: UpdateParams) =>
            postApi.updatePost(blogId, postId, data),

        onSuccess: (data, variables) => {
            // 使特定貼文失效
            queryClient.invalidateQueries({
                queryKey: ['post', variables.blogId, variables.postId]
            });

            // 使列表失效以重新整理網格
            queryClient.invalidateQueries({
                queryKey: ['posts-v2', variables.blogId]
            });

            showSuccess('Post updated');
        },

        onError: (error) => {
            showError('Failed to update post');
            console.error('Update error:', error);
        },
    });
};

// 使用方式
const updatePost = useUpdatePost();

const handleSave = () => {
    updatePost.mutate({
        blogId: 123,
        postId: 456,
        data: { responses: { '101': 'value' } }
    });
};
```

### 刪除 Mutation

```typescript
export const useDeletePost = () => {
    const queryClient = useQueryClient();
    const { showSuccess, showError } = useMuiSnackbar();

    return useMutation({
        mutationFn: ({ blogId, postId }: DeleteParams) =>
            postApi.deletePost(blogId, postId),

        onSuccess: (data, variables) => {
            // 手動從快取移除（樂觀）
            queryClient.setQueryData<{ rows: Post[] }>(
                ['posts-v2', variables.blogId],
                (old) => ({
                    ...old,
                    rows: old?.rows.filter(row => row.S_ID !== variables.postId) || []
                })
            );

            showSuccess('Post deleted');
        },

        onError: (error, variables) => {
            // 回滾 - 重新取得以獲得準確狀態
            queryClient.invalidateQueries({
                queryKey: ['posts-v2', variables.blogId]
            });
            showError('Failed to delete post');
        },
    });
};
```

---

## Query 設定最佳實務

### 預設設定

```typescript
// 在 QueryClientProvider 設定中
const queryClient = new QueryClient({
    defaultOptions: {
        queries: {
            staleTime: 1000 * 60 * 5,        // 5 分鐘
            gcTime: 1000 * 60 * 10,           // 10 分鐘（之前是 cacheTime）
            refetchOnWindowFocus: false,       // 聚焦時不重新取得
            refetchOnMount: false,             // 若為新鮮則掛載時不重新取得
            retry: 1,                          // 失敗的查詢重試一次
        },
    },
});
```

### 個別查詢覆寫

```typescript
// 頻繁變更的資料 - 較短的 staleTime
useSuspenseQuery({
    queryKey: ['notifications', 'unread'],
    queryFn: () => notificationApi.getUnread(),
    staleTime: 30 * 1000,  // 30 秒
});

// 很少變更的資料 - 較長的 staleTime
useSuspenseQuery({
    queryKey: ['form', blogId, 'structure'],
    queryFn: () => formApi.getStructure(blogId),
    staleTime: 30 * 60 * 1000,  // 30 分鐘
});
```

---

## 總結

**現代資料取得秘訣：**

1. **建立 API 服務**：使用 apiClient 在 `features/X/api/XApi.ts`
2. **使用 useSuspenseQuery**：在由 SuspenseLoader 包裝的元件中
3. **快取優先**：在 API 呼叫前檢查網格快取
4. **Query Keys**：一致的命名 ['entity', id]
5. **路由格式**：`/blog/route` 而非 `/api/blog/route`
6. **Mutations**：成功後使用 invalidateQueries
7. **錯誤處理**：onError + useMuiSnackbar
8. **型別安全**：為所有參數和回傳值加上型別

**另見：**
- [component-patterns.md](component-patterns.md) - Suspense 整合
- [loading-and-error-states.md](loading-and-error-states.md) - SuspenseLoader 使用
- [complete-examples.md](complete-examples.md) - 完整實作範例
