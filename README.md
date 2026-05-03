# 基于AI Agent的量化选股策略

> 通过多个专业化Agent模拟人类量化团队分工协作，实现从策略构思、因子挖掘、回测验证到实盘执行的全流程自动化闭环


## ✨ 核心功能

|Agent类型|Agent|核心能力|技术实现|
|-------|------|------|------|
|数据agent|数据工程师|自动拉取多源数据（行情、财报、新闻），清洗并构建特征|API对接+向量数据库|
|因子agent|量化研究员|从非结构化数据（研报、公告）中抽取因子，利用LLM生成创新因子组合|GPT-4 + 强化学习|
|策略agent|策略分析师|将因子自核成交易逻辑，自动编写回测带（支持Qlib等框架）|代码生成模型+模板引擎|
|风控agent|风险官|实时监控敞口、波动率，动态调整风险预算，硬止损|规则引擎+预警系统|
|执行agent|交易员|接受策略信号，通过TWAP/VWAP算法拆单执行，处理异常交易|事件驱动架构+低延迟柜台|



## 🛠️ 技术栈

- HelloAgents框架
- Redis
- 使用ReAct范式
- 使用的工具和API

<!-- ## 🚀 快速开始

### 环境要求

- Python 3.10+
- 其他要求

### 安装依赖

```bash
pip install -r requirements.txt
``` -->

### 配置API密钥

```bash
# 创建.env文件
cp .env.example .env

# 编辑.env文件，填入你的API密钥
# OPENAI_API_KEY=your_key_here
```

### 运行项目

```bash
# 启动Jupyter Notebook
jupyter lab

# 打开main.ipynb并运行
```

## 📖 使用示例

展示如何使用你的项目，最好包含代码示例和运行结果。

```python
# 示例代码
from hello_agents import SimpleAgent

agent = SimpleAgent(name="示例智能体")
result = agent.run("你的输入")
print(result)
```


## 🔮 未来计划

- [ ] 待实现的功能1
- [ ] 待实现的功能2
- [ ] 待优化的部分

## 🤝 贡献指南

欢迎提出Issue和Pull Request！

## 📄 许可证

MIT License

## 👤 作者

- GitHub: [@qiqiqiyi](https://github.com/qiqiqiyi)
- Email: 18262768565@163.com

## 🙏 致谢

感谢Datawhale社区和Hello-Agents项目！
