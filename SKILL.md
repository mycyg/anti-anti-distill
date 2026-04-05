---
name: anti-anti-distill
description: "Detect anti-distillation cleaning in submitted Skill files. Score information density, flag washed content, generate audit reports. | 反反蒸馏：检测 Skill 文件是否被清洗过，评分信息密度，生成审计报告。"
argument-hint: "[file-path-or-slug]"
version: "1.0.0"
user-invocable: true
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---

> **Language / 语言**: This skill supports both English and Chinese. Detect the user's language from their first message and respond in the same language throughout.
>
> 本 Skill 支持中英文。根据用户第一条消息的语言，全程使用同一语言回复。

# 反反蒸馏 Skill（Claude Code 版）

## 触发条件

当用户说以下任意内容时启动：
- `/anti-anti-distill`
- "帮我检测这个 skill"
- "反反蒸馏"
- "审计一下这份 skill"
- "检测清洗"
- "check skill"
- "audit skill"

---

## 工具使用规则

| 任务 | 使用工具 |
|------|---------|
| 读取用户提供的 Skill 文件 | `Read` 工具 |
| 读取 PDF 文档 | `Read` 工具（原生支持 PDF） |
| 读取图片截图 | `Read` 工具（原生支持图片） |
| 搜索文件 | `Glob` / `Grep` 工具 |
| 统计文件信息 | `Bash` → `wc`、`grep -c` 等 |
| 写入报告 | `Write` 工具 |
| 创建目录 | `Bash` → `mkdir -p` |

---

## 主流程

### Step 1：接收输入

接收用户提供的文件，支持以下方式：

**方式 A：指定文件路径**
用户直接给出文件路径，用 `Read` 读取。

**方式 B：指定 colleague-skill 目录**
用户给出 `colleagues/{slug}/` 路径，自动读取其中的：
- `work.md`
- `persona.md`
- `meta.json`
- 或 `SKILL.md`（如果是合并版）

**方式 C：粘贴内容**
用户直接粘贴文档内容。

**方式 D：搜索本地文件**
用户说"帮我找一下"，用 `Glob` 搜索 `**/SKILL.md`、`**/work.md`、`**/persona.md` 等文件。

**方式 E：批量检测**
用户提供一个目录路径，用 `Glob` 搜索其中所有 skill 文件，批量检测。

读取完文件后，自动识别格式：
- **colleague-skill 格式**：检测到 `## Layer 0` 或 `PART A` / `PART B` 或同时存在 `work.md` + `persona.md`
- **通用文档格式**：其他任何 Markdown / TXT / PDF

告知用户：
```
已读取文件：{文件列表}
检测到格式：{colleague-skill 格式 / 通用文档格式}
总字数：约 {N} 字
角色信息：{从 meta.json 或文档中提取的职级/角色信息}

开始检测...
```

---

### Step 2：信号检测

参考 `${CLAUDE_SKILL_DIR}/prompts/detector.md` 中的六大特征信号，逐一扫描。

**如果是 colleague-skill 格式，按文件分区检测：**

#### Work Skill 检测

使用 `Bash` 辅助统计：
```bash
# S1: 统计经验知识库章节的数字密度
grep -oE '[0-9]+([.][0-9]+)?' work.md | wc -l

# S2: 统计废话关键词密度（中文40+模式 + 英文10模式）
grep -cE '遵循.*规范|注意.*合理|综合.*考虑|按.*标准流程|合理.*设置|注意.*安全|关注.*性能|根据.*实际|确保.*可靠|注意.*健壮|be careful|make sure|best practice|appropriate|reasonable|properly|correctly' work.md

# S3: 统计因果连词（辅助信号，主要依赖语义分析）
grep -cE '因为|导致|否则|不然|会出|防止|原因是|为了|目的是|源于|结果|后果' work.md

# S7: 检测 MASK 标签和命名实体缺失
grep -cE 'MASK|【公司名称】|【项目名】|【技术栈】|\[Company\]|\[Project\]|\[Tech\]' work.md

# S8: 计算章节结构均匀度（CV）
# 统计每章字数，计算变异系数
awk '/^## /{if(NR>1)print count;count=0;next}{count+=NF}END{print count}' work.md

# S9: 检测"风险警告"vs"事故描述"
grep -cE '风险|注意|避免|防止|警告' work.md  # 警告型词汇
grep -cE '曾经|那次|当时|结果|问题|故障|事故|踩坑' work.md  # 叙事型词汇
```

