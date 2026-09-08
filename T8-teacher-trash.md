# T8 · 휴지통 (Trash · 教师主视图 / 学生复用)

复现 `/trash` 页面的完整"软删除回收站"，涵盖入口、按角色区分的可回收资源、单条/批量的恢复与永久删除、7 日自动清理、Workspace 风格的视觉体系，以及基于 SECURITY DEFINER RPC 的所有权与管理员判定。仅描述现平台真实存在的入口与逻辑。

---

## 1. Identity（身份与范围）

你是一名 React + Supabase 全栈工程师。任务是复现"멜로디 클래스"平台的回收站页面 `/trash`（教师·管理员 4 个 tab，学生 2 个 tab，同一路由复用）。


**必须命中的现有能力**（缺一不可）：

- 侧边栏入口：`AppSidebar` 教师导航项 `{ title: "휴지통", url: "/trash", icon: Trash2 }`（同一入口对教师与管理员可见）。
- 路由：`<Route path="/trash" element={<RequireAuth><AppLayout><Trash /></AppLayout></RequireAuth>} />`（登录后可访问，包裹在 `AppLayout` 内以获得 `SidebarProvider` 上下文）。
- 页面整体为 Workspace 视觉体系：外层 `-m-3 sm:-m-4 md:-m-6 p-3 sm:p-4 md:p-6 min-h-full`，背景 `#f4f3f0`，内容为两张白卡（`bg-white rounded-[24px] p-4 sm:p-6`，`border 1px rgba(0,0,0,0.07)`，`box-shadow 0 1px 3px rgba(0,0,0,0.06)`）。
- 头部卡片：左侧 `44×44 rounded-2xl bg-[#eeeffe] border-[#c7caff]` 内嵌 `Trash2 text-[#4f52c8]`；右侧标题 "휴지통" + 副标题 "삭제한 항목은 **7일간**(靛蓝加粗) 보관 후 자동으로 영구 삭제됩니다. 본인이 삭제한 항목만 표시됩니다."
- Tab 集合随角色变化（`useAuth()` 的 `isTeacher || isAdmin` = `isStaff`）：
  - 教师/管理员：`songs / lesson_plans / courses / course_student_profiles` → `노래 / 강의안 / 과정 / 학생`
  - 学生：`songs / left_courses` → `노래 / 가입한 반`（`left_courses` 为虚拟表，见 §2.1）
  - `isStaff` 变化时若当前 tab 不在新集合内，自动回落到第一个 tab。
- Tab 采用自绘样式（非默认 shadcn 外观）：`TabsList bg-[#faf9f7] p-1 rounded-xl border`，激活项 `bg-white text-[#4f52c8] shadow-sm border-[#c7caff]`；每个 tab 名后跟一枚圆形计数徽章（激活 `bg-[#eeeffe]/#3739a8/#c7caff`，非激活 `bg-[#f4f3f0]/#6b6880`）。
- 顶部右侧两枚小按钮使用 `wsBtn` 令牌（11px、圆角 7px）：`전체 복원`（`wsBtn.outline` + RotateCcw）、`휴지통 비우기`（自定义 destructive 变体 `bg-[#fee2e2] text-[#991b1b] border-[#fecaca]` + Trash）。当前 tab 为空时 disabled。
- 卡片列表：`grid gap-3 sm:grid-cols-2 lg:grid-cols-3`，卡片 `rounded-[18px] bg-white border`，hover 上浮 `-translate-y-0.5` 并由 JS 设置阴影 `0 8px 28px rgba(79,82,200,.10)`；内含主标题、右上 `X일 남음` 徽章、副标题、`삭제일: <ko-KR locale>`。
- 卡片右键（`ContextMenu`）在整个 tab 空白区触发，提供 `전체 복원 / 전체 영구 삭제`。
- 卡片点击 → 打开明细 `Dialog`（`rounded-2xl`），含 `복원 / 영구 삭제` 两个按钮，二者均再弹 `AlertDialog` 二次确认。
- 底部：`totalCount === 0 && !loading` 时显示 "모든 휴지통이 비어 있어요."。
- 触发引导：页面加载时调用 `useGuideTour()`。


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
- 另有一张**虚拟表** `left_courses`：不存在同名物理表，数据来自 `course_members.left_at`（学生退出班级后 7 日内可恢复），仅出现在学生视图。
- RPC（全部 `SECURITY DEFINER, search_path=public`）：
  - `soft_delete_item(_table text, _id uuid) returns void`
  - `restore_trash_item(_table text, _id uuid) returns void`
  - `purge_trash_item(_table text, _id uuid) returns void`
  - `restore_all_trash(_table text) returns integer`
  - `purge_all_trash(_table text) returns integer`
  - `purge_expired_trash() returns void` —— 定时 job 会对 `deleted_at < now() - interval '7 days'` 的行做硬删除。
  - `move_to_trash(_table, _id) returns uuid` —— 用于"编辑非本人 public 项"时先 `fork_public_item` 再软删的路径（页面本身不直接调用，但需理解其存在，避免其他地方误改）。
  - 学生班级三件套：`get_my_left_courses()`（返回 `course_id, name, level, semester, left_at`）、`restore_course_membership(_course_id)`、`purge_course_membership(_course_id)`。
