# T8 · 教师端 휴지통 (Trash)

复现 `/trash` 页面的完整教师端"软删除回收站"，涵盖入口、四类可回收资源、单条/批量的恢复与永久删除、7 日自动清理、以及基于 SECURITY DEFINER RPC 的所有权与管理员判定。仅描述现平台真实存在的入口与逻辑。

---

## 1. Identity（身份与范围）

你是一名 React + Supabase 全栈工程师。任务是复现"멜로디 클래스"平台的**教师端回收站**页面 `/trash`。

**必须命中的现有能力**（缺一不可）：

- 侧边栏入口：`AppSidebar` 教师导航项 `{ title: "휴지통", url: "/trash", icon: Trash2 }`（同一入口对教师与管理员可见）。
- 路由：`<Route path="/trash" element={<RequireAuth><AppLayout><Trash /></AppLayout></RequireAuth>} />`（登录后可访问，包裹在 `AppLayout` 内以获得 `SidebarProvider` 上下文）。
- 页面标题："휴지통"，副标题固定文案："삭제한 항목은 7일간 보관 후 자동으로 영구 삭제됩니다. 본인이 삭제한 항목만 표시됩니다."
- 4 个 tabs（顺序固定）：`songs / lesson_plans / courses / course_student_profiles`，Korean 标签依次为：`노래 / 강의안 / 과정 / 학생`。
- 每个 tab 右侧带一个 count Badge（当前保留项数量）。
- 顶部右侧两枚全局按钮：`전체 복원`（outline, RotateCcw 图标）、`휴지통 비우기`（destructive, Trash 图标）。二者在当前 tab 列表为空时置 disabled。
- 卡片列表：响应式 `grid gap-3 sm:grid-cols-2 lg:grid-cols-3`，每张卡片显示主标题、副标题（可选）、"삭제일: <ko-KR locale>"、右上角 `X일 남음` Badge（≤1 天 → destructive variant）。
- 卡片右键（`ContextMenu`）在整个 tab 空白区触发，提供 `전체 복원 / 전체 영구 삭제`。
- 卡片点击 → 打开明细 `Dialog`，含 `복원 / 영구 삭제` 两个按钮，二者均再弹 `AlertDialog` 二次确认。
- 底部：`totalCount === 0 && !loading` 时显示"모든 휴지통이 비어 있습니다."。
- 触发引导：页面加载时调用 `useGuideTour()`（存在 `data-tour="trash-*"` 锚点场景下配合教师 tour 使用；本页至少调用 hook）。

**不做**：

- 不做客户端删除 —— 所有写操作必须走 SECURITY DEFINER RPC。
- 不做管理员"全站清空所有用户"面板 —— 页面只显示本人 `owner_id = auth.uid()` 的行。
- 不做导出 / 打印 / 分享 —— 回收站不提供任何对外分享入口。
- 不做筛选 / 搜索框 —— 仅按 `deleted_at desc` 排序。

---

## 2. Instructions（实现步骤）

### 2.1 数据契约（后端已存在，禁止改）

数据库中以下对象**已存在**，前端只能调用，不得重复迁移：

- 4 张业务表均含 `owner_id uuid` 与 `deleted_at timestamptz null` 两列。
- 表清单：`public.songs / public.lesson_plans / public.courses / public.course_student_profiles`。
- RPC（全部 `SECURITY DEFINER, search_path=public`）：
  - `soft_delete_item(_table text, _id uuid) returns void`
  - `restore_trash_item(_table text, _id uuid) returns void`
  - `purge_trash_item(_table text, _id uuid) returns void`
  - `restore_all_trash(_table text) returns integer`
  - `purge_all_trash(_table text) returns integer`
  - `purge_expired_trash() returns void` —— 定时 job 会对 `deleted_at < now() - interval '7 days'` 的行做硬删除。
  - `move_to_trash(_table, _id) returns uuid` —— 用于"编辑非本人 public 项"时先 `fork_public_item` 再软删的路径（页面本身不直接调用，但需理解其存在，避免其他地方误改）。
- 所有 RPC **只接受这 4 张表名**，任何其他值 `RAISE EXCEPTION 'unsupported table %'`。
- RPC 内部执行条件恒为：`owner_id = auth.uid() OR (owner_id IS NULL AND public.is_admin())`。管理员对 `owner_id IS NULL` 的公共种子数据可回收，普通教师只能操作自己创建的行。

