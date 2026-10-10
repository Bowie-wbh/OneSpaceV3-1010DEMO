# OneSpace 设计规范与组件规范

> 适用项目：`OneSpaceV3-1010DEMO`；整理日期：2026-10-10。
> 本文依据当前源码整理，供开发者和 AI 在后续界面修改前阅读。标为「现状」的是已实现行为；标为「修改约定」的是后续应遵循的规则；标为「待统一」的内容尚未完整实现。本次仅整理文档，未修改 UI，也未进行浏览器视觉验收。

## 1. 修改前先读

1. 保持「对话式星座管控平台」定位：紧凑的信息布局、深浅双主题、蓝色/天蓝色主操作、圆角卡片、细边框，深色界面辅以柔和光晕。
2. 修改颜色先查 [全局样式](src/index.css)，修改布局先查 [App](src/App.tsx)，修改业务状态先查 [共享类型](src/types/index.ts)。
3. 优先复用现有业务组件和语义样式；当前没有统一的 `Button`、`Card`、`Modal` 等基础组件库，不要假设它们已存在。
4. 跨面板的状态与行为沿用 Props 和回调传递；筛选、展开、悬停等局部状态保留在组件内。
5. 任何界面改动均检查深浅主题、长文本、空数据、禁用状态和实际可用宽度；不得仅以全屏宽度判断分栏内的布局。
6. 修改公共颜色、尺寸、交互或组件边界时同步更新本文。若源码与文档发生偏差，先核对实际实现，再修订规范。

## 2. 实现基础与源码入口

| 内容 | 当前实现 / 入口 |
| --- | --- |
| UI 技术 | React 19、TypeScript、Vite 6 |
| 样式 | Tailwind CSS 4，通过 `@tailwindcss/vite` 接入；主题在 `src/index.css` 中定义，无独立 Tailwind 配置文件 |
| 图标 | `lucide-react`，以线性图标为主 |
| 图表 | 部分遥感详情使用 `recharts`；其他可视化也有直接使用 HTML/SVG 的实现 |
| 地图 | `react-leaflet` / Leaflet 负责 2D；Cesium 负责 3D；`satellite.js` 用于轨道计算 |
| Cesium 资源 | [index.html](index.html) 从 CDN 引入 Cesium 1.125.0 的脚本和样式，组件使用全局 `Cesium` |
| 页面编排 | [src/App.tsx](src/App.tsx) |
| 主题与全局视觉 | [src/index.css](src/index.css) |
| 通用业务类型 | [src/types/index.ts](src/types/index.ts) |
| 遥感业务类型 | [src/types/earthDemoTypes.ts](src/types/earthDemoTypes.ts) |
| 数据与演示素材 | `src/data/`、`src/assets/`、`public/assets/` |

修改约定：沿用 TSX 函数组件、明确的 Props 类型以及 Tailwind 类名；动态宽度、地图坐标、业务颜色等计算值可使用 `style`。新样式不应另建一套与现有主题脱节的色系。依赖中存在 `motion` 不代表现有动效统一由它实现。

## 3. 页面结构与布局

### 3.1 信息架构

主入口为统一工作区 `workspace`，不是三个独立一级页面。顶部控制展示方式和业务看板，左侧管理对话历史，右侧可打开卫星面板。

| 状态 | 可选值 | 当前初始值 |
| --- | --- | --- |
| 主题 `ThemeMode` | `dark` / `light` | `dark` |
| 展示模式 `WorkspaceViewMode` | `split` 对话+看板 / `kanban` 仅看板 / `chat` 仅对话 | `split` |
| 看板 `WorkspaceKanbanFilter` | `task` 任务管理 / `health` 健康管理 / `innovative` OneEarth | `task` |
| 左侧导航 | 展开 / 收起 | 展开 |
| 右侧卫星面板 | 打开 / 关闭 | 关闭 |

注意：`App.tsx` 中有关看板默认进入 OneEarth 的注释与初始化值不一致；当前实际初始化为 `task`。