- 前 5 个 RPC **只接受这 4 张表名**，任何其他值 `RAISE EXCEPTION 'unsupported table %'`；`left_courses` 永远不会被传入。
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
  | "course_student_profiles"
  | "left_courses"; // 虚拟表：学生"가입한 반"，由 course_members.left_at 支撑

export const TRASH_LABEL: Record<TrashTable, string> = {
  songs: "노래",
  lesson_plans: "강의안",
  courses: "과정",
  course_student_profiles: "학생",
  left_courses: "가입한 반",
};

export const TRASH_RETENTION_DAYS = 7;

export async function softDelete(table: TrashTable, id: string): Promise<void>;   // 仅 4 张真实表，否则 throw
export async function restoreItem(table: TrashTable, id: string): Promise<void>;
export async function purgeItem(table: TrashTable, id: string): Promise<void>;
export async function restoreAll(table: TrashTable): Promise<number>;
export async function purgeAll(table: TrashTable): Promise<number>;
export async function fetchLeftCourses(): Promise<LeftCourseRow[]>;
export function daysRemaining(deletedAt: string): number;
```

- 4 张真实表的 RPC 通过 `supabase.rpc("<name>", { _table, _id })` 调用；`restore_all/purge_all` 返回 `integer`，前端 `Number(data ?? 0)`。
- `left_courses` 走独立 RPC：单条 `restore_course_membership({ _course_id })` / `purge_course_membership({ _course_id })`，列表 `get_my_left_courses()`。**目前没有批量 RPC**，`restoreAll/purgeAll` 对该 tab 在客户端逐条循环调用并返回处理条数。
- `softDelete` 遇到 `left_courses` 直接 `throw new Error("unsupported trash table")` —— 学生离开班级由 `leave_course` RPC 负责（见 S1/S6）。
- `daysRemaining`：`Math.max(0, Math.ceil((new Date(deletedAt).getTime() + 7*24h - Date.now()) / (24h)))`；`left_courses` 用 `left_at` 代入同一公式。


### 2.3 页面加载：按角色并行拉取

在 `Trash.tsx` 的 `loadAll`（`useCallback`，依赖 `[user]`）中：

```ts
const ALL_TABLES: TrashTable[] = ["songs","lesson_plans","courses","course_student_profiles"]; // 教师·管理员
const STUDENT_TABLES: TrashTable[] = ["songs","left_courses"];                                  // 学生
const TABLES = isStaff ? ALL_TABLES : STUDENT_TABLES;
```

对每张真实表构造不同的 `select`：

| table | select |
|---|---|
| `songs` | `id, deleted_at, title, artist, title_bilingual, language` |
| `lesson_plans` | `id, deleted_at, title, level, plan_type` |
| `courses` | `id, deleted_at, name, level, semester` |
| `course_student_profiles` | `id, deleted_at, full_name, student_number, department, course_id` |
| `left_courses`（虚拟） | 不查表，改调 `get_my_left_courses()` RPC |

固定过滤：`.eq("owner_id", user.id).not("deleted_at","is",null).order("deleted_at",{ ascending:false })`。

各 tab 用 `await Promise.all(TABLES.map(...))` 并行发起。任何一路失败 → 该 tab 返回空数组，不阻塞其他 tab。

### 2.4 行 → 卡片映射（primary / secondary）

| table | primary | secondary |
|---|---|---|
| `songs` | `title_bilingual \|\| title \|\| artist \|\| "(제목 없음)"` | `artist \|\| ""` |
| `lesson_plans` | `title \|\| "(제목 없음)"` | ``${level ?? ""} · ${plan_type === "ai_generated" ? "AI 생성" : "내 강의안"}`` |
| `courses` | `name \|\| "(이름 없음)"` | ``${level ?? ""} · ${semester ?? ""}`` |
| `course_student_profiles` | `full_name \|\| "(이름 없음)"` | ``${student_number ?? "-"} · ${department ?? "-"}`` |
| `left_courses` | `name \|\| "(이름 없음)"` | ``${level ?? ""}${semester ? " · " + semester : ""}``；`deleted_at` 位置填 `left_at` |


`raw` 字段完整保留原行 —— 明细 Dialog 展示时可复用（当前仅显示 primary/secondary/日期，但保留为将来扩展）。

### 2.5 交互矩阵

| 触发 | 组件 | 动作 |
|---|---|---|
| 点击卡片 | 自绘 `div`（非 shadcn `Card`）`onClick` | `setDetail(row)` → 打开明细 `Dialog` |
| 明细 Dialog 内 `복원` | `Button variant="outline" size="sm"` | `setSingleDialog({ kind:"restore", row: detail })` |
| 明细 Dialog 内 `영구 삭제` | `Button variant="destructive" size="sm"` | `setSingleDialog({ kind:"purge", row: detail })` |
| 顶部 `전체 복원` | 原生 `button` + `wsBtn.outline` | `setBulkDialog({ kind:"restore" })` |
| 顶部 `휴지통 비우기` | 原生 `button` + `wsBtn.base` + `bg-[#fee2e2] text-[#991b1b] border-[#fecaca]` | `setBulkDialog({ kind:"purge" })` |
| Tab 空白区右键 | `ContextMenu` | 弹出 `전체 복원 / 전체 영구 삭제`（后者 `text-destructive`，列表为空时 disabled） |

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
items: Partial<Record<TrashTable, TrashRow[]>>              // 按角色只装载可见的 tab
loading: boolean                                            // 首次/刷新加载中
bulkDialog: null | { kind:"purge"|"restore" }
singleDialog: null | { kind:"purge"|"restore"; row: TrashRow }
detail: TrashRow | null
```

`TABLES = isStaff ? ALL_TABLES : STUDENT_TABLES`（由 `useAuth()` 的 `isTeacher || isAdmin` 推导），`totalCount = useMemo(() => TABLES.reduce((s,t)=>s+(items[t]?.length ?? 0), 0), [items, TABLES])`。

### 2.8 加载态与空态

- `loading` → 每个 tab 内容区显示骨架块 `<div className="h-40 rounded-2xl bg-[#faf9f7] animate-pulse" />`（不再是 "로딩 중..." 文本）。tab 内容区最小高度 `min-h-[220px]`，避免切换抖动。
- 该 tab 无数据 → 居中空态（`py-14 sm:py-16`）：`64×64` 圆形 `bg-[#eeeffe] border-[#c7caff]` + `Trash2 text-[#4f52c8]` 图标，标题 "휴지통이 비어 있어요"，副文案 `${TRASH_LABEL[t]}에서 삭제한 항목은 7일간 보관 후 자동으로 영구 삭제돼요.`，底部一枚灰色 pill "7일 보관 정책"（天数取 `TRASH_RETENTION_DAYS`）。


---

## 3. Examples（关键代码骨架）

### 3.1 并行拉取（含虚拟表分支）

```ts
const queries = await Promise.all(TABLES.map(async (t) => {
  if (t === "left_courses") {
    try {
      const list = await fetchLeftCourses();          // rpc get_my_left_courses
      return { t, rows: list.map((r) => ({
        id: r.id,
        deleted_at: r.left_at,                        // 复用同一套保留期计时
        primary: r.name || "(이름 없음)",
        secondary: `${r.level || ""}${r.semester ? " · " + r.semester : ""}`,
        raw: r,
      })) };
    } catch { return { t, rows: [] as TrashRow[] }; }
  }

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

### 3.3 卡片剩余天数徽章（三档配色，不使用 shadcn `Badge`）

```tsx
function daysBadgeVariant(days: number) {
  if (days <= 1) return "bg-[#fee2e2] text-[#991b1b] border-[#fecaca]";  // 红：≤1일
  if (days <= 3) return "bg-[#fef3c7] text-[#92400e] border-[#fde68a]";  // 琥珀：2~3일
  return "bg-[#eeeffe] text-[#3739a8] border-[#c7caff]";                 // 靛蓝：4일 이상
}

const days = daysRemaining(row.deleted_at);
<span className={`shrink-0 inline-flex items-center px-2 py-0.5 rounded-full text-[10px] font-bold border ${daysBadgeVariant(days)}`}>
  {days}일 남음
</span>
```


---

## 4. Context（与其它模块的边界）

- **调用侧（谁把行放进回收站）**：不由本页发起 —— 而是散布在 `Songs / Lessons / Courses / Students / Workspace` 等页面的 "삭제" 按钮，那些按钮统一调用 `move_to_trash` 或 `soft_delete_item` RPC，本页只做**接收与逆向操作**。
- **过期清理**：由服务端 `purge_expired_trash()` 定时执行 —— 前端不做本地清理。UI 上仅通过 `daysRemaining` 显示"还剩几天"。
- **管理员差异**：`is_admin()` 用户的 `owner_id IS NULL`（种子公共内容）也会在回收站显示；普通教师看不到 `owner_id IS NULL` 的行，只看到 `owner_id = auth.uid()`。前端不做区分，交给 RPC + `owner_id` 过滤天然拆分。
- **不覆盖的表**：只有这 4 张表实现了 `deleted_at` 语义。其他表（`course_home_blocks / course_calendar_items / course_notices / course_posts / materials / user_roles` 等）走**硬删除**，不进回收站。若未来新增软删表，需要同时改 RPC 白名单 + 本页 `TABLES` 常量 + `TRASH_LABEL`。
- **导航**：`AppSidebar` 的教师组与学生组**各有一个** `{ title: "휴지통", url: "/trash", icon: Trash2 }` 入口，页面本身按 `isTeacher || isAdmin` 决定 tab 集合，因此同一路由为两类角色复用。
- **学生 tab**：`가입한 반` 的入口动作（`leave_course`）在 S1/S6 描述，本页只负责恢复（`restore_course_membership`）与永久删除（`purge_course_membership`）。

---

## 5. Acceptance（验收清单）

1. 侧边栏教师视图能看到 `휴지통` 项；点击进入 `/trash` 无报错。学生登录时同一路由只显示 `노래 / 가입한 반` 两个 tab。
2. 首屏按角色并行拉取，每个 tab 名后的计数徽章与实际卡片数一致。
3. 卡片右上角 `X일 남음` 三档配色：剩 1 天 → 红；剩 2~3 天 → 琥珀；剩 4 天以上 → 靛蓝。
4. 单条复原：点卡片 → 明细 Dialog → 복원 → AlertDialog 确认 → toast `복원 완료` → 该行从当前 tab 消失，源页面（如 `/songs`）重新可见。
5. 单条永久删除：同上流程，源表中该行物理消失（`select ... where id=...` 空）。
6. 批量复原：`전체 복원` → AlertDialog 描述里的数量与卡片总数一致 → toast 显示实际处理条数（真实表取 RPC 返回值，`left_courses` 取循环条数）。
7. 批量永久：AlertTriangle 图标出现在标题左侧，`이 작업은 되돌릴 수 없습니다.` 出现在描述中。
8. Tab 空白区右键弹出与顶部按钮功能一致的 `ContextMenu`；列表为空时两项 disabled。
9. 尝试传入非白名单 `_table`（如手工在 devtools 里调用 `restoreItem("materials" as any, ...)`）→ RPC 报错 `unsupported table materials`，前端 toast 展示。
10. 未登录用户直达 `/trash` 被 `RequireAuth` 挡下；已登录但另一账号创建的行永远不会出现在当前 `owner_id = auth.uid()` 过滤后的列表里。
11. 页面无任何直接的 `.delete()` / `.update()` 调用 —— 所有变更 100% 通过 RPC。
12. 视觉：背景 `#f4f3f0`，两张白卡 `rounded-[24px]`，列表卡片 `rounded-[18px]`，顶部按钮为 11px 小按钮（`wsBtn`），空态文案为 "휴지통이 비어 있어요"，全空时底部为 "모든 휴지통이 비어 있어요."。


---

## 6. 与既有 P / T 文档的边界差异

- **T2（Workspace）**：教师工作台的 "새 반 만들기 / 강의안 생성" 等入口 —— 那里发起软删（如删除自己的课程/讲义案），**推入** 回收站；T8 只负责回收站内**逆向操作**。
- **T5（课程基础与加入）**：`Courses.tsx` 卡片的 `move_to_trash` 调用点在 T5，本文件不重复描述发起端。
- **T7（学生批量管理）**：`course_student_profiles` 行的删除入口在 T7；本页负责它的复原/永久删除。
- **P3（歌曲档案）系列**：歌曲行的删除入口分布在 `Songs.tsx` / 卡片右键，均调 `move_to_trash("songs", ...)`；T8 是它们的唯一恢复出口。
- **P5/T4/T6（课程详情）**：`course_home_blocks / course_calendar_items / course_notices / course_posts / materials` 全部**硬删除**，故本页 tab 不包含这些资源 —— 复现时务必只列 4 张表，不要"顺手加一个通知回收站"。
