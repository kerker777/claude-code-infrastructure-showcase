# 效能優化

用於優化 React 元件效能、避免不必要的重新渲染，以及預防記憶體洩漏的模式。

---

## Memoization 模式

### 使用 useMemo 處理昂貴的運算

```typescript
import { useMemo } from 'react';

export const DataDisplay: React.FC<{ items: Item[], searchTerm: string }> = ({
    items,
    searchTerm,
}) => {
    // ❌ 避免 - 每次渲染都會執行
    const filteredItems = items
        .filter(item => item.name.includes(searchTerm))
        .sort((a, b) => a.name.localeCompare(b.name));

    // ✅ 正確 - 記憶化，只在依賴項改變時重新計算
    const filteredItems = useMemo(() => {
        return items
            .filter(item => item.name.toLowerCase().includes(searchTerm.toLowerCase()))
            .sort((a, b) => a.name.localeCompare(b.name));
    }, [items, searchTerm]);

    return <List items={filteredItems} />;
};
```

**何時使用 useMemo：**
- 過濾/排序大型陣列
- 複雜的計算
- 轉換資料結構
- 昂貴的運算（迴圈、遞迴）

**何時不要使用 useMemo：**
- 簡單的字串串接
- 基本的算術運算
- 過早優化（先做效能分析！）

---

## 使用 useCallback 處理事件處理器

### 問題所在

```typescript
// ❌ 避免 - 每次渲染都建立新函式
export const Parent: React.FC = () => {
    const handleClick = (id: string) => {
        console.log('Clicked:', id);
    };

    // Child 每次 Parent 渲染時都會重新渲染
    // 因為 handleClick 每次都是新的函式參考
    return <Child onClick={handleClick} />;
};
```

### 解決方案

```typescript
import { useCallback } from 'react';

export const Parent: React.FC = () => {
    // ✅ 正確 - 穩定的函式參考
    const handleClick = useCallback((id: string) => {
        console.log('Clicked:', id);
    }, []); // 空依賴 = 函式永不改變

    // Child 只在 props 真正改變時才重新渲染
    return <Child onClick={handleClick} />;
};
```

**何時使用 useCallback：**
- 傳遞給子元件的函式
- 在 useEffect 中作為依賴項的函式
- 傳遞給記憶化元件的函式
- 列表中的事件處理器

**何時不要使用 useCallback：**
- 不傳遞給子元件的事件處理器
- 簡單的內聯處理器：`onClick={() => doSomething()}`

---

## 使用 React.memo 進行元件記憶化

### 基本用法

```typescript
import React from 'react';

interface ExpensiveComponentProps {
    data: ComplexData;
    onAction: () => void;
}

// ✅ 用 React.memo 包裹昂貴的元件
export const ExpensiveComponent = React.memo<ExpensiveComponentProps>(
    function ExpensiveComponent({ data, onAction }) {
        // 複雜的渲染邏輯
        return <ComplexVisualization data={data} />;
    }
);
```

**何時使用 React.memo：**
- 元件頻繁渲染
- 元件渲染成本高
- Props 不常改變
- 元件是列表項目
- DataGrid 的儲存格/渲染器

**何時不要使用 React.memo：**
- Props 本來就經常改變
- 渲染已經很快
- 過早優化

---

## 防抖搜尋

### 使用 use-debounce Hook

```typescript
import { useState } from 'react';
import { useDebounce } from 'use-debounce';
import { useSuspenseQuery } from '@tanstack/react-query';

export const SearchComponent: React.FC = () => {
    const [searchTerm, setSearchTerm] = useState('');

    // 防抖 300ms
    const [debouncedSearchTerm] = useDebounce(searchTerm, 300);

    // 查詢使用防抖後的值
    const { data } = useSuspenseQuery({
        queryKey: ['search', debouncedSearchTerm],
        queryFn: () => api.search(debouncedSearchTerm),
        enabled: debouncedSearchTerm.length > 0,
    });

    return (
        <input
            value={searchTerm}
            onChange={(e) => setSearchTerm(e.target.value)}
            placeholder='Search...'
        />
    );
};
```

**最佳防抖時機：**
- **300-500ms**：搜尋/過濾
- **1000ms**：自動儲存
- **100-200ms**：即時驗證

---

## 預防記憶體洩漏

### 清理 Timeout/Interval

```typescript
import { useEffect, useState } from 'react';

export const MyComponent: React.FC = () => {
    const [count, setCount] = useState(0);

    useEffect(() => {
        // ✅ 正確 - 清理 interval
        const intervalId = setInterval(() => {
            setCount(c => c + 1);
        }, 1000);

        return () => {
            clearInterval(intervalId);  // 清理！
        };
    }, []);

    useEffect(() => {
        // ✅ 正確 - 清理 timeout
        const timeoutId = setTimeout(() => {
            console.log('Delayed action');
        }, 5000);

        return () => {
            clearTimeout(timeoutId);  // 清理！
        };
    }, []);

    return <div>{count}</div>;
};
```