### 3.2 关键尺寸

以下 rem 对应像素按根字号 16px 换算；超大屏根字号会变化。JS 中显式写入的 px 宽度不随 rem 同步放大。

| 部位 | 现状 |
| --- | --- |
| 主内容外间距 | 顶栏 `px-3 pt-2`；主体 `px-3 pb-3`；面板之间常用 12px |
| 侧栏 | 高度 `calc(100vh - 24px)`，左距 12px；展开 `w-64 sm:w-72`（256 / 288px），收起 `w-14 sm:w-16`（56 / 64px） |
| 对话分栏 | 初始宽 460px；拖拽最小 320px，最大 `max(360, 容器宽度 - 360)` |
| 对话分隔条 | 命中宽 `w-3`，可见指示线 2px；悬停/拖动时显示蓝色渐变线 |
| 对话分隔条双击 | 设为当前工作区宽度的 40%；容器不可用时回退到 460px |
| 看板 | `flex-1`，最小宽 360px、最小高 480px |
| 卫星面板 | 初始 380px；拖拽下限 280px，上限 `min(650, max(300, 主体宽度 - 600))`；双击恢复 380px |
| 完整对话 / 输入区域 | 常用 `max-w-3xl mx-auto`，约 768px |

修改约定：保持 `flex-1`、`min-w-0`、`min-h-0` 与内部滚动的组合。列表区域使用 `overflow-y-auto`，容器使用 `overflow-hidden`，避免长内容把整页撑开。拖拽时锁定文本选择、使用 `col-resize`，结束后恢复并清理监听。

### 3.3 响应式与大屏

现状使用 Tailwind 默认断点：`sm` 640px、`md` 768px、`lg` 1024px、`xl` 1280px、`2xl` 1536px；没有在主题中重定义断点。组件常用 `text-xs sm:text-sm`、响应式内边距及网格列数。

`src/index.css` 额外设置根字号：宽度 ≥1920px 为 16px、≥2560px 为 17px、≥3840px 为 18.5px；低于 1920px 未在该文件单独固定字号。

待统一：外层工作区仍为横向分栏，包含固定宽度和最小尺寸；已有局部响应式样式不等于完整移动端适配。窄屏可能出现压缩或裁切，拖拽目前使用鼠标事件。新增页面应明确窄屏排列与滚动策略，不直接复制桌面尺寸作为移动端规范。

## 4. 主题与颜色

### 4.1 主题机制

`App` 中的主题状态同步到 `document.documentElement` 的 `.dark` / `.light`；`index.html` 初始也带 `.dark`。Tailwind 深色变体由 `@variant dark (&:where(.dark, .dark *))` 定义。当前没有主题本地存储或跟随系统主题的实现。

浅色以白色表面、冷灰背景、蓝色操作为主；深色以深蓝黑背景、半透明表面、天蓝操作和柔和边缘光为主。

### 4.2 全局颜色变量

下表列出 `src/index.css` 中实际定义的变量，可直接作为修改入口。

