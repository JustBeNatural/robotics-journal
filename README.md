# 🤖 Robotics Learning Journal

记录机器人论文阅读、代码理解与实验过程，让每次学习都能找到来路和下一步。

## 🌿 Projects

> [!IMPORTANT]
> **这里是学习记录的总索引。** 点击项目名称，查看该项目的论文笔记、环境配置、日期日志和阶段总结。每个项目的学习过程集中保存在自己的目录中。

| Project | 方向 | Contents | 记录状态 |
| :--- | :--- | :--- | :--- |
| [**RAX**](projects/RAX/README.md) | 🖥️ 仿真 / 🦾 真机部署计划 | RoboDojo 交互接口、Franka 双电脑部署；包含本地代码、论文 PDF、学习路线、环境记录和日志入口。 | 📝 框架已建，运行结果待记录 |
| [**AgenticLab**](projects/AgenticLab/README.md) | 📖 论文阅读 | 待确认论文来源，整理阅读动机与学习目标，再逐步补充笔记。 | ⏳ 待读 |
| [**Atomic policy**](projects/atomic-policy/README.md) | 🧪 待明确 | 保留原有实验笔记位置，后续补充学习背景、实验目标与过程。 | ⏳ 待整理 |

状态描述的是本仓库的记录进度。实验是否成功，以项目日志中的实际结果与证据为准。

---

## 🧭 Quick Links

| 我想…… | 从这里进入 |
| :--- | :--- |
| 接着学习 RAX | [项目首页与下一步](projects/RAX/README.md) |
| 回看 RAX 的学习过程 | [日期日志](projects/RAX/logs/README.md) |
| 查看 RAX 的论文理解 | [论文笔记](projects/RAX/paper-notes.md) |
| 查看 RAX 的部署方法 | [环境记录](projects/RAX/setup.md) |
| 回顾阶段收获 | [RAX 总结](projects/RAX/summary.md) |
| 写今天的学习记录 | [学习日志模板](templates/learning-log.md) |
| 开始一个新项目 | [项目首页模板](templates/project.md) |
| 查找跨项目知识 | [知识笔记](notes/README.md) |

---

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

在仓库根目录检查本次变更，再按主题选择文件提交。保留可以追溯的命令、版本与结果。

```bash
git status --short
git diff --stat
# 检查后按需 git add、git commit 和 git push
```
