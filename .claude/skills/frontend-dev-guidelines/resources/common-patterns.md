# 常見模式

表單、身份驗證、DataGrid、對話框及其他常見 UI 元素的常用模式。

---

## 使用 useAuth 進行身份驗證

### 取得當前使用者

```typescript
import { useAuth } from '@/hooks/useAuth';

export const MyComponent: React.FC = () => {
    const { user } = useAuth();

    // Available properties:
    // - user.id: string
    // - user.email: string
    // - user.username: string
    // - user.roles: string[]

    return (
        <div>
            <p>Logged in as: {user.email}</p>
            <p>Username: {user.username}</p>
            <p>Roles: {user.roles.join(', ')}</p>
        </div>
    );
};
```

**絕不要直接呼叫 API 進行身份驗證** - 請一律使用 `useAuth` hook。

---

## 使用 React Hook Form 建立表單

### 基本表單

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { TextField, Button } from '@mui/material';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

// Zod schema for validation
const formSchema = z.object({
    username: z.string().min(3, 'Username must be at least 3 characters'),
    email: z.string().email('Invalid email address'),
    age: z.number().min(18, 'Must be 18 or older'),
});

type FormData = z.infer<typeof formSchema>;

export const MyForm: React.FC = () => {
    const { showSuccess, showError } = useMuiSnackbar();

    const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
        resolver: zodResolver(formSchema),
        defaultValues: {
            username: '',
            email: '',
            age: 18,
        },
    });

    const onSubmit = async (data: FormData) => {
        try {
            await api.submitForm(data);
            showSuccess('Form submitted successfully');
        } catch (error) {
            showError('Failed to submit form');
        }
    };

    return (
        <form onSubmit={handleSubmit(onSubmit)}>
            <TextField
                {...register('username')}
                label='Username'
                error={!!errors.username}
                helperText={errors.username?.message}
            />

            <TextField
                {...register('email')}
                label='Email'
                error={!!errors.email}
                helperText={errors.email?.message}
                type='email'
            />

            <TextField
                {...register('age', { valueAsNumber: true })}
                label='Age'
                error={!!errors.age}
                helperText={errors.age?.message}
                type='number'
            />

            <Button type='submit' variant='contained'>
                Submit
            </Button>
        </form>
    );
};
```

---

## 對話框元件模式

### 標準對話框結構

根據 BEST_PRACTICES.md - 所有對話框都應包含：
- 標題中的圖示
- 關閉按鈕 (X)
- 底部的操作按鈕

```typescript
import { Dialog, DialogTitle, DialogContent, DialogActions, Button, IconButton } from '@mui/material';
import { Close, Info } from '@mui/icons-material';

interface MyDialogProps {
    open: boolean;
    onClose: () => void;
    onConfirm: () => void;
}

export const MyDialog: React.FC<MyDialogProps> = ({ open, onClose, onConfirm }) => {
    return (
        <Dialog open={open} onClose={onClose} maxWidth='sm' fullWidth>
            <DialogTitle>
                <Box sx={{ display: 'flex', alignItems: 'center', justifyContent: 'space-between' }}>
                    <Box sx={{ display: 'flex', alignItems: 'center', gap: 1 }}>
                        <Info color='primary' />
                        Dialog Title
                    </Box>
                    <IconButton onClick={onClose} size='small'>
                        <Close />
                    </IconButton>
                </Box>
            </DialogTitle>

            <DialogContent>
                {/* Content here */}
            </DialogContent>

            <DialogActions>
                <Button onClick={onClose}>Cancel</Button>
                <Button onClick={onConfirm} variant='contained'>
                    Confirm
                </Button>
            </DialogActions>
        </Dialog>
    );
};
```

---

## DataGrid 包裝器模式

### 包裝器元件規範

根據 BEST_PRACTICES.md - DataGrid 包裝器應接受：

**必要的 Props：**
- `rows`：資料陣列
- `columns`：欄位定義
- 載入/錯誤狀態

**選擇性的 Props：**
- 工具列元件
- 自訂操作
- 初始狀態

```typescript
import { DataGridPro } from '@mui/x-data-grid-pro';
import type { GridColDef } from '@mui/x-data-grid-pro';