| CSS 变量 | 浅色 | 深色 | 用途 |
| --- | --- | --- | --- |
| `--color-bg-base` | `#f4f7fb` | `#070a12` | 页面底色 |
| `--color-bg-canvas` | `#f8fafc` | `#0d111d` | 内容画布 |
| `--color-bg-surface` | `#ffffff` | `rgba(14,18,30,.88)` | 表面 |
| `--color-bg-surface-elevated` | `#ffffff` | `rgba(18,24,40,.95)` | 提升层级的表面 |
| `--color-bg-sidebar` | `#ffffff` | `rgba(12,16,28,.94)` | 导航背景 |
| `--color-bg-sidebar-capsule` | `#ffffff` | `#111728` | 胶囊背景 |
| `--color-bg-input` | `#ffffff` | `rgba(15,20,34,.92)` | 输入背景 |
| `--color-bg-hover` | `#f1f5f9` | `rgba(255,255,255,.05)` | 悬停 |
| `--color-bg-active` | `#e2e8f0` | `rgba(56,189,248,.16)` | 激活 |
| `--color-bg-card` | `#ffffff` | `rgba(15,20,34,.75)` | 卡片 |
| `--color-bg-card-hover` | `#f8fafc` | `rgba(22,29,48,.9)` | 卡片悬停 |
| `--color-bg-badge` | `#eff6ff` | `rgba(56,189,248,.12)` | 标签 |
| `--color-border-subtle` | `#e2e8f0` | `rgba(255,255,255,.07)` | 常规边界 |
| `--color-border-strong` | `#cbd5e1` | `rgba(255,255,255,.14)` | 强调边界 |
| `--color-border-focus` | `#3b82f6` | `#38bdf8` | 焦点边框 |
| `--color-text-primary` | `#0f172a` | `#f1f5f9` | 主要文字 |
| `--color-text-secondary` | `#475569` | `#94a3b8` | 次要文字 |
| `--color-text-muted` | `#64748b` | `#64748b` | 弱化文字 |
| `--color-text-inverse` | `#ffffff` | `#090c15` | 强色背景上的文字 |
| `--color-accent-primary` | `#2563eb` | `#38bdf8` | 主操作 |
| `--color-accent-primary-hover` | `#1d4ed8` | `#0ea5e9` | 主操作悬停 |
| `--color-accent-light` | `#dbeafe` | `rgba(56,189,248,.12)` | 主色浅底 |
| `--color-accent-glow` | `rgba(37,99,235,.15)` | `rgba(56,189,248,.3)` | 主色光晕 |
| `--color-success` | `#059669` | `#34d399` | 成功 |
| `--color-success-bg` | `#ecfdf5` | `rgba(52,211,153,.12)` | 成功底色 |
| `--color-warning` | `#d97706` | `#fbbf24` | 警示 |
| `--color-warning-bg` | `#fffbeb` | `rgba(251,191,36,.12)` | 警示底色 |

已有语义类：`bg-base`、`bg-canvas`、`bg-surface`、`bg-surface-elevated`、`bg-sidebar`、`bg-sidebar-capsule`、`bg-input-theme`、`bg-card-theme`、`bg-badge-theme`、`border-subtle`、`border-strong`、`border-focus`、`text-primary`、`text-secondary`、`text-muted`、`text-inverse`、`text-accent`、`bg-accent`、`bg-accent-hover`、`border-accent`。

注意：不是每个变量都定义了对应语义类；`border-focus` 只是边框颜色类，并不会自动限定为焦点状态。需要状态时显式使用 `focus:` / `focus-visible:` 或 `focus-within:` 样式。

### 4.3 业务色与状态

| 语义 | 现有主要表达 |
| --- | --- |
| 操作、选中、信息 | 浅色 blue，深色 sky；选中项常配浅底、边框、加粗文字 |
| 成功、正常、可发送 | emerald；绿色标签、圆点或完成图标 |
| 等待提醒、关注 | amber；必要时倒计时与呼吸提示 |
| 异常、告警、终止 | rose / red；图标与文字共同提示 |
| 未激活、禁用、待执行 | slate、较弱透明度、禁用光标 |
| 地图火点 | `#dc2626`，白色描边，红色光圈 |
| 地图古建筑点 | `#eab308`，白色描边，黄色光圈 |

待统一：暂无全局 error/danger 色变量；现有组件混用语义变量、Tailwind 色阶和 `#0c101c`、`#111728` 等局部颜色。仅修改全局变量不会覆盖所有组件。`ToolCards` 的失败状态当前属于灰态并以文字说明，不是所有失败都已采用红色。

修改约定：一般表面与文字优先使用语义变量；延续现有业务色时同时处理深浅主题。状态不应只靠颜色区分，保留中文状态文字或图标。

## 5. 字体、间距、圆角与图标

### 5.1 字体层级

全局无衬线字体栈为系统字体，含 `-apple-system`、`Segoe UI`、`Roboto`、`PingFang SC`、`Microsoft YaHei` 等；不依赖额外下载字体。编号、时间、数值、指令等常用 `font-mono`。