对 Work Skill 的每个章节，检查以下**九大信号**：

| 信号 | 检测目标 | 检测方法 |
|------|---------|---------|
| **S1: 数值真空** | 经验知识库、技术规范 | 统计具体数字出现密度 |
| **S2: 万能废话密度** | 所有章节 | 统计废话关键词占比 + 语义结构检测 |
| **S3: 因果质量不足** | 经验知识库、CR 重点 | 信息内容分析（4级评分） |
| **S4: 流程坍缩** | 工作流程 | 统计流程步骤数 |
| **S5: 信息密度梯度** | 跨章节对比 | 各章节具体事实密度对比 |
| **S7: 命名实体真空** | 经验知识库、项目经历 | 检测 MASK 标签、缺失的公司/产品/技术名 |
| **S8: 结构完美悖论** | 整体结构 | CV 检测（章节均匀度异常） |
| **S9: 事故叙事缺失** | 踩坑经验 | 检测"风险警告"替代"事故描述" |

#### Persona 检测（S6）

参考 `${CLAUDE_SKILL_DIR}/prompts/persona_checker.md` 中的规则，逐 Layer 检测：

| Layer | 检测重点 |
|-------|---------|
| Layer 0 | 正面描述占比、棱角规则缺失 |
| Layer 1 | 主观评价缺失 |
| Layer 2 | 口头禅通用化、对话示例模板化 |
| Layer 3 | 优先级模糊化、拒绝策略漂白 |
| Layer 4 | 负面协作行为缺失 |
| Layer 5 | 个人偏好/雷区缺失 |

**如果是通用文档格式：**

参考 `${CLAUDE_SKILL_DIR}/prompts/detector.md` 中的通用规则进行检测，重点检查数值真空、废话密度、因果断裂三个信号。

---

### Step 2.5：版本差异比对（可选）

如果用户同时提供了原始版本和疑似清洗版本，执行差异比对分析：

**文件获取方式：**
- 用户明确指定两个文件路径
- 自动搜索备份文件：`{filename}_private_backup.md`、`*_backup*.md`、`*_original*.md`
- 检测到 `anti-distill-output/` 目录结构时，读取 before/after 对照

**比对方法：**

1. **章节对齐**
   - 将原文和清洗版本按相同章节对齐
   - 对无法对齐的章节标记为"新增/删除"

2. **知识点计数**
   ```bash
   # 统计每个章节的具体知识点数量（数字、专有名词、特定配置）
   grep -oE '[0-9]+([.][0-9]+)?|mysql|redis|nginx|timeout|max_connections' section.md | wc -l
   ```

3. **计算知识保留率**
   ```
   保留率 = (清洗后知识点数 / 原文知识点数) × 100%
   
   保留率区间解读：
   - 90-100%：轻微清洗或未清洗
   - 70-90%：轻度清洗
   - 40-70%：中度清洗（anti-distill medium 级别）
   - <40%：重度清洗（anti-distill heavy 级别）
   ```

4. **生成差异热力图**
   - 绿色：高保留率段落（>80%）
   - 黄色：中等保留率（50-80%）
   - 红色：低保留率段落（<50%）

**输出：**
在审计报告中添加"版本对比"章节，列出：
- 整体知识保留率
- 各章节保留率对比表
- 关键信息丢失清单

---

### Step 3：信息密度评分

参考 `${CLAUDE_SKILL_DIR}/prompts/scorer.md` 中的七维评分体系，计算各维度得分。

对每个维度：
1. 根据评分标准逐项检查
2. 引用原文具体片段作为评分依据
3. 给出 0-10 分的评分

计算加权总分和综合判定。

**加权总分计算：**
```
总分 = (具体事实密度 × 3.5 + 因果完整性 × 2.5 + 流程颗粒度 × 2
      + Persona鲜活度 × 3.0 + 上下文丰富度 × 2 + 角色自洽匹配度 × 1.0
      + 术语语境化 × 0.5) / 14.5
```

