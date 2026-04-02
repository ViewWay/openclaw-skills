# Claude Code 集成指南

让 Claude Code / Cursor / Copilot 等 AI 编程工具能自动调用这些知识库。

## 方法一：直接放入项目（推荐）

```bash
# 1. 克隆
git clone https://github.com/ViewWay/openclaw-skills.git
cd openclaw-skills

# 2. 复制到你的项目中
cp -r . /path/to/your-project/.claw-skills/

# 3. 在项目根目录创建或编辑 CLAUDE.md
echo '
## 知识库参考
当用户提问涉及通信工程、嵌入式开发、计算机科学等专业问题时，请参考 `.claw-skills/` 目录下的对应知识文件：
- signal-processing → 信号处理
- communication-protocols → 通信协议
- embedded-arm → 嵌入式 ARM 开发
- rf-engineering → 射频工程
- power-systems → 电力系统
- embedded-os → 嵌入式操作系统
- sensors-actuators → 传感器与执行器
- circuit-design → 电路设计
- data-structures-algorithms → 数据结构与算法
- operating-systems → 操作系统
- computer-networks → 计算机网络
- computer-architecture → 计算机组成
- databases → 数据库
- distributed-systems → 分布式系统
- software-engineering → 软件工程
- cryptography → 密码学
- artificial-intelligence → 人工智能
- compiler-design → 编译原理
- formal-languages → 形式语言与自动机
- math / physics / chemistry / biology → 理科基础
请先阅读对应 SKILL.md，再根据知识库内容回答。
' >> /path/to/your-project/CLAUDE.md
```

Claude Code 启动时会自动读取 `CLAUDE.md`，遇到专业问题时会自动查阅 `.claw-skills/` 下的文件。

## 方法二：全局配置（所有项目生效）

```bash
# Claude Code 全局配置
mkdir -p ~/.claude
cat > ~/.claude/CLAUDE.md << 'EOF'
## 知识库参考
当用户提问涉及以下领域时，请先阅读 ~/claw-skills/ 下的对应文件：
- 通信工程/信号处理/协议 → ~/claw-skills/signal-processing/SKILL.md, ~/claw-skills/communication-protocols/SKILL.md
- 嵌入式/ARM/RTOS → ~/claw-skills/embedded-arm/SKILL.md, ~/claw-skills/embedded-os/SKILL.md
- 射频/电力 → ~/claw-skills/rf-engineering/SKILL.md, ~/claw-skills/power-systems/SKILL.md
- 传感器/电路 → ~/claw-skills/sensors-actuators/SKILL.md, ~/claw-skills/circuit-design/SKILL.md
- 计算机科学 → ~/claw-skills/computer-science-knowledge-base/SKILL.md（索引文件）
- 数学/物理/化学/生物 → ~/claw-skills/math/SKILL.md 等
请先读取对应文件再回答，确保专业知识准确。
EOF

# 克隆知识库到家目录
git clone https://github.com/ViewWay/openclaw-skills.git ~/claw-skills
```

## 方法三：按需引用（轻量级）

不复制文件，直接在对话中告诉 Claude：

```
请参考 https://github.com/ViewWay/openclaw-skills 仓库中
skills/embedded-arm/SKILL.md 的内容，帮我写一个 STM32 的 UART 驱动。
```

Claude Code 支持直接读取 GitHub 仓库文件。

## 方法四：Cursor 集成

```bash
# 1. 复制到项目
cp -r openclaw-skills/ /path/to/project/.claw-skills/

# 2. 在 .cursorrules 中添加：
cat > /path/to/project/.cursorrules << 'EOF'
# Knowledge Base
When answering questions about communication engineering, embedded systems,
computer science, or related fields, read the relevant SKILL.md files
under .claw-skills/ directory first.

## Available knowledge files:
- .claw-skills/signal-processing/SKILL.md
- .claw-skills/communication-protocols/SKILL.md
- .claw-skills/embedded-arm/SKILL.md
- .claw-skills/embedded-os/SKILL.md
- .claw-skills/rf-engineering/SKILL.md
- .claw-skills/power-systems/SKILL.md
- .claw-skills/sensors-actuators/SKILL.md
- .claw-skills/circuit-design/SKILL.md
- .claw-skills/data-structures-algorithms/SKILL.md
- .claw-skills/operating-systems/SKILL.md
- .claw-skills/computer-networks/SKILL.md
- .claw-skills/computer-architecture/SKILL.md
- .claw-skills/databases/SKILL.md
- .claw-skills/distributed-systems/SKILL.md
- .claw-skills/software-engineering/SKILL.md
- .claw-skills/cryptography/SKILL.md
- .claw-skills/artificial-intelligence/SKILL.md
- .claw-skills/compiler-design/SKILL.md
- .claw-skills/formal-languages/SKILL.md
EOF
```

## 方法五：Copilot Chat 集成

在 `.github/copilot-instructions.md` 中添加：

```markdown
## Knowledge Base
Reference .claw-skills/ directory for professional knowledge about
communication engineering, embedded development, and computer science.
Read the relevant SKILL.md before answering domain-specific questions.
```

## 验证

配置完成后，在 Claude Code 中测试：

```
> 帮我设计一个 LoRaWAN 节点的通信方案
```

Claude 会自动读取 `communication-protocols/SKILL.md` 中的 LoRaWAN 章节，给出专业回答。

## 更新知识库

```bash
cd ~/claw-skills   # 或你的项目/.claw-skills/
git pull
```
