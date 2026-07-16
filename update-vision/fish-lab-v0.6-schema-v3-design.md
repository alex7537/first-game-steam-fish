# Fish Lab v0.6 设计方案：Schema v3 多体型 + 种子纹理

> 目标：保留“同一基因永远渲染成同一生物”的确定性，同时把外观空间从鱼类变体扩展到可遗传的水生生物家族。
> 前置条件：v0.5 的存档、饥饿和成长循环完成并稳定后再实现，本方案不改变 v0.5 的开发顺序。

---

## 1. 设计原则

- **LLM 负责理解，Schema 负责表达，SVG 负责确定性呈现**，不把 diffusion 或 flow matching 放进核心玩法链路。
- `bodyPlan` 决定身体拓扑，每种拓扑由独立渲染器负责，避免继续在单个 `fishSvg()` 中堆条件分支。
- 基因仍是身份，成长、饥饿、位置和速度仍是运行时状态，喂养不得修改任何基因字段。
- 随机外观必须由基因内的种子驱动，同一份 JSON 在不同设备和刷新后应得到相同结果。
- v2 存档必须可以自动迁移，未知版本必须明确报错，不能静默生成畸形生物。

## 2. Schema v3

Schema 使用 `bodyPlan` 作为判别字段，公共字段由所有生物共享，体型专属字段集中放在 `anatomy` 中。

```js
{
  schemaVersion: 3,
  generatedBy: "ai", // ai / local / bred / evolved
  id: "FL-108",
  name: "夜光伞鱼",
  generation: 3,
  parents: ["FL-071", "FL-089"],
  mutations: ["生物荧光"],

  bodyPlan: "jellyfish", // fish / octopus / jellyfish / eel / crab
  color: "hsl(228 72% 45%)",
  accent: "hsl(182 86% 70%)",
  pattern: "云斑",
  patternSeed: 3489217314, // uint32
  opacity: 0.72,
  glow: 0.65,
  size: 1.0,
  fatness: 1.1,
  eyeCount: 2,
  eyeSize: 1.0,
  speed: 0.8,
  parts: ["发光", "长触须"],

  anatomy: {
    bellShape: "圆伞",
    tentacleCount: 8,
    tentacleLength: 1.3,
    ribbonCount: 4
  }
}
```

### 2.1 公共字段

`color`、`accent`、`pattern`、`patternSeed`、`opacity`、`glow`、尺寸、眼睛、速度和 `parts` 跨体型可遗传，因此鱼和水母的后代可以稳定表现为发光的半透明鱼。

### 2.2 体型专属字段

| bodyPlan | anatomy 字段 | 渲染重点 |
|---|---|---|
| `fish` | `shape / tail / fin / mouth` | 椭圆身体、尾鳍、背腹鳍与嘴型 |
| `octopus` | `mantleShape / tentacleCount / tentacleLength / webbing` | 头胴、腕足曲线、蹼膜与吸盘 |
| `jellyfish` | `bellShape / tentacleCount / tentacleLength / ribbonCount` | 伞盖、触手、口腕、透明与发光 |
| `eel` | `bodyLength / bodyWave / finStyle / mouth` | 长体曲线、连续背鳍与蛇形摆动 |
| `crab` | `shellShape / clawSize / legCount / spikes` | 甲壳、双螯、步足与棘刺 |

`sanitize()` 必须先校验 `bodyPlan`，再只接收对应的 `anatomy` 白名单字段，并为缺失值填入该体型的安全默认值。

## 3. 解析与生成链路

文字输入继续经过 `localParse()` 或 Claude，但提示词改为先判断 `bodyPlan`、再填写公共字段和对应 `anatomy`，因此“黑红相间的八爪鱼”会进入 `octopusSvg()` 而不是被鱼形参数近似。

离线解析先匹配物种名和体型关键词，再应用颜色、花纹和部件关键词；无法判断时默认 `fish`，但界面明确显示使用了离线鱼类默认值。

真实物种模板按体型拆分为模板表，现有 10 条鱼全部保留为 `bodyPlan: "fish"`，后续每种新体型至少提供 2 个验收模板。

## 4. 渲染器拆分

```js
const BODY_RENDERERS = {
  fish: fishSvg,
  octopus: octopusSvg,
  jellyfish: jellyfishSvg,
  eel: eelSvg,
  crab: crabSvg,
};

function creatureSvg(gene) {
  return BODY_RENDERERS[gene.bodyPlan](gene);
}
```