| 场景 | 现有常用写法 | 16px 根字号下大小 |
| --- | --- | --- |
| 密集数据注释 / 标签 | `text-[10px]`、`text-[11px]` | 10 / 11px |
| 辅助信息 / 紧凑正文 | `text-xs` | 12px |
| 正文 / 常规菜单 | `text-sm` | 14px |
| 局部标题 | `text-base`、`text-lg` | 16 / 18px |
| 品牌强调 | `text-2xl` 等，配 `font-black` | 24px |
| 字重 | `font-medium`、`font-semibold`、`font-bold` | 500 / 600 / 700 |

修改约定：延续紧凑层级，但不要把 10px 扩展为所有正文的默认字号。长标题使用 `min-w-0` + `truncate`；需要完整阅读的正文允许换行。数字保留单位，时间补零。品牌保留文字标识：侧栏为「021模型 | OneSpace」，收起导航后顶栏使用冒号分隔的品牌呈现。

### 5.2 间距与圆角

| 场景 | 常用值 |
| --- | --- |
| 图标与文字 / 紧凑行 | `gap-1`、`gap-1.5`、`gap-2`（4 / 6 / 8px） |
| 控件 / 卡片间距 | `gap-2.5`、`gap-3`、`gap-4`（10 / 12 / 16px） |
| 卡片内边距 | `p-3`、`p-3.5`、`p-4`（12 / 14 / 16px） |
| 弹窗内边距 | `p-5 sm:p-6`（20 / 24px） |
| 小标签 / 内嵌数据块 | `rounded-lg`（8px） |
| 按钮 / 表单 / 子卡片 | `rounded-xl`（12px） |
| 主卡片 / 侧栏 / 消息块 | `rounded-2xl`（16px） |
| 输入容器 / 主要弹窗 | `rounded-3xl`（24px） |
| 胶囊 / 圆形操作 | `rounded-full` |

这些是当前使用模式，不是已经封装的尺寸 Token。新增同类元素优先从上述档位选择，避免无目的增加任意值。

### 5.3 图标与点击区域

图标使用 `lucide-react`。密集操作常为 `w-3.5 h-3.5` 或 `w-4 h-4`，常规操作为 `w-5 h-5`；现有按钮命中区域常见 32、36、44px。图标与文字并排时使用 `shrink-0` 防止被压缩。

修改约定：纯图标按钮提供可理解的 `aria-label`，可同时保留 `title`；不要只用悬停提示承载必要说明。装饰性图标不应重复朗读。当前代码多处仅有 `title`，可访问名称并未统一补齐。

## 6. 阴影、玻璃效果与动画

| 样式 | 实际入口与表现 |
| --- | --- |
| 卡片阴影 | `shadow-theme-card` → `--card-shadow`；浅色轻投影，深色黑色投影配细描边 |
| 卡片悬停 | `shadow-theme-card-hover` → `--card-shadow-hover`；深色增加 sky 光晕 |
| 浮层阴影 | `shadow-theme-floating` → `--floating-shadow` |
| 玻璃面板 | `glass-panel` 使用 `--glass-bg`、`--glass-border`、`--glass-blur` |
| 浅色玻璃值 | 背景 `rgba(255,255,255,.9)`，边框 `rgba(226,232,240,.9)`，模糊 16px |
| 深色玻璃值 | 背景 `rgba(14,18,30,.82)`，边框 `rgba(255,255,255,.07)`，模糊 20px |
| 环境光晕 | `.dark .ambient-glow` / `.light .ambient-glow` 及 App 内绝对定位光斑 |
| 常用过渡 | `transition-colors` / `transition-all`；显式时长常见 200 / 300ms；按下 `active:scale-95` |
| 光晕呼吸 | `animate-glow-pulse`，8s、ease-in-out、循环 |
| 光晕旋转 | `animate-glow-shift`，16s、ease-in-out、循环 |
| 渐变文字 | `text-blue-flow`，5s 线性循环 |
| 任务条目入场 | `animate-stage-item-in`，0.4s |
| 任务列表滚动 | `animate-marquee-vertical`，线性循环，时长需调用处提供 |

