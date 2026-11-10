# Robotics Learning Journal

记录我学习机器人论文、理解代码和进行实验的过程，方便以后回看和继续。

## 项目导航

| 项目 | 学习目标 | 当前记录 | 入口 |
| --- | --- | --- | --- |
| RAX | 理解方法与交互接口，逐步完成仿真和真机验证 | 已有代码、PDF 与学习计划；实际进度待更新 | [项目首页](projects/RAX/README.md) |
| AgenticLab | 待确认论文与学习目标 | 待读占位 | [项目首页](projects/AgenticLab/README.md) |
| Atomic policy | 待补充 | 保留原有空白实验笔记 | [项目首页](projects/atomic-policy/README.md) |

## 最近在学什么

当前已有 RAX 的学习计划，涉及 RoboDojo 仿真接口和 Franka 双电脑部署。下一步是补齐论文来源，选定最小任务，并记录第一次实际运行。

还没有填写日期日志。开始记录后，把最近一次日志链接放在对应项目的首页；本页只保留项目概览。

## 怎么记录

1. 打开项目首页，找到上次记录与下一步。
2. 在项目的 `logs/` 中创建 `YYYY-MM-DD.md`，记录当天的阅读、命令、排错、实验与疑问。
3. 稳定的环境步骤更新到 `setup.md`，论文理解更新到 `paper-notes.md`，阶段收获写进 `summary.md`；引用相关日志。
4. 结束时在项目首页补上这次日志链接，并写明下次从哪里继续。

一天一份日志即可，同一天的多次学习追加在同一文件里。复杂实验需要独立保存配置和多次结果时，再在项目下增加 `experiments/`。

## 目录约定

```text
projects/<项目>/
├── README.md          项目导航、学习目标与下一步
├── paper-notes.md     对论文的理解与疑问
├── setup.md           能重复使用的环境和运行方法
├── logs/              按日期保存学习过程
├── summary.md         阶段收获与未解决问题
└── assets/            笔记引用的图片、资料与结果图
```

新项目先写一个 README，需要时再增加其他文件。RAX 另保留现有实现代码目录。

- [学习日志模板](templates/learning-log.md)
- [项目首页模板](templates/project.md)
- [跨项目知识笔记](notes/README.md)

## 提交到 GitHub

在仓库根目录检查修改，按需要选择文件提交。日志中的计划、实际观察与个人解释分别写清楚，保留可追溯的命令、版本及结果。

```bash
git status --short
git diff --stat
# 检查本次要提交的文件，再 git add 和 git commit
```

代码、PDF 与原有资料已保留到对应项目；之前的研究地图、追踪表和想法索引合并为项目导航与笔记。旧路径的书签和本地配置需要更新。