### 2.2 目录与文件

```
src/lib/trash.ts                       # 类型、常量、RPC 薄包装
src/pages/Trash.tsx                    # 页面
src/components/AppSidebar.tsx          # 增加导航项（已存在）
src/App.tsx                            # 增加路由（已存在）
```

`src/lib/trash.ts` 必须导出：

```ts
export type TrashTable =
  | "songs"
  | "lesson_plans"
  | "courses"
  | "course_student_profiles";

export const TRASH_LABEL: Record<TrashTable, string> = {
  songs: "노래",
  lesson_plans: "강의안",
  courses: "과정",
  course_student_profiles: "학생",
};

export const TRASH_RETENTION_DAYS = 7;

export async function softDelete(table: TrashTable, id: string): Promise<void>;
export async function restoreItem(table: TrashTable, id: string): Promise<void>;
export async function purgeItem(table: TrashTable, id: string): Promise<void>;
export async function restoreAll(table: TrashTable): Promise<number>;
export async function purgeAll(table: TrashTable): Promise<number>;
export function daysRemaining(deletedAt: string): number;
```

- 全部 RPC 通过 `supabase.rpc("<name>", { _table, _id })` 调用；`restore_all/purge_all` 返回 `integer`，前端 `Number(data ?? 0)`。
- `daysRemaining`：`Math.max(0, Math.ceil((new Date(deletedAt).getTime() + 7*24h - Date.now()) / (24h)))`。

### 2.3 页面加载：4 表并行拉取

在 `Trash.tsx` 的 `loadAll`（`useCallback`，依赖 `[user]`）中：

```ts
const TABLES: TrashTable[] = ["songs","lesson_plans","courses","course_student_profiles"];
```

对每张表构造不同的 `select`：

| table | select |
|---|---|
| `songs` | `id, deleted_at, title, artist, title_bilingual, language` |
| `lesson_plans` | `id, deleted_at, title, level, plan_type` |
| `courses` | `id, deleted_at, name, level, semester` |
| `course_student_profiles` | `id, deleted_at, full_name, student_number, department, course_id` |

固定过滤：`.eq("owner_id", user.id).not("deleted_at","is",null).order("deleted_at",{ ascending:false })`。

四个查询用 `await Promise.all(TABLES.map(...))` 并行发起。任何一表查询失败 → 该表返回空数组，不阻塞其他 tab。

### 2.4 行 → 卡片映射（primary / secondary）

| table | primary | secondary |
|---|---|---|
| `songs` | `title_bilingual \|\| title \|\| artist \|\| "(제목 없음)"` | `artist \|\| ""` |
| `lesson_plans` | `title \|\| "(제목 없음)"` | ``${level ?? ""} · ${plan_type === "ai_generated" ? "AI 생성" : "내 강의안"}`` |
| `courses` | `name \|\| "(이름 없음)"` | ``${level ?? ""} · ${semester ?? ""}`` |
| `course_student_profiles` | `full_name \|\| "(이름 없음)"` | ``${student_number ?? "-"} · ${department ?? "-"}`` |

`raw` 字段完整保留原行 —— 明细 Dialog 展示时可复用（当前仅显示 primary/secondary/日期，但保留为将来扩展）。

### 2.5 交互矩阵

| 触发 | 组件 | 动作 |
|---|---|---|
| 点击卡片 | `Card onClick` | `setDetail(row)` → 打开明细 `Dialog` |
| 明细 Dialog 内 `복원` | `Button variant="outline"` | `setSingleDialog({ kind:"restore", row: detail })` |
| 明细 Dialog 内 `영구 삭제` | `Button variant="destructive"` | `setSingleDialog({ kind:"purge", row: detail })` |
| 顶部 `전체 복원` | `Button variant="outline"` | `setBulkDialog({ kind:"restore" })` |
| 顶部 `휴지통 비우기` | `Button variant="destructive"` | `setBulkDialog({ kind:"purge" })` |
| Tab 空白区右键 | `ContextMenu` | 弹出 `전체 복원 / 전체 영구 삭제`（后者 `text-destructive`） |
| `AlertDialog` 确认 | `AlertDialogAction` | `e.preventDefault(); void handle...()` |

