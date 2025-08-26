# CAMEL架构学习

## 多智能体

多智能体（Multiple Agent）由多个相互作用的智能体组成，每个智能体都有自己的目标和策略。这些智能体可以相互通信、协作或竞争，以实现更复杂的行为和决策。

**特点**：

- **协作**：智能体之间可以协作，共同解决问题。
- **竞争**：智能体之间也可以存在竞争关系，如在拍卖或游戏场景中。
- **自主性**：每个智能体都有自己的决策过程，保持一定程度的自主性。
- **复杂性**：多智能体系统的设计与分析比单一智能体系统更复杂，因为需要考虑智能体之间的交互和协调。
- **鲁棒性**：多智能体系统通常具有更好的鲁棒性，因为系统的稳定性和效能不完全依赖于单一决策者。 

## CAMEL

在CAMEL框架中，`ChatAgent` 是最基础的智能体单元，负责处理对话逻辑和任务执行。而`RolePlaying` 和`Workforce` 则是多智能体系统，用于协调多个智能体的协作。

### ChatAgent

ChatAgent 是 CAMEL 框架的基础构建块,具备以下关键特性:

- **角色** **(Role)**：结合目标和内容规范，设定智能体的初始状态，引导智能体在连续交互过程中采取行动。
- **大语言模型** **(LLMs)**：每个智能体都使用大语言模型来增强认知能力。大语言模型使智能体能够理解和生成自然语言，从而解释指令、生成响应并参与复杂对话。
- **记忆** **(Memory)**：包括上下文记忆和外部记忆，使智能体能够以更扎实的方式进行推理和学习。
- **工具** **(Tools)**：智能体可以使用的一组功能，用于与外部世界交互，本质上是为智能体提供具身化能力。
- **通信** **(Communication)**：我们的框架允许智能体之间进行灵活且可扩展的通信，这是解决关键研究问题的基础。
- **推理** **(Reasoning)**：我们为智能体配备了不同的规划和奖励（评论员）学习能力，使其能够以更有指导性的方式优化任务完成。

### RolePlaying

RolePlaying是CAMEL框架的独特合作式智能体框架。该框架通过预定义的提示词为不同的智能体创建唯一的初始设置，帮助智能体克服诸如角色翻转、助手重复指令、模糊回复、消息无限循环以及对话终止条件等多个挑战。

**工作流程：**

初始化阶段

- 设定角色身份
- 加载系统提示词
- 明确任务目标

执行阶段

- User提供具体指令
- Assistant执行并给出解决方案
- 循环往复直至完成任务bi

案例——股票交易员

```python
from colorama import Fore

from camel.societies import RolePlaying
from camel.utils import print_text_animated

def main(model=YOUR_MODEL, chat_turn_limit=50) -> None:
    task_prompt = "Develop a trading bot for the stock market"
    role_play_session = RolePlaying(
        assistant_role_name="Python Programmer",
        assistant_agent_kwargs=dict(model=model),
        user_role_name="Stock Trader",
        user_agent_kwargs=dict(model=model),
        task_prompt=task_prompt,
        with_task_specify=True,
        task_specify_agent_kwargs=dict(model=model),
    )
```



### Workforce

Workforce采用层级架构设计。一个workforce可以包含多个工作节点(worker nodes)，每个工作节点可以包含一个或多个智能体作为工作者。工作节点由workforce内部的协调智能体(coordinator agent)管理，协调智能体根据工作节点的描述及其工具集来分配任务。workforce内部还有一个任务规划智能体(task planner agent)。任务规划智能体负责任务的分解和组合，使workforce能够逐步解决任务。

Workforce内部的通信基于任务通道(task channel)。Workforce初始化时会创建一个所有节点共享的通道。任务会被发布到这个通道中，每个工作节点会监听通道，接受分配给它的任务并解决。

当任务完成后，工作节点会将结果发布回通道，结果会作为其他任务的"依赖项"保留在通道中，供所有工作节点共享。

通过这种机制，workforce能够以协作和高效的方式解决任务。

Workforce具有故障处理机制。当任务失败时，协调智能体会采取行动修复。这些行动可以是：

- 将任务分解为更小的任务并重新分配
- 创建一个能够完成该任务的新工作者