**评分阈值（v2.0 更新）：**

| 总分 | 判定 | 建议动作 |
|------|------|---------|
| 6.5+ | 正常 | 通过 |
| 4.5-6.4 | 轻度可疑 | 标注关注，下一轮复核 |
| 2.5-4.4 | 中度可疑 | 人工复核 |
| < 2.5 | 高度可疑 | 退回补充或启动调查 |

---

### Step 4：生成审计报告

综合 Step 2 的信号检测结果和 Step 3 的评分结果，生成完整审计报告。

**报告格式：**

```markdown
# 反蒸馏检测审计报告

> 检测时间：{timestamp}
> 检测对象：{文件名}
> 格式：{colleague-skill / 通用文档}
> 角色信息：{职级/角色}

---

## 一、综合判定

| 项目 | 结果 |
|------|------|
| 可疑度 | {高度/中度/轻度/正常} |
| 置信度 | {高/中/低} |
| 加权评分 | {X}/10 |
| 建议动作 | {通过/人工复核/退回补充} |

---

## 二、信号检测结果

| 信号 | 状态 | 详情 |
|------|------|------|
| S1: 数值真空 | {命中/未命中} | {具体描述} |
| S2: 废话密度 | {命中/未命中} | {X%，阈值 X%，语义检测} |
| S3: 因果质量 | {命中/未命中} | {信息内容等级 X/3} |
| S4: 流程坍缩 | {命中/未命中} | {平均步骤数 X} |
| S5: 密度梯度 | {命中/未命中} | {梯度方向描述} |
| S6: Persona 模板化 | {命中/未命中} | {清洗指数 X.XX} |
| S7: 命名实体真空 | {命中/未命中} | {MASK 标签数 X} |
| S8: 结构完美悖论 | {命中/未命中} | {CV 系数 X.XX} |
| S9: 事故叙事缺失 | {命中/未命中} | {警告/事故比例 X} |

---

## 三、评分明细

| 维度 | 得分 | 权重 | 加权得分 | 依据 |
|------|------|------|---------|------|
| 具体事实密度 | X/10 | ×3.5 | X | {引用原文片段} |
| 因果完整性 | X/10 | ×2.5 | X | {引用原文片段} |
| 流程颗粒度 | X/10 | ×2 | X | {引用原文片段} |
| Persona 鲜活度 | X/10 | ×3.0 | X | {引用原文片段} |
| 上下文丰富度 | X/10 | ×2 | X | {引用原文片段} |
| 角色自洽匹配度 | X/10 | ×1.0 | X | {从文档内容推断职级} |
| 术语语境化 | X/10 | ×0.5 | X | {术语是否出现在使用语境中} |
| **加权总分** | | | **X/10** | (满分14.5) |

---

## 四、可疑细节

{逐条列出可疑段落，包含：
- 原文引用
- 可疑原因
- "正常写法应该是什么样的"参考示例}

---

## 五、批量检测结果

{如果是批量检测，列出每个文件/同事的检测结果表格}

| 同事 | 角色 | 可疑度 | 总分 | 建议动作 |
|------|------|--------|------|---------|
| {name} | {role} | {高度/中度/轻度/正常} | X/10 | {动作} |

---

## 六、建议

{根据检测结果给出具体建议，例如：
- "XX 的经验知识库章节信息密度极低，建议补充具体踩坑经验"
- "XX 的 Persona 过于模板化，建议补充个人化的对话示例和行为特征"
- "整体检测未发现明显清洗痕迹，可通过"}
```

用 `Write` 工具将报告写入 `{output_dir}/audit_report_{timestamp}.md`。

告知用户：
```
审计报告已生成：{report_path}

综合判定：{可疑度}
加权评分：{X}/10
建议动作：{通过/人工复核/退回补充}

详细结果见报告。
```

---

### Step 5：交互式复核

报告生成后，进入交互模式。用户可以：

- "详细看一下 XX 的经验知识库" → 展示该章节的逐条分析
- "对比一下正常版本应该长什么样" → 参考反蒸馏工具的清洗逻辑，逆向展示"原始版本可能包含什么"
- "所有可疑段落列出来" → 仅展示标记为可疑的内容
- "只看高风险的" → 仅展示高度可疑的段落
- "下一个" → 如果是批量检测，切换到下一个文件

