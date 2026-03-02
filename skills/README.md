# Skills 目录

存放 OpenClaw Agent 技能包。

## 结构规范

每个 skill 是一个独立目录，包含：

```
skill-name/
├── SKILL.md          # Skill 定义（必需）
├── script.sh         # 执行脚本（可选）
├── config.json       # 配置文件（可选）
└── README.md         # 说明文档（可选）
```

## 添加新 Skill

1. 在本目录创建新文件夹
2. 编写 SKILL.md
3. 提交 PR
