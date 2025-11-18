# 樣式指南

使用 MUI v7 sx prop、inline 樣式和主題整合的現代化樣式寫法。

---

## Inline 與分離樣式

### 判斷門檻

**<100 行：在元件頂部使用 inline 樣式**

```typescript
import type { SxProps, Theme } from '@mui/material';

const componentStyles: Record<string, SxProps<Theme>> = {
    container: {
        p: 2,
        display: 'flex',
        flexDirection: 'column',
    },
    header: {
        mb: 2,
        borderBottom: '1px solid',
        borderColor: 'divider',
    },
    // ... more styles
};

export const MyComponent: React.FC = () => {
    return (
        <Box sx={componentStyles.container}>
            <Box sx={componentStyles.header}>
                <h2>Title</h2>
            </Box>
        </Box>
    );
};
```

**>100 行：使用獨立的 `.styles.ts` 檔案**

```typescript
// MyComponent.styles.ts
import type { SxProps, Theme } from '@mui/material';

export const componentStyles: Record<string, SxProps<Theme>> = {
    container: { ... },
    header: { ... },
    // ... 100+ lines of styles
};

// MyComponent.tsx
import { componentStyles } from './MyComponent.styles';

export const MyComponent: React.FC = () => {
    return <Box sx={componentStyles.container}>...</Box>;
};
```

### 實際範例：UnifiedForm.tsx

**第 48-126 行**：78 行 inline 樣式（可接受）

```typescript
const formStyles: Record<string, SxProps<Theme>> = {
    gridContainer: {
        height: '100%',
        maxHeight: 'calc(100vh - 220px)',
    },
    section: {
        height: '100%',
        maxHeight: 'calc(100vh - 220px)',
        overflow: 'auto',
        p: 4,
    },
    // ... 15 more style objects
};
```

**指導原則**：使用者能接受約 80 行的 inline 樣式。在 100 行左右時自行判斷。

---

## sx Prop 使用模式

### 基本用法

```typescript
<Box sx={{ p: 2, mb: 3, display: 'flex' }}>
    Content
</Box>
```

### 使用主題

```typescript
<Box
    sx={{
        p: 2,
        backgroundColor: (theme) => theme.palette.primary.main,
        color: (theme) => theme.palette.primary.contrastText,
        borderRadius: (theme) => theme.shape.borderRadius,
    }}
>
    Themed Box
</Box>
```

### 響應式樣式

```typescript
<Box
    sx={{
        p: { xs: 1, sm: 2, md: 3 },
        width: { xs: '100%', md: '50%' },
        flexDirection: { xs: 'column', md: 'row' },
    }}
>
    Responsive Layout
</Box>
```

### 偽選擇器

```typescript
<Box
    sx={{
        p: 2,
        '&:hover': {
            backgroundColor: 'rgba(0,0,0,0.05)',
        },
        '&:active': {
            backgroundColor: 'rgba(0,0,0,0.1)',
        },
        '& .child-class': {
            color: 'primary.main',
        },
    }}
>
    Interactive Box
</Box>
```

---

## MUI v7 寫法

### Grid 元件（v7 語法）

```typescript
import { Grid } from '@mui/material';

// ✅ 正確 - v7 語法使用 size prop
<Grid container spacing={2}>
    <Grid size={{ xs: 12, md: 6 }}>
        Left Column
    </Grid>
    <Grid size={{ xs: 12, md: 6 }}>
        Right Column
    </Grid>
</Grid>

// ❌ 錯誤 - 舊的 v6 語法
<Grid container spacing={2}>
    <Grid xs={12} md={6}>  {/* OLD - Don't use */}
        Content
    </Grid>
</Grid>
```

**關鍵變更**：使用 `size={{ xs: 12, md: 6 }}` 而不是 `xs={12} md={6}`

### 響應式 Grid

```typescript
<Grid container spacing={3}>
    <Grid size={{ xs: 12, sm: 6, md: 4, lg: 3 }}>
        Responsive Column
    </Grid>
</Grid>
```

### 巢狀 Grid

```typescript
<Grid container spacing={2}>
    <Grid size={{ xs: 12, md: 8 }}>
        <Grid container spacing={1}>
            <Grid size={{ xs: 12, sm: 6 }}>
                Nested 1
            </Grid>
            <Grid size={{ xs: 12, sm: 6 }}>
                Nested 2
            </Grid>
        </Grid>
    </Grid>

    <Grid size={{ xs: 12, md: 4 }}>
        Sidebar
    </Grid>
</Grid>
```