interface DataGridWrapperProps {
    rows: any[];
    columns: GridColDef[];
    loading?: boolean;
    toolbar?: React.ReactNode;
    onRowClick?: (row: any) => void;
}

export const DataGridWrapper: React.FC<DataGridWrapperProps> = ({
    rows,
    columns,
    loading = false,
    toolbar,
    onRowClick,
}) => {
    return (
        <DataGridPro
            rows={rows}
            columns={columns}
            loading={loading}
            slots={{ toolbar: toolbar ? () => toolbar : undefined }}
            onRowClick={(params) => onRowClick?.(params.row)}
            // Standard configuration
            pagination
            pageSizeOptions={[25, 50, 100]}
            initialState={{
                pagination: { paginationModel: { pageSize: 25 } },
            }}
        />
    );
};
```

---

## Mutation 模式

### 更新並使快取失效

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

export const useUpdateEntity = () => {
    const queryClient = useQueryClient();
    const { showSuccess, showError } = useMuiSnackbar();

    return useMutation({
        mutationFn: ({ id, data }: { id: number; data: any }) =>
            api.updateEntity(id, data),

        onSuccess: (result, variables) => {
            // Invalidate affected queries
            queryClient.invalidateQueries({ queryKey: ['entity', variables.id] });
            queryClient.invalidateQueries({ queryKey: ['entities'] });

            showSuccess('Entity updated');
        },

        onError: () => {
            showError('Failed to update entity');
        },
    });
};

// Usage
const updateEntity = useUpdateEntity();

const handleSave = () => {
    updateEntity.mutate({ id: 123, data: { name: 'New Name' } });
};
```

---

## 狀態管理模式

### TanStack Query 處理伺服器狀態（主要方式）

使用 TanStack Query 處理**所有伺服器資料**：
- 資料取得：useSuspenseQuery
- 資料變更：useMutation
- 快取：自動
- 同步：內建

```typescript
// ✅ 正確 - 使用 TanStack Query 處理伺服器資料
const { data: users } = useSuspenseQuery({
    queryKey: ['users'],
    queryFn: () => userApi.getUsers(),
});
```

### 使用 useState 處理 UI 狀態

使用 `useState` **僅處理本地 UI 狀態**：
- 表單輸入（非受控）
- 模態框開啟/關閉
- 已選擇的分頁
- 暫時性的 UI 標記

```typescript
// ✅ 正確 - 使用 useState 處理 UI 狀態
const [modalOpen, setModalOpen] = useState(false);
const [selectedTab, setSelectedTab] = useState(0);
```

### 使用 Zustand 處理全域客戶端狀態（最小化使用）

使用 Zustand **僅處理全域客戶端狀態**：
- 主題偏好設定
- 側邊欄收合狀態
- 使用者偏好設定（非來自伺服器）

```typescript
import { create } from 'zustand';

interface AppState {
    sidebarOpen: boolean;
    toggleSidebar: () => void;
}

export const useAppState = create<AppState>((set) => ({
    sidebarOpen: true,
    toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
}));
```

**避免 prop drilling** - 改用 context 或 Zustand。

---

## 總結

**常見模式：**
- ✅ 使用 useAuth hook 取得當前使用者（id、email、roles、username）
- ✅ 使用 React Hook Form + Zod 建立表單
- ✅ 包含圖示與關閉按鈕的對話框
- ✅ DataGrid 包裝器規範
- ✅ 包含快取失效的 Mutation
- ✅ 使用 TanStack Query 處理伺服器狀態
- ✅ 使用 useState 處理 UI 狀態
- ✅ 使用 Zustand 處理全域客戶端狀態（最小化使用）

**另請參考：**
- [data-fetching.md](data-fetching.md) - TanStack Query 模式
- [component-patterns.md](component-patterns.md) - 元件結構
- [loading-and-error-states.md](loading-and-error-states.md) - 錯誤處理
