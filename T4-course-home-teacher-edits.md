# T4 · 班级首页教师端编辑增量（Hero 编辑 / 从教案填充 / Realtime 联动 / 侧栏 Toolbar）

> 本提示词只覆盖**教师专属**的编辑能力，在 P5a/P5b 的公共骨架之上叠加。学生视图不含本文件描述的任何 UI/权限。

---

## 1. Identity

你是一名资深 React + Supabase 工程师。你要在既有的班级首页（`CourseDetail` + `HomeTab` + `CourseHomeBlockRenderer`）之上，实现「编辑模式」下才可见的教师增量：封面 Hero 编辑器、右侧 Home 模块 Toolbar、Toolbar → HomeTab 的事件桥、buildBlockInsertPrefilled 的教案自动填充、以及 courses / course_home_blocks / course_header 三条 Realtime 通道，让老师在多标签页 / 多设备的编辑立即互相同步。

---

## 2. Instructions

### 2.1 编辑模式开关（在 `src/pages/CourseDetail.tsx`）

- 仅当 `isOwner = course.owner_id === user.id` 时渲染 Hero 右上角控制条：
  - `<Switch id="edit-mode">` + Label "편집 모드"，state `editMode`。
  - `editMode === true` 时额外显示 `<Button>표지 편집</Button>`（图标 `Image`），打开 `CourseHeroEditor`。
- 计算 `effectiveEditMode = isOwner && editMode`，只把这个值传给 `HomeTab / CalendarTab / …`。
- 布局：当 `effectiveEditMode && activeTab === "home"` 时，主区改为 `xl:grid-cols-[minmax(0,1fr)_320px] xl:items-start`，右列渲染 `<CourseModuleToolbar activeTab={activeTab} />`；其他 tab 或非编辑态一律铺满。

### 2.2 Hero 显示与覆写

`CourseDetail` 的 Hero `<section>`：

- 背景：优先使用 `course.cover_image_url` 作为绝对定位 `<img>` + 半透明 `rgba(10,5,30,0.5)` 遮罩；无封面时回退到 `bg-gradient-to-br from-primary via-primary to-primary/70` + `bg-[#2e3d6b]` 装饰层。
- 文案 = **`course_header` block 的覆写 ∪ 课程原字段**：
  - `title = headerContent.title_override?.trim() || course.name`
  - 若 `show_date !== false`：`start_date ? "YYYY년 M월 D일 시작" : course.semester`
  - 若 `show_level !== false`：追加 `course.level`
  - 若 `show_class_time !== false`：追加 `course.class_time`
  - 三段用 " · " 连接得默认 subtitle；`subtitle_override` 覆盖它。
  - `extra_line` 作为第三行小字。
- Hero 内所有文字使用 `text-white / drop-shadow`，遵循深色叠层。

### 2.3 `CourseHeroEditor`（`src/components/courses/CourseHeroEditor.tsx`）

Dialog（max-w-2xl），三个 Tab：

1. **직접 업로드**：`<Input type="file" accept="image/*">`。
   - 目标 bucket：`course-home-assets`，路径 `${courseId}/hero-${Date.now()}.${ext}`，`upsert: true`。
   - 上传成功后取 `getPublicUrl`，再 `supabase.from("courses").update({ cover_image_url })`。
2. **Pixabay 검색**：调用 edge function `pixabay-search`，body `{ keywords:[kw], lang:"ko", perKeyword:8, totalLimit:8 }`。返回 `assets[{id,url,thumb,user}]`，3 列缩略图 grid，点击即 `persist(url)`。悬浮层显示 `© user`。
3. **제거**：预览当前封面，`persist(null)` 恢复默认渐变。

统一 `persist(url|null)`：更新 `courses.cover_image_url`，成功后 toast `표지가 업데이트되었습니다.`，调用父组件 `onChanged(url)` 就地同步 `course` state，关闭 Dialog。失败必须 toast `저장 실패` + error.message。

底部固定一行 `저작권을 확인하세요.` 提示。

### 2.4 侧栏 `CourseModuleToolbar`

`src/components/courses/CourseModuleToolbar.tsx`：