注意：CSS 注释声称玻璃效果仅深色启用，但 `.glass-panel` 规则并未限定 `.dark`，实际两种主题都可生效。

待统一：组件引用了 `animate-fadeIn`、`animate-in`、`fade-in`、`slide-in-from-right` 等类，但当前全局样式和依赖配置未提供这些类的明确实现；不能把它们视为已验证可用动效。当前也未提供统一的 `prefers-reduced-motion` 处理。

修改约定：背景光晕保持 `pointer-events-none`，避免遮挡操作。新增动效必须有实际定义，优先服务状态变化；新增持续动画时考虑减少动态效果偏好。

## 7. 通用组件模式

### 7.1 按钮与选择项

| 模式 | 当前典型组合 / 行为 |
| --- | --- |
| 主按钮 | `bg-blue-600 text-white dark:bg-sky-400 dark:text-slate-950`；悬停加深蓝或提高天蓝亮度 |
| 次按钮 | 浅色白底或 slate 浅底、深色深蓝底；细边框与弱阴影 |
| 图标按钮 | 圆角矩形或圆形；悬停 slate / 白色低透明底 |
| 快捷回复 | `px-3.5 py-1.5 rounded-full text-xs font-semibold border` |
| 当前选中 | 颜色 + 底色/边框共同变化；如顶栏视图切换、历史对话选择 |
| 禁用 | 原生 `disabled` + 不可用样式 + 清楚的状态说明；事件逻辑同时阻止非法执行 |

修改约定：同一操作组保持相同高度、圆角和字重；表单中非提交按钮显式设置 `type="button"`。优先保留原生 button 语义，并提供可见键盘焦点。

### 7.2 卡片、列表和表格

现状以「标题/操作区 + 内容区」组织卡片，常用细边框、`rounded-xl` / `rounded-2xl`、12–16px 内边距。选中列表项使用蓝/天蓝边框和浅色底。密集数据用小字号、等宽数字、状态圆点和标签组合；长列表内部滚动，空列表提供文字占位。

全局滚动条为 5px；`.custom-scrollbar` 为 4px，透明轨道、圆角滑块，悬停使用主色。Firefox 使用 `scrollbar-width: thin`，实际宽度由浏览器控制。

修改约定：表格标题、单位、状态含义保持明确；新增列时检查横向空间。不要为了塞入更多字段无限缩小字号。复杂图表应保留图例、单位和空数据反馈。

### 7.3 输入与表单

常规输入以 `rounded-xl`、细边框、浅灰/深蓝背景、`text-xs sm:text-sm` 为主；焦点使用 blue / sky 边框。错误提示为 rose 文字、淡红背景及警示图标。

修改约定：字段应有可关联的 label；必填、校验错误和提交结果要明确。占位符不能替代标签，禁用不能只靠变灰。新增字段复用同一表单的高度与边距。

### 7.4 弹窗、抽屉和层级

现状多数弹窗使用 `fixed inset-0` + 遮罩 + 居中面板；配置弹窗 `max-w-md rounded-3xl`，影像预览用更深遮罩。`TimeSeriesUploadModal` 点击遮罩关闭、面板内部阻止冒泡；不同弹窗的关闭行为需逐个核对。

| 层级 | 现有实例 |
| --- | --- |
| `z-0` | 背景光晕 |
| `z-10` | 主体区域 |
| `z-20` / `z-30` | 导航、顶栏容器、分隔条 |
| `z-40` | 遥感历史抽屉 |
| `z-50` | 常规弹窗、下拉、部分提示 |
| `z-[1000]` / `z-[2000]` / `z-[99999]` | 部分地图浮层和健康详情弹窗 |

