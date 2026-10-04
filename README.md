# Hina Learning Notes｜学习笔记

记录 Hina 在电子信息、嵌入式驱动、电机控制和仿真学习中的理解过程、疑问与阶段性练习。

## 仓库定位

- 本仓库：**我是怎么学会的**，以学习日记、大纲和逐步练习为主。
- [工程知识库](https://github.com/Hina-hdu/Hina-EngineeringKnowledge)：**开发时应该怎么做**，按项目、方法和验证证据组织；该仓库私有，无权限时链接不可访问。
- [ENVO0323 固件](https://github.com/Hina-hdu/ENVO0323)：实际工程源码，私有仓库；笔记不是固件，也不替代最新代码。

## 阅读入口

- [学习目录](LearningNotes/学习目录.md)
- [Hina 的学习笔记](LearningNotes/Hina的学习笔记.md)
- [ENVO0323 学习路线](LearningNotes/ENVO0323-Learning-Roadmap.md)
- [EEPROM 驱动大纲](LearningNotes/EEPROM-Driver-Outline.md)
- [APP 初始化练习](LearningNotes/ENVO0323-App-BringUp-Init-Guide.md)

使用 Obsidian 时，请把本仓库中的 **LearningNotes 文件夹**打开为仓库，而不是整个 Git 根目录。`[[双向链接]]` 在 Obsidian 中使用；GitHub 不会完整呈现这些链接。

## 备份与维护

- GitHub：https://github.com/Hina-hdu/Hina-LearningNotes
- 默认分支：`main`；当前可见性：**公开**，仅放适合公开的学习内容。
- 已有 Obsidian Git 插件继续使用；提交与推送是否成功，以插件状态或 GitHub 最新提交为准。插件文件与个人配置留在本机，不作为笔记内容继续追踪。
- 新设备请在 Obsidian 重新安装、启用 Git 插件，并单独配置 GitHub 登录；登录令牌不能写入本仓库。
- 手工备份前先检查 `git status`，再提交笔记与说明，最后推送；换设备开始编辑前先同步，遇到冲突不要强制覆盖。
- 自动整理工程知识的每天 21:00 任务**不是 GitHub 自动推送任务**。
- 本地的 `EngineeringKnowledge/` 是一个独立 Git 仓库，已被本仓库排除；不要在本仓库中强制添加它，也不要作为子模块重复上传。
- 不上传账号凭据、公司资料、原理图、工程源码快照或个人敏感信息。公开内容必须先检查；忽略规则不会清除旧提交中已经存在的文件。

## 使用边界

学习计划不等于已实现，编译成功不等于仿真通过，仿真通过不等于实机验证。涉及 PWM、采样极性、相序和保护的结论必须记录版本、条件和实测结果。

本仓库未声明开源许可证；不要据此推定第三方资料允许再分发。

配置日期：2026-10-04。