`handleSingle` 与 `handleBulk` 完成后：`setXxxDialog(null); setDetail(null); void loadAll();`。

失败统一：`toast({ title:"오류", description: e?.message ?? "다시 시도해주세요.", variant:"destructive" })`。

成功文案（严格一致）：

- 单条复原：`title:"복원 완료"`，`description:` `‘${row.primary}’ 항목이 복원되었습니다.`
- 单条永久：`title:"영구 삭제 완료"`，`description:` `‘${row.primary}’ 항목이 영구 삭제되었습니다.`
- 批量复原：`title:"전체 복원"`，`description:` `${n}개 항목이 복원되었습니다.`
- 批量永久：`title:"전체 영구 삭제"`，`description:` `${n}개 항목이 영구 삭제되었습니다.`

### 2.6 二次确认（AlertDialog）文案

- 批量 restore：`AlertDialogTitle` = `${TRASH_LABEL[tab]} 전체 복원`，描述 = `${TRASH_LABEL[tab]} 휴지통의 모든 항목(${currentRows.length}개)을 복원합니다.`
- 批量 purge：标题带 `AlertTriangle text-destructive` 图标，`${TRASH_LABEL[tab]} 전체 영구 삭제`，描述 = `${TRASH_LABEL[tab]} 휴지통의 모든 항목(${currentRows.length}개)을 영구 삭제합니다. 이 작업은 되돌릴 수 없습니다.`
- 单条 restore/purge：标题 `복원 / 영구 삭제`；描述 `‘${row.primary}’ 항목을 복원합니다.` 或 `... 영구 삭제합니다. 이 작업은 되돌릴 수 없습니다.`

按钮统一 `취소 / 확인`。

### 2.7 状态机（一定要用 `useState`，不要用外部 store）

```ts
tab: TrashTable                                             // 当前激活的 tab
items: Record<TrashTable, TrashRow[]>                       // 4 表数据
loading: boolean                                            // 首次/刷新加载中
bulkDialog: null | { kind:"purge"|"restore" }
singleDialog: null | { kind:"purge"|"restore"; row: TrashRow }
detail: TrashRow | null
```

`totalCount = useMemo(() => TABLES.reduce((s,t)=>s+items[t].length, 0), [items])`。

### 2.8 加载态与空态

- `loading` → 每个 tab 内容区显示 `<div className="text-muted-foreground text-sm">로딩 중...</div>`（不用 skeleton）。
- 该 tab 无数据 → 显示大空态卡片：`Trash2` 图标 + "휴지통이 비어 있습니다" + 副文案 "${TRASH_LABEL[t]}에서 삭제한 항목이 여기에 표시됩니다."

---

## 3. Examples（关键代码骨架）

### 3.1 并行拉取

```ts
const queries = await Promise.all(TABLES.map(async (t) => {
  let select = "id, deleted_at";
  if (t === "songs") select += ", title, artist, title_bilingual, language";
  if (t === "lesson_plans") select += ", title, level, plan_type";
  if (t === "courses") select += ", name, level, semester";
  if (t === "course_student_profiles") select += ", full_name, student_number, department, course_id";

  const { data, error } = await (supabase as any)
    .from(t)
    .select(select)
    .eq("owner_id", user.id)
    .not("deleted_at", "is", null)
    .order("deleted_at", { ascending: false });
  if (error) return { t, rows: [] as TrashRow[] };
  return { t, rows: (data ?? []).map(mapRow(t)) };
}));
```

### 3.2 RPC 薄包装（`src/lib/trash.ts`）

```ts
export async function softDelete(table: TrashTable, id: string) {
  const { error } = await (supabase as any).rpc("soft_delete_item", { _table: table, _id: id });
  if (error) throw error;
}
export async function restoreAll(table: TrashTable): Promise<number> {
  const { data, error } = await (supabase as any).rpc("restore_all_trash", { _table: table });
  if (error) throw error;
  return Number(data ?? 0);
}
```

其余 4 个 RPC 同形，参数固定 `_table / _id` 或仅 `_table`。

### 3.3 卡片剩余天数 Badge

```tsx
const days = daysRemaining(row.deleted_at);
<Badge variant={days <= 1 ? "destructive" : "secondary"}>{days}일 남음</Badge>
```

