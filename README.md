# 🎮 小游戏合集

> 这个仓库里的小游戏，都来自家里小朋友的许愿：“想要一个这样的游戏！”
> 大人负责把愿望写成代码，小朋友负责出主意和试玩。
> 放在这里，是为了把这些一起做游戏的时光好好保存。如果恰好也被别的小朋友喜欢，那就赚到了 🙂
>
> The little games in this repo all started as wishes from the kid at home: "I want a game like this!"
> The grown-up turned those wishes into code; the kid came up with the ideas and play-tested every one.
> They live here so we can keep these game-making days together. And if some other kid happens to love them too — bonus! 🙂

纯网页小游戏合集。所有游戏**零外部依赖**（无 CDN、无构建工具、无外部图片/音频），下载后断网双击即玩。

## 怎么玩

- 双击根目录 `index.html`：进入游戏选择页，点卡片开始。
- 也可以直接打开任意 `games/<游戏名>/index.html`。

## 游戏列表

| 游戏 | 类型 | 说明 |
|---|---|---|
| [头头药水大作战](games/potion-master/index.html) | 2D 弹幕守关 | 化身方块脑袋，用隐身/尖刺/护盾撑过 5 波攻势 |
| [废弃城堡大冒险](games/castle/index.html) | 2D 横版闯关 | 古堡与金字塔两张地图各 10 关，带宠物夺王冠 |
| [时空小队大冒险](games/hero-squad/index.html) | 2D 横版动作 | 小晚控时间、小猫宝瞬移、百变捏人造型决定技能 |

## 目录结构

```
web-games/
├── index.html          # 入口：游戏选择页
├── games/
│   ├── potion-master/  # 头头药水大作战（单文件）
│   ├── castle/         # 废弃城堡大冒险（单文件 + README）
│   └── hero-squad/     # 时空小队大冒险（单文件）
└── .gitignore
```

每个游戏一个自包含目录，互不依赖，无任何外部引用（无 CDN、无外链资源）。

## 添加新游戏

1. 在 `games/` 下新建目录，放入自包含的 `index.html`；
2. 打开根 `index.html`，在 `GAMES` 数组里加一条（标题/图标/简介/标签/路径）；
3. 完成。
