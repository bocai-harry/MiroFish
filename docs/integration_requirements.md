# MiroFish 本地部署集成需求文档

## 版本信息
- **文档版本**: v1.0
- **创建日期**: 2026-03-11
- **适用项目**: MiroFish (https://github.com/666ghj/MiroFish)
- **项目版本**: v0.1.2

---

## 1. 项目概述

### 1.1 什么是MiroFish

MiroFish是一款基于多智能体技术的新一代AI预测引擎，由盛大集团战略支持和孵化。其核心能力包括：

- **群体智能仿真**: 通过多Agent在虚拟社交平台（Twitter/Reddit）上的互动，模拟现实世界的舆论演化
- **知识图谱驱动**: 使用Zep Cloud构建知识图谱，提取实体和关系作为Agent的基础
- **LLM智能生成**: 利用大语言模型自动生成Agent人设、模拟配置和预测报告
- **双平台并行**: 同时在Twitter和Reddit两个虚拟平台运行模拟，观察不同平台的行为差异

### 1.2 核心工作流程

```
Step 1: 图谱构建
   ↓ 上传文档 + 描述模拟需求
   ↓ LLM生成本体定义（10个实体类型 + 6-10个关系类型）
   ↓ Zep Cloud构建知识图谱（最多2000个实体节点）

Step 2: 环境搭建
   ↓ 从图谱读取实体
   ↓ 为每个实体生成OASIS Agent Profile（2000字详细人设）
   ↓ LLM智能生成模拟配置（时间、事件、Agent行为参数）

Step 3: 开始模拟
   ↓ OASIS引擎启动Twitter + Reddit双平台模拟
   ↓ Agent根据人设自主发帖、评论、互动
   ↓ 支持动态图谱记忆更新

Step 4: 报告生成
   ↓ ReportAgent与模拟环境深度交互
   ↓ 自动采访关键Agent获取观点
   ↓ 生成预测报告

Step 5: 深度互动
   ↓ 与任意Agent对话
   ↓ 动态注入变量观察反应
```

---

## 2. 数据来源与输入规范

### 2.1 支持的输入方式

| 输入类型 | 是否必填 | 说明 |
|---------|---------|------|
| **simulation_requirement** | ✅ 必填 | 自然语言描述模拟需求（如"模拟武汉大学舆情事件的演化"） |
| **files** | ❌ 可选 | 支持PDF、Markdown、TXT格式，可多文件上传 |
| **additional_context** | ❌ 可选 | 额外说明和背景信息 |

### 2.2 文件处理规范

**支持的文件格式**:
- PDF (通过PyMuPDF提取文本)
- Markdown/MD (自动编码检测)
- TXT (自动编码检测，支持UTF-8/GBK等)

**文件大小限制**:
- 单个文件上限: 50MB (`MAX_CONTENT_LENGTH = 50 * 1024 * 1024`)
- 推荐单个文件: < 10MB（避免处理过慢）

**文本处理流程**:
```
文件上传 → 编码检测 → 文本提取 → 预处理（标准化换行、移除空行） 
   → 智能分块（500字符/块，50字符重叠） → Zep图谱构建
```

### 2.3 数据量建议

| 类型 | 建议范围 | 说明 |
|------|----------|------|
| 单个文件大小 | < 10MB | 大文件建议拆分 |
| 总文本长度 | < 100万字 | 超长文本建议提取关键章节 |
| 实体数量 | < 2000个 | Zep图谱上限，超过会被截断 |
| Agent数量 | 50-500个 | 最佳模拟效果区间 |
| 模拟时长 | 24-168小时 | 突发事件短、持续话题长 |

---

## 3. Agent生成机制

### 3.1 Agent创建流程

```python
# 1. 从Zep图谱读取实体
entities = zep_reader.filter_defined_entities(graph_id)

# 2. 为每个实体生成Profile
for entity in entities:
    profile = generator.generate_profile_from_entity(
        entity=entity,
        use_llm=True  # 使用LLM生成详细人设
    )

# 3. 保存为OASIS格式
- Reddit: JSON格式 (reddit_profiles.json)
- Twitter: CSV格式 (twitter_profiles.csv)
```

### 3.2 人设生成内容

**LLM生成的详细人设**（约2000字）包括：

| 字段 | 说明 | 示例 |
|------|------|------|
| **bio** | 社交媒体简介 | "武汉大学计算机系研究生，关注教育公平..." |
| **persona** | 详细人设描述 | 包含背景、性格、行为模式、立场观点等 |
| **age** | 年龄 | 18-60随机或基于实体类型 |
| **gender** | 性别 | male/female/other |
| **mbti** | MBTI类型 | INTJ、ENFP等16种类型 |
| **country** | 国家 | 中国、US等 |
| **profession** | 职业 | 学生、教授、记者等 |
| **interested_topics** | 感兴趣话题 | ["AI", "教育公平"] |
| **activity_level** | 活跃度 | 0.0-1.0，学生0.8，官方机构0.2 |
| **stance** | 立场 | supportive/opposing/neutral/observer |
| **influence_weight** | 影响力权重 | 官方3.0，媒体2.5，普通人1.0 |

### 3.3 实体类型区分

**个人类型实体**（生成具体人设）:
```python
INDIVIDUAL_ENTITY_TYPES = [
    "student", "alumni", "professor", "person", "publicfigure", 
    "expert", "faculty", "official", "journalist", "activist"
]
```

**群体/机构类型实体**（生成代表账号人设）:
```python
GROUP_ENTITY_TYPES = [
    "university", "governmentagency", "organization", "ngo", 
    "mediaoutlet", "company", "institution", "group", "community"
]
```

---

## 4. 数据清洗与分析机制

### 4.1 数据清洗规则（项目内部实现）

**文本预处理** (`TextProcessor.preprocess_text`):
```python
# 1. 标准化换行
text = text.replace('\r\n', '\n').replace('\r', '\n')

# 2. 移除连续空行（保留最多两个换行）
text = re.sub(r'\n{3,}', '\n\n', text)

# 3. 移除行首行尾空白
lines = [line.strip() for line in text.split('\n')]
text = '\n'.join(lines)
```

**智能分块** (`split_text_into_chunks`):
```python
# 配置参数
chunk_size = 500      # 每块字符数
overlap = 50          # 重叠字符数

# 智能句子边界检测
for sep in ['。', '！', '？', '.\n', '!\n', '?\n', '\n\n', '. ', '! ', '? ']:
    last_sep = text[start:end].rfind(sep)
    if last_sep != -1 and last_sep > chunk_size * 0.3:
        end = start + last_sep + len(sep)
        break
```

**编码自动检测** (`_read_text_with_fallback`):
```
1. 首先尝试 UTF-8
2. 使用 charset_normalizer 检测编码
3. 回退到 chardet 检测编码
4. 最终使用 UTF-8 + errors='replace' 兜底
```

### 4.2 数据分析机制

**本体生成** (LLM驱动):
- 输入: 文档内容(最多5万字) + 模拟需求
- 输出: 10个实体类型 + 6-10个关系类型 + 分析摘要
- 规则: 必须包含Person和Organization兜底类型

**实体关系提取**:
- 由Zep Cloud自动完成
- 支持时序记忆（记录实体间的互动历史）
- 支持社区摘要（自动识别实体群组）

**模拟配置生成** (LLM驱动):
```python
# 自动生成内容
time_config:        # 时间配置（符合中国人作息）
  - 深夜0-5点: 活跃度0.05
  - 工作时间9-18点: 活跃度0.7
  - 晚间高峰19-22点: 活跃度1.5

event_config:       # 事件配置
  - 初始帖子内容
  - 热点话题关键词
  - 舆论引导方向

agent_configs:      # 每个Agent的行为配置
  - 活跃度、发言频率
  - 立场、情感倾向
  - 响应速度、影响力
```

### 4.3 不依赖外部服务

| 功能 | 实现方式 | 外部依赖 |
|------|----------|----------|
| 文件解析 | 项目内部 FileParser | PyMuPDF, charset-normalizer |
| 文本清洗 | 项目内部 TextProcessor | 无（纯Python） |
| 文本分块 | 项目内部 智能分块算法 | 无 |
| 实体提取 | **LLM智能生成** | OpenAI API |
| 关系识别 | **LLM智能生成** | OpenAI API |
| 知识图谱 | Zep Cloud API | Zep Cloud |
| 数据清洗工具 | ❌ 未使用 | 无 |
| MCP服务 | ❌ 未使用 | 无 |

---

## 5. 系统集成方案

### 5.1 部署架构

**方案A: 独立部署 + API调用**（推荐）
```
┌─────────────────────────────────────────────┐
│              你的业务系统                      │
│  - 真实社交数据收集                            │
│  - 业务逻辑处理                               │
│  - 调用MiroFish API进行模拟                    │
└─────────────────────────────────────────────┘
                        │
                        ▼ REST API
┌─────────────────────────────────────────────┐
│           MiroFish 服务 (Flask)              │
│  - /api/graph/*     图谱构建                   │
│  - /api/simulation/* 模拟管理                │
│  - /api/report/*   报告生成                  │
└─────────────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   ┌─────────┐    ┌─────────┐    ┌─────────┐
   │ OpenAI  │    │Zep Cloud│    │ OASIS   │
   │ API     │    │ 图谱    │    │ 模拟引擎 │
   └─────────┘    └─────────┘    └─────────┘
```

**方案B: Docker Compose编排**
```yaml
version: '3.8'
services:
  your-system:
    build: ./your-app
    ports:
      - "3000:3000"
    networks:
      - mirofish-network
      
  mirofish-backend:
    build: ./MiroFish/backend
    ports:
      - "5001:5001"
    environment:
      - LLM_API_KEY=${LLM_API_KEY}
      - ZEP_API_KEY=${ZEP_API_KEY}
    networks:
      - mirofish-network

  mirofish-frontend:
    build: ./MiroFish/frontend
    ports:
      - "3001:3000"
    networks:
      - mirofish-network
```

### 5.2 核心API端点

| 端点 | 方法 | 功能 | 耗时 |
|------|------|------|------|
| `/api/graph/ontology/generate` | POST | 上传文档，生成本体定义 | 1-3分钟 |
| `/api/graph/build` | POST | 构建知识图谱 | 5-15分钟 |
| `/api/simulation/create` | POST | 创建模拟 | < 1秒 |
| `/api/simulation/prepare` | POST | 生成Agent配置 | 5-30分钟 |
| `/api/simulation/start` | POST | 启动模拟 | < 5秒 |
| `/api/simulation/{id}/run-status` | GET | 查询运行状态 | < 1秒 |
| `/api/simulation/interview` | POST | 采访特定Agent | 10-60秒 |
| `/api/report/generate` | POST | 生成预测报告 | 2-5分钟 |

### 5.3 集成调用流程

```python
# 示例：在你的系统中调用MiroFish进行舆情预测

import requests

MIROFISH_API = "http://localhost:5001/api"

# Step 1: 上传文档并生成本体
files = {'files': open('report.pdf', 'rb')}
data = {
    'simulation_requirement': '预测某政策发布后的舆论反应',
    'project_name': '政策舆情分析'
}
response = requests.post(
    f"{MIROFISH_API}/graph/ontology/generate",
    files=files, data=data
)
project_id = response.json()['data']['project_id']

# Step 2: 构建图谱
response = requests.post(
    f"{MIROFISH_API}/graph/build",
    json={'project_id': project_id}
)
task_id = response.json()['data']['task_id']

# Step 3: 等待图谱构建完成（轮询）
while True:
    status = requests.get(f"{MIROFISH_API}/graph/task/{task_id}")
    if status.json()['data']['status'] == 'completed':
        break
    time.sleep(5)

# Step 4: 创建并启动模拟
sim = requests.post(
    f"{MIROFISH_API}/simulation/create",
    json={'project_id': project_id}
)
simulation_id = sim.json()['data']['simulation_id']

# Step 5: 准备模拟环境
requests.post(
    f"{MIROFISH_API}/simulation/prepare",
    json={'simulation_id': simulation_id}
)

# Step 6: 启动模拟
requests.post(
    f"{MIROFISH_API}/simulation/start",
    json={'simulation_id': simulation_id, 'platform': 'parallel'}
)

# Step 7: 监控模拟进度并生成报告
# ...
```

---

## 6. 基于真实社交数据的扩展建议

### 6.1 当前限制

**MiroFish原生不支持**:
- ❌ 直接导入微博/微信/抖音等真实社交数据
- ❌ 基于真实用户行为训练
- ❌ 特定账号的个性化行为预测

### 6.2 扩展方案设计

#### 新增模块1: 真实数据导入服务

```python
# backend/app/api/external_data.py（新增）

@external_bp.route('/import/social-profile', methods=['POST'])
def import_social_profile():
    """
    导入真实社交媒体账号数据，生成Agent Profile
    
    请求体：
    {
        "account_id": "user_123",
        "platform": "weibo",  // 微博、微信、抖音等
        "profile_data": {
            "username": "真实用户名",
            "bio": "个人简介",
            "historical_posts": ["历史发帖1", "历史发帖2", ...],
            "interaction_patterns": {
                "post_frequency": "daily",
                "active_hours": [9, 10, 11, 14, 20, 21],
                "common_topics": ["科技", "美食"],
                "typical_sentiment": "positive",
                "interaction_style": "aggressive"
            }
        },
        "simulation_requirement": "模拟该账号对AI监管政策的反应"
    }
    """
    # 实现：解析真实数据 → LLM分析历史发帖 → 生成Agent Profile
```

#### 新增模块2: 真实数据Profile生成器

```python
# backend/app/services/real_data_profile_generator.py（新增）

class RealDataProfileGenerator(OasisProfileGenerator):
    """
    基于真实社交媒体数据的Agent生成器
    """
    
    def generate_from_real_data(
        self,
        account_data: Dict[str, Any],
        use_llm_analysis: bool = True
    ) -> OasisAgentProfile:
        """
        从真实账号数据生成Agent Profile
        
        分析维度：
        1. 历史发帖分析（话题偏好、语言风格、情感倾向）
        2. 互动模式分析（发帖频率、活跃时段、回应方式）
        3. 社交网络分析（关注关系、影响力范围）
        """
        if use_llm_analysis:
            # 使用LLM深度分析历史数据
            persona = self._analyze_with_llm(
                posts=account_data.get('historical_posts', []),
                bio=account_data.get('bio', ''),
                interactions=account_data.get('interaction_history', [])
            )
        
        return OasisAgentProfile(
            user_id=account_data['account_id'],
            user_name=account_data['username'],
            name=account_data.get('display_name', account_data['username']),
            bio=account_data.get('bio', ''),
            persona=persona,
            activity_level=self._calculate_real_activity(account_data),
            active_hours=self._extract_active_hours(account_data),
            interested_topics=account_data['interaction_patterns']['common_topics'],
            # ... 其他字段
        )
```

#### 新增模块3: 单账号行为预测API

```python
# backend/app/api/prediction.py（新增）

@prediction_bp.route('/predict-account-behavior', methods=['POST'])
def predict_account_behavior():
    """
    针对特定真实账号进行行为预测
    
    适用场景：
    - 预测某个KOL对品牌活动的反应
    - 模拟特定用户对政策的观点表达
    - 评估目标账号在舆情事件中的影响力
    """
    data = request.get_json()
    
    # 1. 基于真实数据创建目标Agent
    agent_profile = RealDataProfileGenerator().generate_from_real_data(
        data['account_profile']
    )
    
    # 2. 创建场景化的背景Agent
    background_agents = generate_scenario_agents(data['scenario'])
    
    # 3. 创建聚焦式模拟环境
    simulation = create_targeted_simulation(
        target_agent=agent_profile,
        background_agents=background_agents,
        scenario=data['scenario']
    )
    
    # 4. 运行模拟并专项采访
    interview_results = conduct_focused_interviews(
        simulation_id=simulation['id'],
        target_agent_id=agent_profile.user_id,
        questions=data['prediction_config']['interview_questions']
    )
    
    # 5. 生成预测报告
    return jsonify({
        "success": True,
        "data": {
            "predicted_reactions": prediction_reactions,
            "likely_posts": predicted_posts,
            "influence_assessment": influence_analysis,
            "confidence_score": 0.85
        }
    })
```

### 6.3 扩展开发工作量

| 任务 | 预计工时 | 优先级 |
|------|----------|--------|
| 新增数据导入API | 4-8小时 | P1 |
| 扩展Profile生成器 | 8-16小时 | P1 |
| 单账号预测模块 | 16-24小时 | P2 |
| 批量账号预测 | 8-12小时 | P3 |
| 实时数据接入（WebSocket） | 16-24小时 | P3 |
| 集成测试 | 8-12小时 | P1 |
| **总计** | **60-96小时** | 约2-3周 |

---

## 7. 环境要求与配置

### 7.1 系统要求

| 组件 | 最低要求 | 推荐配置 |
|------|----------|----------|
| **操作系统** | Windows 10 / Linux | Windows 11 / Ubuntu 22.04 |
| **Python** | 3.11 | 3.11-3.12 |
| **Node.js** | 18+ | 20 LTS |
| **内存** | 8GB | 16GB+ |
| **存储** | 10GB | 50GB+ SSD |
| **网络** | 可访问OpenAI和Zep | 稳定互联网连接 |

### 7.2 必需的环境变量

```env
# LLM API配置（支持OpenAI SDK格式的任意LLM API）
LLM_API_KEY=your_api_key
LLM_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
LLM_MODEL_NAME=qwen-plus

# Zep Cloud配置（知识图谱服务）
ZEP_API_KEY=your_zep_api_key

# 可选配置
FLASK_HOST=0.0.0.0
FLASK_PORT=5001
FLASK_DEBUG=True
```

### 7.3 推荐的LLM模型

| 模型 | 提供商 | 效果 | 成本 |
|------|--------|------|------|
| **qwen-plus** | 阿里云百炼 | ⭐⭐⭐⭐⭐ 最佳中文效果 | 中等 |
| **gpt-4o** | OpenAI | ⭐⭐⭐⭐⭐ 最佳通用效果 | 较高 |
| **gpt-4o-mini** | OpenAI | ⭐⭐⭐⭐ 性价比高 | 低 |
| **qwen-turbo** | 阿里云百炼 | ⭐⭐⭐⭐ 快速经济 | 低 |

---

## 8. 关键限制与注意事项

### 8.1 数据处理能力限制

| 限制项 | 限制值 | 说明 |
|--------|--------|------|
| 单个文件大小 | 50MB | 超过需拆分 |
| LLM上下文长度 | 50,000字符 | 超长文本会被截断 |
| Zep图谱实体数 | 2,000个 | 超过会被截断 |
| Agent配置批次 | 15个/批 | 分批生成避免超时 |
| 模拟时长 | 24-168小时 | 根据场景灵活调整 |
| 每小时激活Agent | 1-总数的90% | 避免过载 |

### 8.2 API调用成本估算

**小规模模拟（50个Agent，72轮）**:
- 本体生成: ~1-2次LLM调用
- Profile生成: ~50次LLM调用（可并行）
- 模拟配置生成: ~5-10次LLM调用
- 报告生成: ~10-20次LLM调用
- **总计**: ~70-80次LLM调用

**以qwen-plus为例**（阿里云百炼）:
- 预估成本: 每次模拟约 **5-10元人民币**
- 月度100次模拟: **500-1000元**

### 8.3 数据隐私与安全

- **文本存储**: 上传文档保存在本地 `backend/uploads/projects/`
- **图谱数据**: 存储在Zep Cloud（需遵守Zep隐私政策）
- **模拟数据**: 保存在本地SQLite数据库
- **敏感数据**: 建议本地部署，避免上传云端

---

## 9. 总结与建议

### 9.1 当前MiroFish能力

✅ **优势**:
- 完整的文档→图谱→Agent→模拟→报告全流程
- LLM驱动的智能实体提取和人设生成
- 双平台（Twitter/Reddit）并行模拟
- 支持模拟过程中的深度互动和采访
- 模块化架构，易于扩展

❌ **局限**:
- 不直接支持外部社交平台数据导入
- 不支持基于真实用户行为的个性化预测
- 数据清洗能力较基础（仅文本预处理）
- 依赖LLM质量，成本较高

### 9.2 集成建议

**短期（1-2周）**:
1. 独立部署MiroFish，通过API调用进行宏观舆情推演
2. 将真实社交数据整理成文档格式上传，作为"现实种子"
3. 验证模拟效果，调整Prompt和配置

**中期（2-4周）**:
1. 开发数据导入模块，支持结构化的真实账号数据
2. 扩展Profile生成器，分析历史发帖生成个性化人设
3. 实现单账号行为预测功能

**长期（1-3月）**:
1. 接入实时数据流（WebSocket/Kafka）
2. 构建反馈循环：真实互动数据 → 优化Agent行为模型
3. 开发可视化监控面板，实时观察模拟与真实数据对比

### 9.3 联系方式

- **官方仓库**: https://github.com/666ghj/MiroFish
- **在线Demo**: https://666ghj.github.io/mirofish-demo/
- **团队邮箱**: mirofish@shanda.com

---

**文档结束**

*注：本文档基于MiroFish v0.1.2版本分析生成，后续版本可能有变化，请以官方文档为准。*
