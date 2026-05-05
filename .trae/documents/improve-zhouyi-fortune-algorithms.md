# 改进后端周易算命核心功能算法逻辑 - 实施计划

## 一、摘要

本计划旨在系统性地改进 `zhouyi-fortune-framework` 项目中周易算命核心功能的算法逻辑。改进范围涵盖**周易起卦**、**八字命理分析**、**六爻占卜**三大模块，目标是提升算法准确性、增强分析深度、优化代码质量。所有算法改进将基于权威文献（《周易·系辞传》《三命通会》《渊海子平》等）进行验证，确保结果不是幻觉。

## 二、现状分析

### 2.1 项目架构
- **技术栈**: TypeScript / Node.js / Jest
- **核心依赖**: `lunar-javascript`（农历/八字转换）
- **目录结构**:
  - `/src/core/` — 核心算法（ZhouyiMethod.ts, FortuneTeller.ts）
  - `/src/bazi/` — 八字计算（baZiCalculator.ts, stemsAndBranches.ts, lunarConverter.ts）
  - `/src/liuyao/` — 六爻占卜（liuYaoCalculator.ts）
  - `/src/fate/` — 命理分析（fateAnalyzer.ts）
  - `/src/interpretation/` — 解释生成（Interpreter.ts）
  - `/src/interfaces/` — 类型定义

### 2.2 当前问题识别

#### 周易模块 (ZhouyiMethod.ts)
| 问题 | 严重程度 | 说明 |
|------|---------|------|
| 蓍草起卦未实现 | 高 | `yarrowStalkMethod()` 直接调用 `coinCastMethod()`，没有真正的揲蓍算法 |
| 爻辞数据严重缺失 | 高 | 仅4个卦（乾、坤、泰、否）有爻辞，其余60卦缺失 |
| 卦象关联分析缺失 | 中 | 无本卦、变卦、互卦的综合分析 |
| 硬币起卦概率正确 | 低 | 当前实现概率分布正确（老阳1/8、少阳3/8、少阴3/8、老阴1/8） |

#### 八字模块 (baZiCalculator.ts / fateAnalyzer.ts)
| 问题 | 严重程度 | 说明 |
|------|---------|------|
| 无格局判断 | 高 | 未实现正官格、七杀格等八正格判断 |
| 无神煞系统 | 高 | 天乙贵人、桃花、驿马等常用神煞未实现 |
| 五行强弱计算粗糙 | 中 | 仅简单计数+月令加权，未考虑通根、透干、合会等 |
| 喜用神判断简单 | 中 | 仅基于日主强弱二元判断，未考虑调候、通关等 |
| 十神分析不完整 | 中 | 日干本身未参与十神计算，地支藏干十神权重固定0.5 |
| 大运流年算法基础 | 低 | 基本逻辑正确，但吉凶判断过于简化 |

#### 六爻模块 (liuYaoCalculator.ts)
| 问题 | 严重程度 | 说明 |
|------|---------|------|
| 爻辞数据仅12条 | 高 | 仅存储了乾卦和坤卦的爻辞，其他卦缺失 |
| 手动起卦逻辑有疑 | 中 | `manualCast` 将三次掷币结果平铺为6爻，需验证是否符合传统规则 |
| 时间起卦算法简单 | 低 | 基于年月日时分的简单求和取模，可保留 |

## 三、改进方案

### 3.1 周易起卦模块改进

#### 3.1.1 实现真正的蓍草起卦算法（yarrowStalkMethod）
**文件**: `/workspace/src/core/ZhouyiMethod.ts`

根据《周易·系辞传》"大衍之数五十，其用四十有九"实现四营十八变揲蓍法：

```
算法步骤（每爻需三变）：
1. 准备50根蓍草，取出一根不用（象征太极），剩余49根
2. 第一变（四营）：
   - 分而为二：随机分为左右两堆
   - 挂一：从右堆取1根挂于左手小指间（象征人）
   - 揲四：右手4根一组数左堆，余数夹于无名指与中指间
   - 归奇：左手4根一组数右堆，余数夹于中指与食指间
   - 挂扐数总和 = 挂1 + 左余 + 右余，必为5或9
   - 剩余蓍草 = 49 - 挂扐数，为44或40
3. 第二变：用剩余蓍草重复四营，挂扐数总和为4或8
   - 剩余可能为40、36、32
4. 第三变：再次重复四营，挂扐数总和为4或8
   - 剩余可能为36、32、28、24
5. 定爻：剩余蓍草 ÷ 4
   - 36÷4=9 → 老阳（阳爻，变爻）
   - 32÷4=8 → 少阴（阴爻）
   - 28÷4=7 → 少阳（阳爻）
   - 24÷4=6 → 老阴（阴爻，变爻）
6. 重复以上6次（共18变），从下至上得六爻
```

**概率分布验证**：
- 老阳（9）概率 ≈ 1/8
- 少阳（7）概率 ≈ 3/8
- 少阴（8）概率 ≈ 3/8
- 老阴（6）概率 ≈ 1/8

