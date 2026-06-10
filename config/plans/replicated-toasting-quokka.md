# 备品备件管理系统 — 基于 Excel 表头重构

## Context

当前系统是一个通用的备品备件管理系统（4 个模型：Category、Item、Transaction、StockTake），与 `beijian.xlsx` 中的真实业务数据结构不匹配。Excel 包含两个 Sheet：

- **发货项目清单(含BBOM SN)**：6,264 行，38 列，涵盖项目→机器→部件的层级关系，含 SN、维保、SLA、备件库房、GAP 分析等
- **新增BOM**：72 行，22 列，BOM 主数据（BBOM 编码、物料大类/小类、型号、生命周期）

目标是按真实业务数据结构重新设计系统，同时保留现有的出入库交易和盘点功能。

## New Prisma Schema

新增 4 个模型，修改 1 个现有模型：

### 新增模型

**Project** — 34 个唯一项目
- name (unique), city, contractNumber, oem
- implementationDate, warrantyStart, warrantyEnd
- projectSla, remark

**Machine** — 232 个唯一整机 SN
- machineSn (unique, 泛联整机SN), manufacturerSn (厂商整机SN)
- product, modelCode, manufacturer
- projectId → Project

**Bom** — BOM 主数据（BBOM 编码为自然键）
- bomCode (unique, BBOM编码), sbomCode
- materialCategory (物料大类), materialSubcategory (物料小类), category (Sheet1 类别)
- model, subModel, name (描述), manufacturer, manufacturerModel
- unit, quantity, nandType, firmwareVersion
- lifecycle, effectiveDate, expiryDate
- supplier, detailDescription, processCode
- status, isSpare, remark

**Part** — 核心实体，对应 Sheet1 每行（6,264 条 SN 级记录）
- partSn (unique, 部件序列号)
- bomCode → Bom, machineId → Machine, projectId → Project
- description, model, subModel, nandType, firmwareVersion, equipmentCategory
- purchaseDate, projectWarrantyMonths, supplierWarrantyMonths, postStartupWarrantyMonths
- projectSla, supplierSla, failureRate
- isSpare, spareResponsible, spareQuantity, spareWarehouse, spareStrategy, spareStatus
- monthGap, gapMonths, slaGap
- supplier, remark

### 修改模型

**Item** — 新增 `bomCode` 可选字段关联 Bom，其余不变

### 保留不变

Category, Transaction, StockTake 模型完全不变

## Implementation Plan

### Step 1: 数据层重构
- 替换 `prisma/schema.prisma` 为新设计
- `npx prisma db push` 应用到数据库（无现有数据，安全替换）
- `npx prisma generate` 重新生成客户端
- **验证**：检查 `src/generated/prisma/` 中新模型已生成

### Step 2: 数据导入功能
- 创建 `src/lib/excel.ts`：Excel 序列号日期转换（`excelSerialToDate`）、导入主逻辑
- 创建 `src/app/api/import/route.ts`：POST 端点，先导 BOM(Sheet2)，再导 项目→机器→部件(Sheet1)，使用 upsert 防重复
- 创建 `src/app/import/page.tsx`：导入页面（上传/触发/进度展示）
- **验证**：执行导入，数据库中有 34 项目、232 机器、72+ BOM、6,264 部件

### Step 3: 新增 API 路由（按依赖顺序）
- `src/app/api/boms/route.ts` + `[id]/route.ts`：GET 列表/详情、POST 创建、PUT 更新、DELETE
- `src/app/api/projects/route.ts` + `[id]/route.ts`：同上，详情含 machines 和 parts 统计
- `src/app/api/machines/route.ts` + `[id]/route.ts`：按 project 筛选，详情含 parts
- `src/app/api/parts/route.ts` + `[id]/route.ts`：多条件筛选（项目/BOM/备件状态/库房）、搜索、分页

### Step 4: 新增页面和组件
- `src/app/boms/page.tsx` + `[id]/page.tsx`：BOM 列表 + 详情
- `src/app/projects/page.tsx` + `[id]/page.tsx`：项目列表 + 详情（机器/部件统计）
- `src/app/parts/page.tsx` + `[id]/page.tsx`：部件列表（分页/筛选） + 详情
- `src/components/project-form-dialog.tsx`：项目新增/编辑弹窗
- `src/components/bom-form-dialog.tsx`：BOM 新增/编辑弹窗
- **验证**：每个页面可正常加载、搜索、筛选

### Step 5: 修改现有文件和仪表盘
- `src/components/sidebar.tsx`：新增导航（项目管理、部件管理、BOM管理、数据导入）
- `src/app/page.tsx`（仪表盘）：使用新模型数据——项目数、机器数、部件数、备件数、维保到期项目等
- `src/components/item-form-dialog.tsx`：增加 bomCode 可选字段
- `src/app/items/page.tsx`：表格中展示 bomCode，支持按 BOM 筛选
- `src/app/items/[id]/page.tsx`：展示关联 BOM 信息
- **验证**：仪表盘正确显示、导航可访问所有页面、出入库和盘点功能正常

### Step 6: 端到端验证
- `npm test` 全部通过
- 开发服务器 `localhost:3000` 正常运行
- 所有页面可访问、搜索/筛选/分页正常
- 出入库交易和盘点功能不受影响

## Key Design Decisions

| 决策 | 原因 |
|------|------|
| Part.partSn 作为 @unique | 6,264 SN 中仅 2 个重复（0.03%），视为数据异常 |
| Bom.bomCode 作为自然键 (@unique) | 业务上 BBOM 编码本身就是唯一标识 |
| Part 不直接关联 Transaction/StockTake | 交易和盘点针对仓库库存(Item)，非 SN 级部件 |
| Part 保留冗余规格字段 | 实测 14/129 BBOM 在不同行存在规格差异 |
| Category 独立于 BOM 物料分类 | 仓库分类与 BOM 物料分类正交，用途不同 |
| Excel 日期序列号转 DateTime | 公式：`new Date((serial - 25569) * 86400 * 1000)` |
