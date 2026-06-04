# 007 - Skill 机制实现

## 定位

Pi 实现的是 [Agent Skills 标准](https://agentskills.io/specification)。**Skill 是按需加载的能力包**：一个含 `SKILL.md` 的目录，内含 frontmatter + 指令，外加可选的脚本 / 参考文档 / 资源。核心理念是**渐进式披露（progressive disclosure）**——只把"有哪些 skill"放进系统提示词，完整指令等任务匹配时才加载。

## 关键文件

| 文件 | 说明 |
|------|------|
| `packages/agent/src/harness/skills.ts` | 底层、可移植的加载器（通过抽象的 `ExecutionEnv` 做文件 IO） |
| `packages/coding-agent/src/core/skills.ts` | coding-agent 应用层加载器 |
| `packages/coding-agent/src/core/tools/read.ts` | `read` 工具；对 `SKILL.md` 有特判（仅 UI 标签，**不剥 frontmatter**） |
| `packages/coding-agent/src/core/agent-session.ts` | `_expandSkillCommand()` 展开 `/skill:name` 命令 |
| `packages/coding-agent/docs/skills.md` | 官方使用文档 |

---

## 工作流程

1. **启动扫描**：`loadSkills()`（`skills.ts:387`）从多处扫描——全局 `~/.pi/agent/skills/`、`~/.agents/skills/`；项目 `.pi/skills/`、向上查找的 `.agents/skills/`；package.json 的 `pi.skills`；settings 的 `skills` 数组；CLI 的 `--skill`。
2. **发现规则**（`loadSkillsFromDirInternal`，`skills.ts:173`）：目录里有 `SKILL.md` 就当作 skill 根、不再递归；否则递归子目录找 `SKILL.md`；某些位置允许根 `.md` 直接作为 skill。遵守 `.gitignore`/`.ignore`/`.fdignore`，跳过 `node_modules`，用 `canonicalizePath` 去重符号链接、同名 skill 碰撞保留先发现者。
3. **校验**（`validateName`/`validateDescription`，`skills.ts:92`）：name ≤64 字符、仅小写字母/数字/连字符、不能首尾或连续连字符；description 必填且 ≤1024 字符。**大多数违规只 warning 仍加载，唯独 description 缺失才拒绝。**
4. **注入系统提示词**：`formatSkillsForPrompt()`（`skills.ts:335`）只把每个 skill 的 `name` / `description` / `location` 以 XML（`<available_skills>`）写入提示词。`disable-model-invocation: true` 的 skill 从提示词隐藏。
5. **按需加载正文**：任务匹配时用 `read` 工具读 `SKILL.md`。

> Pi 故意放宽了标准里"name 必须等于父目录名"这条要求（`skills.ts:296` 用 frontmatter name，缺失才回退父目录名），方便多 harness 共享 skill 目录。

---

## 两条加载路径（重要）

| 路径 | 是否剥 frontmatter | frontmatter 是否重复占 token |
|------|------|------|
| 模型用 `read` 工具 | ❌ 返回整文件 | 重复一次（name/description 已在系统提示词里） |
| `/skill:name` 斜杠命令 | ✅ `stripFrontmatter`（`agent-session.ts:1160`） | 不重复 |

- **正文（instructions/body）永不重复**：系统提示词里只有 name/description/location，正文从不常驻，`read` 一次只占一次 token——这正是渐进式披露的目的。
- **frontmatter 只在 `read` 路径下重复一次**，且只是几十~两百 token，相对正文可忽略。
- `/skill:name` 走 `_expandSkillCommand()`，剥掉 frontmatter 再注入，连那点重复都没有；命令后的参数以 `User: <args>` 追加。

---

## 是否有专门的 "skill 工具"？

**没有。** 这是 pi 的设计取舍：
- 加载完整内容靠**通用 `read` 工具** + **`/skill:name` 命令展开**，没有独立的 `Skill` 工具。
- 对比 Claude Code（**有**独立 `Skill` 工具）；pi 刻意贴合 Agent Skills 标准的"纯文件 + 渐进披露"理念。
- 但**单个 skill 内部可携带脚本/资源**（`scripts/`、`references/`、`assets/`），SKILL.md 用相对路径引用，模型再通过 `bash` 等通用工具执行。frontmatter 还有实验性的 `allowed-tools` 字段，可预批准一组工具。

---

## 实践建议

- `SKILL.md` 写短：`read` 默认按行数/字节截断（`read.ts` 的 `DEFAULT_MAX_LINES`/`DEFAULT_MAX_BYTES`），过长会被截断、需 offset 续读。把细节拆到 `references/` 里按需再读。
- description 要具体——它决定模型何时加载该 skill。
- 在意 frontmatter 那点重复开销，可引导走 `/skill:name`；但相比把所有正文常驻提示词，渐进披露已省了绝大头。