#### 3.1.2 补全64卦384爻爻辞数据
**文件**: 新建 `/workspace/src/core/hexagramTexts.ts`

将64卦的卦辞和384条爻辞完整录入，数据结构：
```typescript
interface HexagramText {
  name: string;
  number: number;
  guaCi: string;           // 卦辞
  xiangCi: string;         // 象曰
  tuanCi: string;          // 彖曰
  yaoCi: string[];         // 六爻爻辞（初九到上九/上六）
  yongJiu?: string;        // 用九（仅乾卦）
  yongLiu?: string;        // 用六（仅坤卦）
}
```

数据来源：基于《周易》原文（已通过网络检索获取完整内容）。

#### 3.1.3 增加卦象关联分析
**文件**: `/workspace/src/core/ZhouyiMethod.ts`

增加以下分析维度：
- **本卦**：原始卦象
- **变卦（之卦）**：动爻变化后的卦象
- **互卦**：取本卦234爻为下互卦，345爻为上互卦
- 生成综合分析文本，包含本卦、变卦、互卦的卦辞关联解读

### 3.2 八字命理分析模块改进

#### 3.2.1 实现格局判断系统
**文件**: 新建 `/workspace/src/bazi/patternAnalyzer.ts`

实现八正格判断逻辑：
```
格局判定步骤：
1. 定日主强弱（已有基础，需增强）
2. 看月令本气与透干：
   - 月令本气透出天干 → 以该十神定格局
   - 如月令寅（甲木），天干见甲 → 建禄格
   - 如月令寅（甲木），天干见丙 → 食神格
   - 如月令寅（甲木），天干见戊 → 偏财格
3. 八正格：正官格、七杀格、正印格、偏印格、正财格、偏财格、食神格、伤官格
4. 特殊格局（可选）：从格、专旺格、化气格
```

#### 3.2.2 实现神煞系统
**文件**: 新建 `/workspace/src/bazi/shenShaAnalyzer.ts`

实现以下常用神煞：
| 神煞 | 查法口诀 | 实现复杂度 |
|------|---------|-----------|
| 天乙贵人 | 甲戊庚牛羊，乙己鼠猴乡，丙丁猪鸡位，壬癸兔蛇藏，六辛逢马虎 | 低 |
| 文昌 | 甲在巳、乙在午、丙戊在申、丁己在酉、庚在亥、辛在子、壬在寅、癸在卯 | 低 |
| 桃花（咸池） | 申子辰见酉，寅午戌见卯，亥卯未见子，巳酉丑见午 | 低 |
| 驿马 | 申子辰马在寅，寅午戌马在申，亥卯未马在巳，巳酉丑马在亥 | 低 |
| 羊刃 | 甲刃卯、乙刃辰、丙戊刃午、丁己刃未、庚刃酉、辛刃戌、壬刃子、癸刃丑 | 低 |
| 太极贵人 | 甲乙生人子午中，丙丁鸡兔定亨通... | 低 |

#### 3.2.3 增强五行强弱计算
**文件**: `/workspace/src/fate/fateAnalyzer.ts`

当前 `calculateWuxingStrength` 仅做简单加权，需增强：
- 考虑天干通根（天干在地支中有同类五行）
- 考虑地支合会（三合局、三会局）对五行的增强
- 考虑地支藏干的实际力量分配（本气、中气、余气）
- 引入更精细的月令当令权重

#### 3.2.4 纳音五行深度分析
**文件**: `/workspace/src/bazi/baZiCalculator.ts` 及 `/workspace/src/fate/fateAnalyzer.ts`

当前已有基础纳音查询（`getNayinInfo`），需增强：
- 年柱纳音作为"本命纳音"在分析中的权重
- 纳音五行与正五行的生克关系分析
- 纳音在合婚、择日等场景的应用（接口预留）

### 3.3 六爻占卜模块改进

#### 3.3.1 补全64卦爻辞
**文件**: `/workspace/src/liuyao/liuYaoCalculator.ts`

复用周易模块的 `hexagramTexts.ts` 数据，或建立共享的爻辞数据源。

#### 3.3.2 修正手动起卦逻辑
验证当前 `manualCast` 方法的输入处理是否符合传统六爻规则（每次掷币结果应为3枚硬币的组合，而非直接传入6个结果）。

### 3.4 接口扩展

**文件**: `/workspace/src/interfaces/interfaces.ts`

新增/扩展接口：
```typescript
// 新增：格局分析
interface PatternAnalysis {
  pattern: string;           // 格局名称
  patternType: 'normal' | 'special';  // 正格/变格
  isFormed: boolean;         // 是否成格
  analysis: string;
}

// 新增：神煞分析
interface ShenShaAnalysis {
  shenShaList: ShenShaInfo[];
  analysis: string;
}

interface ShenShaInfo {
  name: string;
  type: '吉神' | '凶煞';
  position: string;          // 所在柱
  description: string;
}

// 扩展：BaZiInfo 增加格局和神煞
interface BaZiInfo {
  // ... 现有字段
  pattern?: PatternAnalysis;
  shenSha?: ShenShaAnalysis;
}
```

