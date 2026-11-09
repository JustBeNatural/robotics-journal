# 每日 + 每周工作流

## 每次实验前（2 分钟）
1. 从 `03-daily-logs/TEMPLATE.md` 新建 `YYYY-MM-DD.md`。
2. 写目标、假设以及“什么结果算成功”。

## 实验中
3. 记录实际执行的命令、commit/依赖版本、观察到的输出、错误和证据路径。
4. 对需要持续追踪的问题建立 GitHub Issue。

## 结束时（5–10 分钟）
5. 分开写“事实”和“自己的解释”。
6. 写下一次最小验证，填写实验/Issue 链接。
7. 用 `git status` 检查敏感文件后提交。

```bash
git status
git add README.md 00-roadmap 01-paper-notes 02-environment 03-daily-logs 04-experiments 05-debugging 06-knowledge WORKFLOW.md
# 如有更新的其他安全文件，再按需单独 git add
git commit -m "docs: log RoboDojo observation-action investigation"
git push
```

## 每周复盘（15–20 分钟）
- 本周完成了哪些可验证的小目标？
- 哪三个理解发生了变化？
- 哪些问题还没有证据支持？
- 将可复用的结论整理进 `06-knowledge/`。
- 更新 `00-roadmap/roadmap.md`，补上日志和实验的链接。

## Git 使用建议
优先小规模的、有主题的提交；每天无需固定提交。避免把上游项目完整复制进仓库，按许可证保留署名和出处。