### 清理事件監聽器

```typescript
useEffect(() => {
    const handleResize = () => {
        console.log('Resized');
    };

    window.addEventListener('resize', handleResize);

    return () => {
        window.removeEventListener('resize', handleResize);  // 清理！
    };
}, []);
```

### 使用 Abort Controller 處理 Fetch

```typescript
useEffect(() => {
    const abortController = new AbortController();

    fetch('/api/data', { signal: abortController.signal })
        .then(response => response.json())
        .then(data => setState(data))
        .catch(error => {
            if (error.name === 'AbortError') {
                console.log('Fetch aborted');
            }
        });

    return () => {
        abortController.abort();  // 清理！
    };
}, []);
```

**注意**：使用 TanStack Query 時，這會自動處理。

---

## 表單效能

### 監聽特定欄位（不是全部）

```typescript
import { useForm } from 'react-hook-form';

export const MyForm: React.FC = () => {
    const { register, watch, handleSubmit } = useForm();

    // ❌ 避免 - 監聽所有欄位，任何改變都會重新渲染
    const formValues = watch();

    // ✅ 正確 - 只監聽需要的欄位
    const username = watch('username');
    const email = watch('email');

    // 或監聽多個特定欄位
    const [username, email] = watch(['username', 'email']);

    return (
        <form onSubmit={handleSubmit(onSubmit)}>
            <input {...register('username')} />
            <input {...register('email')} />
            <input {...register('password')} />

            {/* 只在 username/email 改變時重新渲染 */}
            <p>Username: {username}, Email: {email}</p>
        </form>
    );
};
```

---

## 列表渲染優化

### Key Prop 的使用

```typescript
// ✅ 正確 - 穩定且唯一的 key
{items.map(item => (
    <ListItem key={item.id}>
        {item.name}
    </ListItem>
))}

// ❌ 避免 - 使用 index 作為 key（列表改變時不穩定）
{items.map((item, index) => (
    <ListItem key={index}>  // 列表重新排序時會有問題
        {item.name}
    </ListItem>
))}
```

### 記憶化列表項目

```typescript
const ListItem = React.memo<ListItemProps>(({ item, onAction }) => {
    return (
        <Box onClick={() => onAction(item.id)}>
            {item.name}
        </Box>
    );
});

export const List: React.FC<{ items: Item[] }> = ({ items }) => {
    const handleAction = useCallback((id: string) => {
        console.log('Action:', id);
    }, []);

    return (
        <Box>
            {items.map(item => (
                <ListItem
                    key={item.id}
                    item={item}
                    onAction={handleAction}
                />
            ))}
        </Box>
    );
};
```

---

## 避免元件重新初始化

### 問題所在

```typescript
// ❌ 避免 - 每次渲染都重新建立元件
export const Parent: React.FC = () => {
    // 每次渲染都建立新的元件定義！
    const ChildComponent = () => <div>Child</div>;

    return <ChildComponent />;  // 每次渲染都會卸載和重新掛載
};
```

### 解決方案

```typescript
// ✅ 正確 - 定義在外部或使用 useMemo
const ChildComponent: React.FC = () => <div>Child</div>;

export const Parent: React.FC = () => {
    return <ChildComponent />;  // 穩定的元件
};

// ✅ 或者如果是動態的，使用 useMemo
export const Parent: React.FC<{ config: Config }> = ({ config }) => {
    const DynamicComponent = useMemo(() => {
        return () => <div>{config.title}</div>;
    }, [config.title]);

    return <DynamicComponent />;
};
```

---

## 延遲載入大型相依套件

### 程式碼分割

```typescript
// ❌ 避免 - 在最上層匯入大型函式庫
import jsPDF from 'jspdf';  // 大型函式庫立即載入
import * as XLSX from 'xlsx';  // 大型函式庫立即載入

// ✅ 正確 - 需要時才動態匯入
const handleExportPDF = async () => {
    const { jsPDF } = await import('jspdf');
    const doc = new jsPDF();
    // 使用它
};

const handleExportExcel = async () => {
    const XLSX = await import('xlsx');
    // 使用它
};
```

---

## 總結

**效能檢查清單：**
- ✅ `useMemo` 用於昂貴的運算（filter、sort、map）
- ✅ `useCallback` 用於傳遞給子元件的函式
- ✅ `React.memo` 用於昂貴的元件
- ✅ 對搜尋/過濾進行防抖（300-500ms）
- ✅ 在 useEffect 中清理 timeout/interval
- ✅ 監聽特定表單欄位（不是全部）
- ✅ 在列表中使用穩定的 key
- ✅ 延遲載入大型函式庫
- ✅ 使用 React.lazy 進行程式碼分割

**另請參考：**
- [component-patterns.md](component-patterns.md) - 延遲載入
- [data-fetching.md](data-fetching.md) - TanStack Query 優化
- [complete-examples.md](complete-examples.md) - 情境中的效能模式