待统一：这些是分散的当前取值，不是统一层级系统；数值还受父级层叠上下文影响。当前没有通用 Modal，也没有统一的 Esc 关闭、焦点锁定/恢复和 `role="dialog"` / `aria-modal` 机制。

修改约定：新增弹窗检查与地图及其他浮层的叠放关系，必要时使用 Portal；补齐键盘与焦点行为。不要继续以不断增大 z-index 作为通用解决方案。

## 8. 业务组件索引与使用约定

下表为职责和关键契约索引，不替代各文件完整 Props 定义。

| 组件 / 源码 | 职责与关键契约 |
| --- | --- |
| [Sidebar](src/components/Sidebar.tsx) | 左侧品牌、工作区入口、对话历史、搜索、主题切换；展开状态、选中会话和主题由外部控制，搜索词和搜索框显隐在内部管理 |
| [Header](src/components/Header.tsx) | 视图切换、看板筛选、在轨卫星入口和时钟；通过 `onWorkspaceViewModeChange`、`onWorkspaceKanbanFilterChange`、`onToggleOrbitForecast` 通知父级；下拉监听外部点击关闭 |
| [ChatInputArea](src/components/ChatInputArea.tsx) | 输入、附件、新对话、发送/终止；消费 `isGenerating`、`disabled`、`prefillPrompt`，通过 `onSendMessage(message, attachments)` 等回调通知父级 |
| [ChatConversation](src/components/ChatConversation.tsx) | 根据 `ChatMessage[]` 展示消息和业务卡片；确认窗口、确认任务、快捷回复等使用专门回调，保持消息与业务状态关联 |
| [ToolCards](src/components/ToolCards.tsx) | 一轨成像状态胶囊；根据 satellites 推导可用状态，仅 `sendable` 可点击并触发 `onSelectSingleOrbit` |
| [FlowStepsTimeline](src/components/FlowStepsTimeline.tsx) | 对话、任务看板、创新应用复用的流程时间线；关键输入为 `steps`、`currentStepIndex`，支持标题、折叠与结果影像/火点 |
| [TaskManagementKanban](src/components/TaskManagementKanban.tsx) | 任务列表、详情、流程和成果；`injectedTask` 接收外部任务，`focusRequestId` 用于重新聚焦；回放通过 `onReplayTask`、`onReplayPhase`、`onSkipReplay` 联动对话 |
| [SatelliteSplitPanel](src/components/SatelliteSplitPanel.tsx) | 右侧卫星状态面板；父级提供 `width`、选中 ID、列表与关闭回调；`onSelectSatellite` 返回 Satellite 对象 |
| [TimeSeriesUploadModal](src/components/TimeSeriesUploadModal.tsx) | 当前实际是长时序监测配置表单；`isOpen` 控制显隐，`onSubmit(TimeSeriesTaskData)` 返回来源、日期、标题和天数；名称虽带 Upload，但当前没有文件上传控件 |
| [HealthCheckView](src/components/HealthCheckView.tsx) | 健康管理容器，接收 selectedSatellite 和视图模式，组织健康信息并可打开卫星列表 |
| [HealthStatusDashboard](src/components/HealthStatusDashboard.tsx) | 分系统健康、异常和详情；按卫星信息和 `overviewTimeSpan` 展示；局部分页有 3 / 5 / 10 / 20 条选项 |
| [InnovativeAppView](src/components/InnovativeAppView.tsx) | OneEarth / 创新应用容器，组织 2D/3D 地图、业务场景和遥感浮层；通过 `onLaunchApp` 与工作区任务规划联动 |
| [CesiumGlobe](src/components/earthDemo/CesiumGlobe.tsx) | 3D 地球、空间点和轨道展示；依赖全局 Cesium，相关坐标和标记类型沿用 data 中定义 |
| [PointMindMapOverlay](src/components/earthDemo/PointMindMapOverlay.tsx) | 围绕选中点展示数据类型与层级；关键输入为 `point`、`screenPos`、`isOpen`，可回传所选数据类型层级 |
| [HistoryDataDrawer](src/components/earthDemo/HistoryDataDrawer.tsx) | 历史遥感记录抽屉；父级管理类别、记录选择和足迹叠加，内部管理搜索、状态筛选和时间排序 |
| [RecordDetailModal](src/components/earthDemo/RecordDetailModal.tsx) | 遥感记录详情；`record` 为 null 时不渲染，通过 `onOverlayFootprint` 与地图联动 |
| [WeatherForecastBar](src/components/earthDemo/WeatherForecastBar.tsx) | 机场天气和日期/逐小时气象展示；对外提供机场选择与关闭回调 |