- 仅 `activeTab === "home"` 时渲染，其余返回 `null`。
- `Card` 使用 `xl:sticky xl:top-6 xl:z-20`。
- 循环 `blockDefinitions`（来自 `home-blocks/blockCatalog`），每项一张卡片：图标 + label + description + 右侧 `+` 按钮。
- 点击 `+` 通过 `window.dispatchEvent(new CustomEvent(COURSE_TOOLBAR_EVENT_NAME, { detail:{ tab:"home", type } }))` 通知 `HomeTab`，避免 prop 穿透。
- 常量：`COURSE_TOOLBAR_EVENT_NAME = "course-module-toolbar:add"`，配套导出 `dispatchCourseToolbarAdd(detail)` 帮助函数。

### 2.5 HomeTab 事件桥 + 教案自动填充

在 `HomeTab.tsx`：

- 挂载 `useEffect` 监听 `COURSE_TOOLBAR_EVENT_NAME`，`detail.tab === "home"` 时调用 `handleAddFromToolbar(detail.type)`。
- `handleAddFromToolbar` 内：
  1. 若课程绑定了教案，先 `SELECT * FROM lesson_plans WHERE course_id = :id ORDER BY created_at`，再批量 `SELECT id, lesson_plan_id, week_number, title, week_type, is_generated, content, song_ids FROM lesson_plan_weeks WHERE lesson_plan_id IN (...)`，按 `lesson_plan_id` 分组挂到 `plans[i].weeks`。
  2. 调 `buildBlockInsertPrefilled({ courseId, type, plans, existingCount })`（见 §2.6）拿到 `{ block_type, content, position }`。
  3. `INSERT INTO course_home_blocks` 得到新行；把它 append 到本地 `blocks` state 并推入 `history`（配合 P5a 的 Undo/Redo）。
  4. 失败 toast `블록 추가 실패`。

### 2.6 `buildBlockInsertPrefilled`（在 `home-blocks/blockCatalog`）

规则表（按 `type` 决定 content 预填）：

| type | 预填逻辑 |
|---|---|
| `course_header` | 空对象；由 Hero 编辑器/输入面板补 override 字段 |
| `weekly_preview` | 取最新 `plans[0].weeks` 中最近一周非 orientation/exam 的 `week_number,title`，写入 `{ lesson_plan_id, week_number }` |
| `songs_list` | 展开所有 week 的 `song_ids` 去重（保序），最多 8 首，写入 `{ song_ids:[...] }` |
| `vocab_grid` | 从 `plans[0].weeks[*].content.vocabulary` 摊平取前 12 个，写 `{ items:[{term,gloss}] }` |
| `goals` / `curriculum` | 取 `plans[0].content.overview` / `plans[0].content.weekly_outline` 的摘要 |
| `teacher_info` | 若 `profiles.is_public_to_students`，注入 `{ name, avatar_url, bio, contact }` |
| 其他（`text/image/video/lyrics/…`） | 返回 `createBlockInsert(type)` 的默认空 content |

`position` = `existingCount`（追加末尾），后续拖拽再由 P5a 的排序逻辑覆写。

### 2.7 三条 Realtime 通道

在 `CourseDetail` / `HomeTab` 内**分开挂**，防止一个订阅泄漏拖垮另一个：

1. **course 行**（`CourseDetail`）：`channel("course-row-${id}").on("postgres_changes", { event:"UPDATE", schema:"public", table:"courses", filter:"id=eq.${id}" }, refetch)`。同时监听自定义事件 `"course:updated"` 供本页 dialog 保存后立即刷新。
2. **course_header block**（`CourseDetail`）：`channel("course-hero-${id}").on("*", { table:"course_home_blocks", filter:"course_id=eq.${id}" }, reloadHeader)`；`reloadHeader` 只 `SELECT content WHERE block_type='course_header'` 并 setState，保证 Hero 文案实时。
3. **blocks 列表**（`HomeTab`）：`channel("course-home-${id}").on("*", { table:"course_home_blocks", filter:"course_id=eq.${id}" }, refetchBlocks)`；`refetchBlocks` 全量拉取当前 `blocks` 并按 `position` 排序 setState，同时**跳过本地刚提交的 tx_id**（在 payload.new.updated_by === self 时忽略）以避免抖动。