## 四、实施步骤

### 步骤1：数据层建设
1. 创建 `/workspace/src/core/hexagramTexts.ts` — 64卦完整爻辞数据
2. 创建 `/workspace/src/bazi/shenShaData.ts` — 神煞查法规则数据

### 步骤2：周易模块改进
1. 修改 `/workspace/src/core/ZhouyiMethod.ts`:
   - 实现 `yarrowStalkMethod()` 真正的揲蓍算法
   - 引入 `hexagramTexts.ts` 替换硬编码爻辞
   - 增加互卦计算和本卦/变卦/互卦综合分析

### 步骤3：八字模块改进
1. 创建 `/workspace/src/bazi/patternAnalyzer.ts` — 格局判断
2. 创建 `/workspace/src/bazi/shenShaAnalyzer.ts` — 神煞计算
3. 修改 `/workspace/src/fate/fateAnalyzer.ts`:
   - 增强 `calculateWuxingStrength` 算法
   - 在 `analyzeFate` 中整合格局和神煞分析
   - 增强纳音分析

### 步骤4：六爻模块改进
1. 修改 `/workspace/src/liuyao/liuYaoCalculator.ts`:
   - 引入共享爻辞数据
   - 修正手动起卦输入验证

### 步骤5：接口更新
1. 修改 `/workspace/src/interfaces/interfaces.ts` — 扩展类型定义

### 步骤6：测试与验证
1. 编写单元测试验证揲蓍算法概率分布
2. 编写测试验证格局判断典型案例
3. 编写测试验证神煞查法
4. 运行 `npm run test` 确保全部通过
5. 运行 `npm run build` 确保无编译错误

## 五、验证标准

1. **蓍草起卦**: 运行10000次，统计各爻类型出现频率，应与理论概率（老阳1/8、少阳3/8、少阴3/8、老阴1/8）误差<2%
2. **爻辞完整性**: 64卦 × 6爻 = 384条爻辞全部可查询
3. **格局判断**: 至少验证10个经典八字案例的格局判断正确性
4. **神煞查法**: 天乙贵人、桃花、驿马、文昌、羊刃等核心神煞查法与手工计算一致
5. **编译通过**: `npm run build` 无错误
6. **测试通过**: `npm run test` 全部通过

## 六、假设与决策

### 已做决策
1. **揲蓍法实现方式**: 采用"过揲法"（计算剩余蓍草数÷4），而非"挂扐法"，因为过揲法更易于程序化验证
2. **爻辞数据来源**: 采用传统《周易》原文，不含现代解读
3. **格局判断范围**: 优先实现八正格（正官、七杀、正印、偏印、正财、偏财、食神、伤官），特殊格局（从格、专旺格）作为二期
4. **神煞范围**: 优先实现最常用的6种神煞（天乙贵人、文昌、桃花、驿马、羊刃、太极贵人）
5. **五行强弱增强**: 在现有算法基础上增加通根、合会因素，不推翻现有框架

### 技术假设
1. `lunar-javascript` 库的八字计算结果是准确的，不做替换
2. 项目使用 CommonJS 模块系统，新增文件遵循此规范
3. 不引入新的运行时依赖（仅使用现有 `lunar-javascript`）

## 七、文件变更清单

| 操作 | 文件路径 | 说明 |
|------|---------|------|
| 新建 | `/workspace/src/core/hexagramTexts.ts` | 64卦完整爻辞数据 |
| 修改 | `/workspace/src/core/ZhouyiMethod.ts` | 实现揲蓍法、互卦分析、引入完整爻辞 |
| 新建 | `/workspace/src/bazi/patternAnalyzer.ts` | 八字格局判断 |
| 新建 | `/workspace/src/bazi/shenShaAnalyzer.ts` | 神煞计算 |
| 新建 | `/workspace/src/bazi/shenShaData.ts` | 神煞规则数据 |
| 修改 | `/workspace/src/fate/fateAnalyzer.ts` | 增强五行分析、整合格局神煞、增强纳音 |
| 修改 | `/workspace/src/liuyao/liuYaoCalculator.ts` | 引入完整爻辞、修正起卦逻辑 |
| 修改 | `/workspace/src/interfaces/interfaces.ts` | 扩展类型定义 |
| 新建/修改 | `/workspace/src/**/*.test.ts` | 新增单元测试 |

## 八、风险与缓解

| 风险 | 可能性 | 影响 | 缓解措施 |
|------|--------|------|---------|
| 爻辞数据量大导致编译/加载慢 | 低 | 中 | 使用按需加载或分模块导出 |
| 格局判断规则存在流派差异 | 中 | 中 | 以《三命通会》《渊海子平》为基准，文档注明依据 |
| 神煞查法版本不一 | 中 | 低 | 以日干查法为主（主流），文档说明查法依据 |
| 增强算法后向后兼容性问题 | 低 | 高 | 保持现有接口不变，新增字段为可选 |