以下文件存在于源码中，但当前扫描未发现其他 TSX 文件对它们的引用，不能直接当作工作区已经接入的组件：

| 文件 | 当前定义的用途 |
| --- | --- |
| [OrbitForecastCard](src/components/OrbitForecastCard.tsx) | 卫星预报卡片；接收 satellites、状态切换和可选关闭回调 |
| [TelemetryInterpretationDashboard](src/components/TelemetryInterpretationDashboard.tsx) | 遥测解译面板，具有系统、时间、正常/异常筛选 |
| [SpatialFilterPanel](src/components/earthDemo/SpatialFilterPanel.tsx) | 空间点列表与火点/建筑筛选，接收选中点及切换回调 |

### 8.1 对话输入的现有交互

- Enter 发送，Shift+Enter 换行；空文本且无附件时不可发送。
- 生成中主按钮变为 rose 色终止按钮，调用 `onStopGenerating`。
- `disabled` 阻止输入和发送/附件操作；新建对话按钮仍可用，以便开始新任务。
- 文件选择支持 `.csv,.xlsx,.json,.txt,.kml,.geojson`，可多选、删除预览项；此为前端选择限制，不代表已有服务端校验或上传能力。
- 发送后清空文本和附件；预填内容同步后聚焦输入框。
- 文本框随内容增高：JS 上限 180px，CSS 另有 `max-h-44`，实际还受根字号与 CSS 限制，不应当作已经统一的高度值。
- 待完善：当前 Enter 处理未检查输入法组合状态；调整键盘行为时需避免中文输入法确认候选词触发发送。

### 8.2 任务与流程状态

共享 `FlowStepItem.status` 为 `pending | running | success | warning | error`。新增流程应延续该类型，使用稳定 key、步骤说明和明确结果，避免用文字匹配代替状态。

卫星 `status` 为 `in-bound | upcoming`；建链 `linkState` 包含 `connecting`、`link-success`、`link-failed`、`start-success`、`start-failed`、`sendable`、`window-closed`。成功的过程提示不等于可发送；ToolCards 只在 `sendable` 时可操作，在其推导状态为 `window-closed` 时不显示。

长时序配置会校验日期完整性和结束日期不得早于开始日期，天数包含首尾两天；成功提交后调用关闭回调。当前默认日期、部分天气、任务和回放数据属于演示内容，不应升格为产品长期业务约束。

### 8.3 地图与图表的特殊约定

2D 与 3D 空间点保持相同业务配色；选中状态增加光圈与发光，不改变点本体尺寸，避免选中时位置感跳动。Leaflet 的图标与 Cesium 的 Canvas 标记分别实现，修改颜色或选中效果时应同时核对。

地图底图、轨迹、空间点、浮层的交互层应保持清晰，背景和纯装饰不要吞掉鼠标事件。数值、坐标、单位、图例与详情状态应与所选对象一致。

待统一：HistoryDataDrawer 以及部分地图菜单/提示使用固定深色，而非完整跟随全局主题；当前不能承诺所有 OneEarth 浮层均支持浅色。图表和地图色值也有内联实现，改主题时需要一并检查。

## 9. 后续组件编写约定

以下是基于现有项目组织方式的维护约定，不表示已完成基础组件抽取。

