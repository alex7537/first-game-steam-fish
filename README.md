# Fish Lab

一个语言驱动的程序化鱼类生成原型，也是“做出第一个鱼类游戏并上架 Steam”的起点。

当前版本是无依赖的单页 Demo：输入中文描述，离线解析颜色、体型、尺寸和游速，并把生成的鱼放进同一个鱼缸。它不调用外部 AI 服务，也不需要 API key。

## 运行

直接用浏览器打开 `index.html`，或者在项目目录启动一个静态服务器：

```bash
python -m http.server 8000
```

然后访问 <http://localhost:8000>。

## 当前架构

```text
文本描述
   │
   ▼
离线关键词解析 ──► FishGenes ──► Canvas 鱼类绘制与游动
```

原型暂时放在单个 `index.html` 中，便于直接部署到 GitHub Pages。玩法确定后再拆分渲染、数据和游戏逻辑。

## 基因 Schema

```js
{
  color: "#ffc857", // 身体颜色
  size: 1.0,        // 整体尺寸倍率
  aspect: 1.2,      // 身体长宽比
  speed: 1.0        // 游动速度倍率
}
```

## 路线图

- [x] 文本描述生成鱼类
- [x] 多鱼同缸
- [ ] 保存和读取鱼类基因
- [ ] 杂交与遗传
- [ ] 成长、饥饿和互动循环
- [ ] 音效、音乐与完整美术风格
- [ ] 桌面版封装和 Steamworks 接入
- [ ] 商店页素材、测试与发布

## 部署

仓库根目录的 `index.html` 可直接作为 GitHub Pages 入口。在仓库 Settings → Pages 中选择从 `main` 分支部署即可。

## License

[MIT](LICENSE)