---

## 型別安全的樣式

### 樣式物件型別

```typescript
import type { SxProps, Theme } from '@mui/material';

// Type-safe styles
const styles: Record<string, SxProps<Theme>> = {
    container: {
        p: 2,
        // Autocomplete and type checking work here
    },
};

// Or individual style
const containerStyle: SxProps<Theme> = {
    p: 2,
    display: 'flex',
};
```

### 主題感知樣式

```typescript
const styles: Record<string, SxProps<Theme>> = {
    primary: {
        color: (theme) => theme.palette.primary.main,
        backgroundColor: (theme) => theme.palette.primary.light,
        '&:hover': {
            backgroundColor: (theme) => theme.palette.primary.dark,
        },
    },
    customSpacing: {
        padding: (theme) => theme.spacing(2),
        margin: (theme) => theme.spacing(1, 2), // top/bottom: 1, left/right: 2
    },
};
```

---

## 不要使用的寫法

### ❌ makeStyles（MUI v4 寫法）

```typescript
// ❌ 避免 - 舊的 Material-UI v4 寫法
import { makeStyles } from '@mui/styles';

const useStyles = makeStyles((theme) => ({
    root: {
        padding: theme.spacing(2),
    },
}));
```

**為何避免**：已棄用，v7 支援不佳

### ❌ styled() 元件

```typescript
// ❌ 避免 - styled-components 寫法
import { styled } from '@mui/material/styles';

const StyledBox = styled(Box)(({ theme }) => ({
    padding: theme.spacing(2),
}));
```

**為何避免**：sx prop 更靈活且不會建立新元件

### ✅ 改用 sx Prop

```typescript
// ✅ 建議用法
<Box
    sx={{
        p: 2,
        backgroundColor: 'primary.main',
    }}
>
    Content
</Box>
```

---

## 程式碼風格規範

### 縮排

**4 個空格**（不是 2 個，也不是 tabs）

```typescript
const styles: Record<string, SxProps<Theme>> = {
    container: {
        p: 2,
        display: 'flex',
        flexDirection: 'column',
    },
};
```

### 引號

**單引號**（專案標準）

```typescript
// ✅ 正確
const color = 'primary.main';
import { Box } from '@mui/material';

// ❌ 錯誤
const color = "primary.main";
import { Box } from "@mui/material";
```

### 結尾逗號

**物件和陣列都要加上結尾逗號**

```typescript
// ✅ 正確
const styles = {
    container: { p: 2 },
    header: { mb: 1 },  // Trailing comma
};

const items = [
    'item1',
    'item2',  // Trailing comma
];

// ❌ 錯誤 - 沒有結尾逗號
const styles = {
    container: { p: 2 },
    header: { mb: 1 }  // Missing comma
};
```

---

## 常用樣式模式

### Flexbox 版面配置

```typescript
const styles = {
    flexRow: {
        display: 'flex',
        flexDirection: 'row',
        alignItems: 'center',
        gap: 2,
    },
    flexColumn: {
        display: 'flex',
        flexDirection: 'column',
        gap: 1,
    },
    spaceBetween: {
        display: 'flex',
        justifyContent: 'space-between',
        alignItems: 'center',
    },
};
```

### 間距

```typescript
// Padding
p: 2           // All sides
px: 2          // Horizontal (left + right)
py: 2          // Vertical (top + bottom)
pt: 2, pr: 1   // Specific sides

// Margin
m: 2, mx: 2, my: 2, mt: 2, mr: 1

// Units: 1 = 8px (theme.spacing(1))
p: 2  // = 16px
p: 0.5  // = 4px
```

### 定位

```typescript
const styles = {
    relative: {
        position: 'relative',
    },
    absolute: {
        position: 'absolute',
        top: 0,
        right: 0,
    },
    sticky: {
        position: 'sticky',
        top: 0,
        zIndex: 1000,
    },
};
```

---

## 總結

**樣式檢查清單：**
- ✅ 使用 `sx` prop 進行 MUI 樣式設定
- ✅ 使用 `SxProps<Theme>` 做型別安全
- ✅ <100 行：inline；>100 行：獨立檔案
- ✅ MUI v7 Grid：`size={{ xs: 12 }}`
- ✅ 4 個空格縮排
- ✅ 單引號
- ✅ 結尾逗號
- ❌ 不要使用 makeStyles 或 styled()

**另請參考：**
- [component-patterns.md](component-patterns.md) - 元件結構
- [complete-examples.md](complete-examples.md) - 完整樣式範例