1. 组件名和文件使用 PascalCase；Props 使用 `XxxProps`。外部事件使用 `onXxx`，内部处理使用 `handleXxx`；布尔状态保持 `isXxx` / `showXxx` 等清楚含义。
2. 业务共享类型优先放在 `src/types/`；地图专用类型也有存放在 `src/data/mockRemoteSensingData.ts` 的现状，扩展前确认现有定义，不重复创建不兼容类型。
3. 全局主题、工作区模式、选中卫星等跨组件状态沿用父级控制；不在多个子组件中复制同一份业务真值。
4. 展示与事件回调保持明确分工；新增任务操作接入现有任务/消息链路，避免单独写一个视觉上成功但未联动的按钮。
5. 计时器、鼠标事件、地图对象等在生命周期结束时清理；定时刷新避免引起整块布局跳动。
6. 复用布局时优先使用现有语义类；当重复模式确实需要抽取时再建立 Button、Badge、Panel、Dialog 等基础组件，显式定义变体和状态。
7. 保留已有关键交互 ID，修改其语义或入口时检查关联逻辑。列表 key 优先使用稳定业务 ID。

下面是使用现有语义类的新增卡片写法示例，不是项目里已经存在的公共 Card API：

```tsx
<section className="rounded-2xl border border-subtle bg-card-theme p-4 shadow-theme-card">
  <h2 className="text-sm font-semibold text-primary">任务概览</h2>
  <p className="mt-2 text-xs text-secondary">请选择需要查看的任务。</p>
  <button
    type="button"
    onClick={onOpenTask}
    className="mt-3 rounded-xl bg-accent bg-accent-hover px-3 py-2 text-xs font-semibold text-inverse transition-colors focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-[var(--color-border-focus)]"
  >
    查看任务
  </button>
</section>
```

## 10. 修改后的检查清单

- [ ] 深色、浅色均检查文字对比、边框、输入占位和操作状态；地图固定深色区域单独核对。
- [ ] `split`、`kanban`、`chat` 三种模式及三类看板均不破坏原有入口和联动。
- [ ] 侧栏展开/收起、卫星面板开关、两个分隔条拖动与双击后布局正确。
- [ ] 窄窗口与大屏根字号变化后无新增裁切；长列表内部可滚动，长标题不挤掉按钮。
- [ ] 新增可交互控件具有默认、悬停、焦点、选中/禁用等适用状态；仅图标控件有可访问名称。
- [ ] 对话发送、中文输入法、附件、生成终止、空数据、失败提示与确认后状态保持清楚。
- [ ] 弹窗和地图浮层层级正确；新增弹窗检查关闭、键盘操作和焦点返回。
- [ ] 若修改 TSX/CSS，执行 `npm run lint`（当前实际为 `tsc --noEmit`）和 `npm run build`，并做相应浏览器验证；两条命令都不能替代视觉检查。
- [ ] 若仅修改本文，检查源码链接、参数和现状说明即可；本文整理本身未声称运行过 UI 或通过视觉测试。

## 11. 已知待统一项

| 项目 | 现状及后续方向 |
| --- | --- |
| Token 覆盖 | 全局变量与局部固定颜色并存；后续按组件逐步收敛，避免只改变量导致局部主题不一致 |
| 基础组件 | 尚无统一 Button/Card/Modal；重复样式较多，需实际抽取后才能视作公共 API |
| 层级 | 多套 z-index 与局部层叠上下文并存；建立统一层级前需检查地图与 Portal |
| 动画 | 部分引用类未见定义；新增或依赖前先确认生成 CSS 中存在对应规则 |
| 可访问性 | 对话输入法、焦点样式、弹窗键盘/焦点行为、减少动态效果未统一完成 |
| 响应式 | 桌面分栏优先；窄屏、触控拖拽和高度不足场景需要单独设计和验证 |
| 主题 | 当前默认深色且不持久化，地图部分浮层固定深色；完整主题覆盖仍需逐项验证 |
| 文档与注释 | 个别注释与代码不一致；以已核实的实现为依据更新，而不是把注释直接当作规范 |

维护本文时保留「现状 / 修改约定 / 待统一」的区别；不要将建议写成已经实现的能力。