---

## 4. Context（与其它模块的边界）

- **调用侧（谁把行放进回收站）**：不由本页发起 —— 而是散布在 `Songs / Lessons / Courses / Students / Workspace` 等页面的 "삭제" 按钮，那些按钮统一调用 `move_to_trash` 或 `soft_delete_item` RPC，本页只做**接收与逆向操作**。
- **过期清理**：由服务端 `purge_expired_trash()` 定时执行 —— 前端不做本地清理。UI 上仅通过 `daysRemaining` 显示"还剩几天"。
- **管理员差异**：`is_admin()` 用户的 `owner_id IS NULL`（种子公共内容）也会在回收站显示；普通教师看不到 `owner_id IS NULL` 的行，只看到 `owner_id = auth.uid()`。前端不做区分，交给 RPC + `owner_id` 过滤天然拆分。
- **不覆盖的表**：只有这 4 张表实现了 `deleted_at` 语义。其他表（`course_home_blocks / course_calendar_items / course_notices / course_posts / materials / user_roles` 等）走**硬删除**，不进回收站。若未来新增软删表，需要同时改 RPC 白名单 + 本页 `TABLES` 常量 + `TRASH_LABEL`。
- **导航**：`AppSidebar` 教师与管理员共享同一入口；学生角色（`isTeacher === false && isAdmin === false`）的侧边栏不含 `/trash` 项（学生路由已在 `AppSidebar` 的 `studentItems / teacherItems` 分组中天然区隔）。

---

## 5. Acceptance（验收清单）

1. 侧边栏教师视图能看到 `휴지통` 项；点击进入 `/trash` 无报错。
2. 首屏并行拉取 4 表，每 tab 计数 Badge 与实际卡片数一致。
3. 卡片右上角 `X일 남음`：`deleted_at` 距今 6 天 → `1일 남음` destructive；距今 1 天 → `6일 남음` secondary。
4. 单条复原：点卡片 → 明细 Dialog → 복원 → AlertDialog 确认 → toast `복원 완료` → 该行从当前 tab 消失，源页面（如 `/songs`）重新可见。
5. 单条永久删除：同上流程，源表中该行物理消失（`select ... where id=...` 空）。
6. 批量复原：`전체 복원` → AlertDialog 描述里的数量与卡片总数一致 → toast 显示实际处理条数（RPC 返回值）。
7. 批量永久：AlertTriangle 图标出现在标题左侧，`이 작업은 되돌릴 수 없습니다.` 出现在描述中。
8. Tab 空白区右键弹出与顶部按钮功能一致的 `ContextMenu`；列表为空时两项 disabled。
9. 尝试传入非白名单 `_table`（如手工在 devtools 里调用 `restoreItem("materials" as any, ...)`）→ RPC 报错 `unsupported table materials`，前端 toast 展示。
10. 未登录用户直达 `/trash` 被 `RequireAuth` 挡下；已登录但另一账号创建的行永远不会出现在当前 `owner_id = auth.uid()` 过滤后的列表里。
11. 页面无任何直接的 `.delete()` / `.update()` 调用 —— 所有变更 100% 通过上述 5 个 RPC。

---

## 6. 与既有 P / T 文档的边界差异

- **T2（Workspace）**：教师工作台的 "새 반 만들기 / 강의안 생성" 等入口 —— 那里发起软删（如删除自己的课程/讲义案），**推入** 回收站；T8 只负责回收站内**逆向操作**。
- **T5（课程基础与加入）**：`Courses.tsx` 卡片的 `move_to_trash` 调用点在 T5，本文件不重复描述发起端。
- **T7（学生批量管理）**：`course_student_profiles` 行的删除入口在 T7；本页负责它的复原/永久删除。
- **P3（歌曲档案）系列**：歌曲行的删除入口分布在 `Songs.tsx` / 卡片右键，均调 `move_to_trash("songs", ...)`；T8 是它们的唯一恢复出口。
- **P5/T4/T6（课程详情）**：`course_home_blocks / course_calendar_items / course_notices / course_posts / materials` 全部**硬删除**，故本页 tab 不包含这些资源 —— 复现时务必只列 4 张表，不要"顺手加一个通知回收站"。
