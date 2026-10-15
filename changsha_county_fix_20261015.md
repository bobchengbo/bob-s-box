# 长沙县「大山冲」错配修复 + 合并脚本脆弱性处理（2026-10-15）

## 用户反馈
输入「长沙县大山冲」出的是「长沙」的报告（市级），且页面提示「已采用长沙县」但数据明显是太原/长沙套模板——匹配逻辑错把县级错配到上级地级市，且核心版横幅不透明。

## 根因
1. **匹配逻辑缺陷**：`generateReport` 原用 `Object.keys(CITY_DB).find(k => city.includes(k))`，`Array.find` 返回**第一个**命中（按对象插入顺序）。输入「长沙县大山冲」同时命中「长沙县」(len3) 与「长沙」(len2)，因「长沙」在对象中位置靠前被先命中 → 错配市级。
2. **横幅不透明**：原横幅只说「核心数据版」，不显示实际匹配到的行政区、也无回退提示，用户无法判断落点是否正确。
3. **历史坑（已暴露）**：之前二次合并 `merge_all.py` 把注入点算飞，导致 CITY_DB 提前 `};` 闭合、380 条 lite 漂到对象外、整段内联脚本语法崩。已从 git HEAD 恢复 442 版本重建。

## 修复（commit 968e757，已 push，Pages HTTP 200）
1. **最长优先匹配**：`Object.keys(CITY_DB).sort((a,b)=>b.length-a.length).find(k => city.includes(k))` —— 县级优先于上级市，实测「长沙县大山冲项目」→ 长沙县(lite)，「长沙」→ 长沙(lite)。
2. **横幅透明化**：核心版横幅现在显示 `已匹配行政区：<实际>`；若输入未精确收录而回退到上级行政区，提示「输入 X 未精确收录，已回退至匹配到的上级行政区 Y」；未收录则用默认模板。
3. **长沙县入库**：用 2024 统计公报真实数据（GDP 2266.35亿 +6.0%、常住人口145.31万、城乡居民人均收入57852元、主导产业 工程机械/汽车及零部件/电子信息、人口净流入）以单条 lite 条目安全插入 CITY_DB 闭合 `};` 前。**未重跑全量 rebuild**，避免再次触发脆弱合并脚本。
   - 校验：CITY_DB key 442→443，长沙县 `_lite:true`、gdp[4]=2266 通过；主内联脚本 `node --check` OK；最长匹配实测正确。

## 合并结果（苏浙 + 广东 + 广西 + 长沙县，2026-10-15 14:40+）
- 苏浙县组完成：江苏 40 + 浙江 53 = 93 县（真实 2024 数据，7 个有可支配收入，其余 None 未编造）。
- 重写稳妥版合并脚本 /tmp/merge_robust.js：**node 解析现有 CITY_DB → 仅保留非 _lite 深度库(63) → 丢弃全部旧 lite → 从 /tmp/lite_src.json 重建 lite → 重建 CITY_DB → 写盘前 node 校验（key 数=深度+新lite、长沙县在内、主内联脚本 node --check）**，校验失败不写盘。彻底规避旧 merge_all.py 的注入点算飞问题。
- 汇总 /tmp/parse_pref.py 收集 12 个数据源（8 组地级市 + 广东/广西/苏浙县 + 长沙县 extra）= 455 条去重 lite。
- 应用结果：深度 63 + 真实 lite 450 = **513 key**（长沙县 gdp[4]=2266 校验通过、最长匹配正确、inline 语法 OK）。
- commit 51ae7bb 已 push，Pages HTTP 200。
- 待归并：安徽/湖北/湖南/四川 4 组县仍在跑（子代理），回来后重跑 /tmp/parse_pref.py + /tmp/merge_robust.js 增量并入（幂等）。
- /tmp/merge_all.py（脆弱版）已弃用，勿再用。
- 湖南县组、苏浙/安徽/湖北/湖南/四川/广西 6 个县组仍在跑（此前 spawn 的子代理）。合并县数据时必须**重写 merge_all.py 为稳妥版**：用 node 解析 CITY_DB 过滤出非 `_lite` 的深度库 → 与 pref_*.py/prefcounty_*.py 重新生成 lite 块 → **写盘前 node 校验（语法 + key 数 + 目标县在内）**，校验失败不写盘。
- 当前 CITY_DB 仍残留 8 条早期测试城市（石家庄/唐山/郑州/武汉/成都/西安/沈阳/昆明）作为 lite 重复条目（它们也在 pref 省文件里），无害但下次 rebuild 一并清理（去重：保留非 `_lite` 深度 + 从省文件重建 lite）。
- 永久地址不变：https://bobchengbo.github.io/bob-s-box/project-research-tool.html

## 关键文件
- project-research-tool.html：generateReport 匹配（~7140 行，现最长优先）；核心版横幅 innerHTML（~5645 行）；长沙县 lite 条目插在 CITY_DB 闭合 `};` 前。
- /tmp/insert_csx.py：安全单条插入脚本（写盘前 node 校验），可复用。
- /tmp/merge_all.py：脆弱，待重写为「node 解析+校验」稳妥版再用于县批量合并。