---

当用户提供目录或说"检测所有"时，进入批量模式：

1. 用 `Glob` 搜索目录下所有 skill 文件
2. **搜索私有备份文件**（P8 取证搜索）：
   ```bash
   # 搜索常见备份文件命名模式
   find {dir} -type f \( \
     -name "*_private_backup*" -o \
     -name "*_backup*.md" -o \
     -name "*.original.md" -o \
     -name "*_cleaned.md" -o \
     -name "*_raw.md" -o \
     -name "*before*.md" -o \
     -name "*after*.md" \) 2>/dev/null
   
   # 搜索备份目录
   find {dir} -type d \( \
     -name "*_cleaned" -o \
     -name "*backup*" -o \
     -name "*original*" -o \
     -name "anti-distill-output" \) 2>/dev/null
   
   # 检测隐藏备份（文件名中包含日期时间戳）
   find {dir} -name "*20[0-9][0-9]*[0-9][0-9]*[0-9][0-9]*.md" 2>/dev/null
   ```
3. 逐个执行 Step 2-4 的检测
4. **执行跨文件分析**：
   - 计算同批次文件评分标准差，识别异常集中的"安全分数"
   - 检测结构相似度（多份文件章节数、字数分布过于相似）
   - 标记评分聚类异常（如多份文件集中在 6.0-6.5 分区间）
5. 汇总到一份报告中
6. 按可疑度排序，高风险排在前面
7. 在报告开头提供汇总表格

---

## 边界情况处理

### 文件太短（< 300 字）
提醒用户："文件内容过少，无法进行有效检测。建议提供完整 Skill 文件。"

### 仅提供 Persona 或仅提供 Work Skill
告知用户："仅检测到部分文件。建议同时提供 work.md 和 persona.md 以获得完整检测。对已有部分执行检测。"

### 无法确认角色信息
提醒用户："未能自动识别角色信息，请补充职级和岗位。这将影响'角色匹配度'维度的评分准确性。"

### 文件已经是清洗版本 + 原始版本对比
如果用户同时提供了两份文件，执行对比分析，直接识别清洗痕迹。

---

---

# English Version

# Anti-Anti-Distill Skill (Claude Code Edition)

## Trigger Conditions

Activate when the user says:
- `/anti-anti-distill`
- "Check skill"
- "Audit skill"
- "Detect cleaning"
- "Help me check this skill"

---

## Main Flow

### Step 1: Receive Input

Accept files from the user:

- **Option A**: File path → `Read` the file
- **Option B**: colleague-skill directory → Read `work.md`, `persona.md`, `meta.json`
- **Option C**: Pasted content → Use directly
- **Option D**: Search → `Glob` for skill files
- **Option E**: Directory path → Batch detect all skill files

Auto-detect format:
- **colleague-skill format**: Contains `## Layer 0` or `PART A` / `PART B`
- **General document format**: Any other Markdown / TXT / PDF

### Step 2: Signal Detection

Refer to `${CLAUDE_SKILL_DIR}/prompts/detector.md` for the six characteristic signals of anti-distillation cleaning. Scan each section for:
- Numerical vacuum (missing specific numbers/thresholds)
- High generic statement density ("correct but useless" content)
- Broken causation chains (rules without reasons)
- Flow collapse (multi-step processes compressed to one sentence)
- Information density gradient anomaly
- Persona template-ization

### Step 3: Information Density Scoring

Refer to `${CLAUDE_SKILL_DIR}/prompts/scorer.md` for the seven-dimension scoring system. Calculate weighted total score.

### Step 4: Generate Audit Report

Combine signal detection results and scoring results into a complete audit report. Write to `{output_dir}/audit_report_{timestamp}.md`.

### Step 5: Interactive Review

Enter interactive mode for per-section detailed analysis, comparison with expected content, and filtering.

---

## Batch Detection Mode

When user provides a directory or says "check all":
1. `Glob` for all skill files
2. Execute detection on each
3. Aggregate into one report, sorted by suspicion level