所有 channel 必须在 unmount `supabase.removeChannel(channel)`，且写在 `useEffect` 内、依赖 `[course?.id]`。

### 2.8 权限与安全

- 只有 `isOwner` 才能看见 Toolbar / Hero 编辑按钮 / 表지 편집 / 편집 모드 Switch；组件层不做 UI 隐藏依赖，必须再由 RLS 兜底：`course_home_blocks` 与 `courses` 的 `UPDATE / INSERT / DELETE` 策略要求 `auth.uid() = courses.owner_id`。
- Storage：`course-home-assets` 的写入策略限定 `name like courseId || '/%'` 且 owner 校验；学生只读。
- 学生打开链接时 `editMode` 恒为 false，任何 dispatch/handler 都因为 `isOwner === false` 而不挂载。

---

## 3. Examples

### 3.1 Realtime + 本地写入不打架

```ts
// HomeTab.tsx
useEffect(() => {
  const ch = supabase.channel(`course-home-${course.id}`)
    .on("postgres_changes",
        { event: "*", schema: "public", table: "course_home_blocks", filter: `course_id=eq.${course.id}` },
        (payload) => {
          const row = (payload.new ?? payload.old) as CourseHomeBlockRow;
          if (row && lastLocalTxId.current === row.updated_at) return; // 忽略自己刚写的
          void refetchBlocks();
        })
    .subscribe();
  return () => { supabase.removeChannel(ch); };
}, [course.id]);
```

### 3.2 Toolbar → HomeTab 事件桥

```ts
// CourseModuleToolbar.tsx
onClick={() => dispatchCourseToolbarAdd({ tab: "home", type: block.type })}

// HomeTab.tsx
useEffect(() => {
  const handler = (e: Event) => {
    const d = (e as CustomEvent<CourseToolbarAddEventDetail>).detail;
    if (d.tab === "home") void handleAddFromToolbar(d.type);
  };
  window.addEventListener(COURSE_TOOLBAR_EVENT_NAME, handler);
  return () => window.removeEventListener(COURSE_TOOLBAR_EVENT_NAME, handler);
}, [handleAddFromToolbar]);
```

---

## 4. Context

- 前置：P5a（HomeTab 编辑模式与 Undo/Redo）、P5b（7 大块渲染 + 编辑弹窗）、T3a（LessonPlanDetail 数据结构）。
- 涉及表：`courses`（cover_image_url / owner_id / start_date / level / class_time / semester）、`course_home_blocks`（block_type / content / position / course_id）、`lesson_plans` + `lesson_plan_weeks`、`profiles`（is_public_to_students / name / avatar_url / bio）。
- Storage bucket：`course-home-assets`（public read，owner write）。
- Edge functions：`pixabay-search`（关键字 → 图库缩略图）。
- 事件常量：`COURSE_TOOLBAR_EVENT_NAME`，跨组件通信；`"course:updated"` 用于同页面对话框保存后立即刷新。

---

## 5. Acceptance

1. 非 owner 用户打开班级页看不到 Switch / Toolbar / 표지 편집 按钮，也无法触发对应事件。
2. Owner 打开 편집 모드 → 右侧出现 Toolbar，点击任一 `+` 立即在最下方新增对应 block，且刷新页面后仍在。
3. 上传封面 → Hero 背景立即更新；另一台设备打开同一课程 5 秒内看到新封面（走 courses 通道）。
4. 在 `course_header` block 中改 title_override → Hero 大标题立即刷新（走 course-hero 通道），无需刷新页面。
5. 有教案的课程添加 `weekly_preview / songs_list / vocab_grid` 时，content 已按 §2.6 自动填入；无教案时创建的是空块但不报错。
6. 关闭 Dialog / 卸载 HomeTab 后再 mount，不出现重复订阅（可用 Supabase Realtime 后台 → 连接数验证）。
7. 学生视图始终没有 Toolbar；即使手动 `window.dispatchEvent(...)` 也不会写入（RLS 拒绝）。
