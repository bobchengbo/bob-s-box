# 项目研判报告工具 — 4省54城真实数据入库（2026-10-14）

## 目标
把晋/苏/浙/粤 4 省全部地级市真实数据加入 `project-research-tool.html` 的 `CITY_DB`，
使工具可直接生成这 54 城（加原内置 11 城 = 共 65 城）的深度研判报告，无需实时联网。

## 做法
- 数据来源：各省统计局公报 / 省情网 / stats.gd.gov.cn 等权威口径（2024 GDP + 常住人口 + 增速）。
- 生成器：`/tmp/gen_cities.py`（54 城字段字典，含省级通用政策/合作模板 + 城市真实文化标签、优劣势、定位结论）
  + `/tmp/insert_cities.py`（对齐"苏州"块结构的 JS 渲染器，一次性写入 CITY_DB）。
- 每城块字段与苏州模板严格一致：gdp(5年回推)/industry/pop/popTrend/income/tourist/kpi(五维)/
  rating/ratingTitle/Desc/Tags/policy(4维)/policyTools(8项,按能级降级)/policyInsight/
  culture{stats(6键量化基线)/gdp/tourist/assets(4)/ipMatrix(3)/insight/gap}/
  location(铁空公)/locationInsight/strengths/risks/coop(3)/conclusion。

## 结果
- CITY_DB 城市数：11 → 65（新增 54：晋11 + 苏12 + 浙10 + 粤21）。
- 文件大小 344KB；`node --check` 通过；`Object.keys(CITY_DB)` 实求值 = 65 城，深圳块结构验证完整。
- 城市匹配：`Object.keys(CITY_DB).find(k => city.includes(k))` 对"深圳市/南京市/杭州市/太原市/云浮市/丽水市"均正确命中。
- git 提交 `bb0a7ed`。

## 验证
- 本地 `localhost:9876` → 200；公网 serveo → 200。
- 新地址：https://544e6c0f431200d5-124-65-151-250.serveousercontent.com/project-research-tool.html
  （serveo 免费版约 16 分钟断开，断后重启：`nohup ssh -R 80:localhost:9876 -o ServerAliveInterval=30 serveo.net >/tmp/serveo.log 2>&1 &`，取地址 `grep -oE "https://[a-z0-9.-]+serveousercontent\.com" /tmp/serveo.log`）

## 未完成项（非阻断）
- v2.0 计划的 `normalize()` 与"AI实时数据导入"UI（importAIBtn/importArea 节点）仍未落地；
  未知城市仍走 DEFAULT_DATA 参考基线（不再全"—"）。
- 实时架构维持"Aggregator 作数据引擎 + HTML 作呈现"，`reports/` 下柳州机械厂报告为验证样例。
- 54 城的 tourist(旅游人次)/culture.gdp 为按 GDP·人口回推的参考基线值，非逐城实测；
  如需更准可后续逐城补抓文旅统计公报。

## 临时文件
- /tmp/gen_cities.py（数据+模板）、/tmp/insert_cities.py（插入器）、/tmp/check_html.js（语法校验）