每个渲染器只读取公共字段和自己的 `anatomy`，输出统一坐标系与边界盒，使鱼缸运动、选中、碰壁、缩放和导出 SVG 无需理解具体体型。

运动行为通过体型配置调整速度、转向和动画频率，但仍复用同一个实体循环；例如水母缓慢脉冲、鳗鱼连续摆动、螃蟹横向移动。

## 5. 跨体型繁殖

- 后代的结构体型以 50% 概率继承任一亲本，突变时才以小概率出现第三种体型，避免生成没有渲染规则的混合拓扑。
- 公共字段沿用现有离散继承、连续混合、高斯扰动和 HSL 最短弧插值，因此跨体型外观仍可解释。
- `anatomy` 只从与后代 `bodyPlan` 相同的亲本继承；若双方都不同，则使用该体型默认值再施加一次小突变。
- 异体亲本的代表性特征通过公共字段转译，例如水母贡献透明和发光、螃蟹贡献甲壳色和尖刺部件、八爪鱼贡献触须部件。
- `generatedBy` 继续记录来源但不进入 `speciesKey()`，出生方式不能把同一物种拆成多个图鉴条目。

## 6. 种子化程序纹理

`patternSeed` 是纹理的唯一随机源，渲染时使用固定的 uint32 PRNG 生成条纹、斑点、云斑和大理石纹参数，禁止直接调用无种子的 `Math.random()`。

纹理优先使用 SVG `pattern`、`mask`、`clipPath` 和少量确定性噪声路径生成，不引入外部位图，保证 Pages 离线可运行和单鱼 SVG 可导出。

繁殖时 70% 直接继承一方种子、25% 通过双亲种子哈希得到新种子、5% 完全突变；相同双亲的不同后代仍可不同，但每个后代一旦出生就永久稳定。

所有 SVG 定义的 DOM id 必须包含鱼的 `gene.id` 或种子哈希，避免多条鱼同屏时纹理引用互相串色。

## 7. 图鉴与版本迁移

`speciesKey()` v3 必须包含 `bodyPlan`、规范化后的 `anatomy` 离散字段和公共离散性状，但不包含 `id`、谱系、`generatedBy`、运行时状态或原始 `patternSeed`。

`patternSeed` 不直接参与物种去重，否则每个纹理种子都会制造一个新物种；图鉴卡片保存首次发现个体作为该物种的稳定展示样本。

`migrateGeneV2toV3()` 将所有 v2 基因设为 `bodyPlan: "fish"`，把 `shape/tail/fin/mouth` 移入 `anatomy`，将 `accent` 统一转为 HSL，并为纹理、透明度和发光字段补默认值。

存档外层 `saveVersion` 与基因 `schemaVersion` 独立：如果容器结构不变则无需提升 `saveVersion`，加载时逐条迁移基因即可。

## 8. 图像生成模型的边界

如果未来接入 diffusion 或 flow matching，只生成图鉴肖像或装饰纹理并缓存结果，不替代可遗传的 Schema 和确定性 SVG 实体。

模型产物不得参与 `speciesKey()`、碰撞或繁殖计算，生成失败时也不得阻断孵化和养成主循环。

## 9. 实现顺序

1. `feature/schema-v3-body-plan`：新增迁移、判别 Schema、解析和图鉴键，所有 v2 鱼视觉保持不变。
2. `feature/body-renderers`：依次落地 octopus、jellyfish、eel、crab 渲染器与运动配置。
3. `feature/cross-body-breeding`：实现跨体型遗传、特征转译和体型模板。
4. `feature/seeded-patterns`：实现确定性 PRNG、程序纹理和种子遗传。

每一步独立验收并合并，禁止在同一 PR 同时重写存档、成长循环和全部渲染器。

## 10. 验收清单

- [ ] 输入“黑红相间的八爪鱼”生成八足章鱼，刷新和导出后外观不变
- [ ] 五种 `bodyPlan` 各有独立轮廓、动画和至少 2 个真实模板
- [ ] v2 图鉴与 v0.5 存档加载后全部迁移为视觉一致的 fish 类型
- [ ] 鱼 × 水母可以产出带透明或发光特征的后代，谱系与来源记录正确
- [ ] 同一基因在 Chrome 和 Firefox 中产生一致的结构、颜色和纹理
- [ ] 纹理种子不会让图鉴把每个个体误判为新物种
- [ ] 12 个生物与 8 粒饲料同屏时维持流畅，导出的 SVG 不缺纹理定义
- [ ] 未知 `bodyPlan`、错误 `anatomy` 和未来 schemaVersion 均明确报错且不污染存档
