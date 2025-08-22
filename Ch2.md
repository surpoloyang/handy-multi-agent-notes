# Agent的构成组件

## Agent的提示词工程

我们使用`camel.prompts`的`AISocietyPromptTemplateDict`类为“健康顾问给高血压和糖尿病患者提供建议与咨询服务”主题角色扮演的详细任务prompt。
代码如下：

```python
from colorama import Fore
from camel.societies import RolePlaying
from camel.utils import print_text_animated
from camel.models import ModelFactory
from camel.types import ModelPlatformType
from camel.prompts import AISocietyPromptTemplateDict
from dotenv import load_dotenv
import os

load_dotenv()
api_key = os.getenv('MODELSCOPE_SDK_TOKEN')

# 创建模型
model = ModelFactory.create(
    model_platform=ModelPlatformType.OPENAI_COMPATIBLE_MODEL,
    model_type="Qwen/Qwen2.5-72B-Instruct",
    url='https://api-inference.modelscope.cn/v1/',
    api_key=api_key
)

def main(model=model, chat_turn_limit=10) -> None:
    # 使用 AISocietyPromptTemplateDict 创建角色模板
    prompt_template_dict = AISocietyPromptTemplateDict()
    # 设置健康咨询任务
    task_prompt = "健康顾问为患者提供专业的健康建议和咨询服务"
    task_specify_prompt = prompt_template_dict.TASK_SPECIFY_PROMPT.format(
        assistant_role="健康顾问",
        user_role="患者",
        task=task_prompt,
        word_limit= 1000,
    )
    # 创建角色扮演会话
    role_play_session = RolePlaying(
        assistant_role_name="健康顾问",
        assistant_agent_kwargs=dict(
            model=model,
        ),
        user_role_name="患者",
        user_agent_kwargs=dict(
            model=model,
        ),
        task_prompt=task_specify_prompt,
        with_task_specify=False,
        output_language='中文'
    )

    print(Fore.CYAN + "=" * 60)
    print(Fore.CYAN + "健康顾问AI角色扮演系统")
    print(Fore.CYAN + "=" * 60 + "\n")

    print(
        Fore.GREEN
        + f"健康顾问系统消息:\n{role_play_session.assistant_sys_msg}\n"
    )
    print(
        Fore.BLUE + f"患者系统消息:\n{role_play_session.user_sys_msg}\n"
    )

    print(Fore.YELLOW + f"原始任务提示:\n{task_prompt}\n")
    print(
        Fore.CYAN
        + "指定的任务提示:"
        + f"\n{role_play_session.specified_task_prompt}\n"
    )
    print(Fore.RED + f"最终任务提示:\n{role_play_session.task_prompt}\n")

    print(Fore.MAGENTA + "开始健康咨询会话...\n")

    n = 0
    input_msg = role_play_session.init_chat()
    
    while n < chat_turn_limit:
        n += 1
        print(Fore.CYAN + f"=== 第 {n} 轮咨询 ===\n")
        
        assistant_response, user_response = role_play_session.step(input_msg)

        if assistant_response.terminated:
            print(
                Fore.GREEN
                + (
                    "健康顾问已终止咨询。原因: "
                    f"{assistant_response.info['termination_reasons']}."
                )
            )
            break
            
        if user_response.terminated:
            print(
                Fore.GREEN
                + (
                    "患者已终止咨询。"
                    f"原因: {user_response.info['termination_reasons']}."
                )
            )
            break

        # 打印患者的回应
        print_text_animated(
            Fore.BLUE + f"患者:\n\n{user_response.msg.content}\n"
        )
        
        # 检查任务完成标志
        if "CAMEL_TASK_DONE" in user_response.msg.content:
            print(Fore.GREEN + "健康咨询任务完成！")
            break

        # 打印健康顾问的建议
        print_text_animated(
            Fore.GREEN + "健康顾问:\n\n"
            f"{assistant_response.msg.content}\n"
        )

        input_msg = assistant_response.msg
        print(Fore.CYAN + "-" * 50 + "\n")

    if n >= chat_turn_limit:
        print(Fore.YELLOW + "达到最大咨询轮次限制。")
    
    print(Fore.MAGENTA + "\n健康咨询会话结束。")
    print(Fore.CYAN + "感谢使用健康顾问AI系统！")

if __name__ == "__main__":
    main()

```

信息如下：
```bash
============================================================
健康顾问AI角色扮演系统
============================================================

健康顾问系统消息:
BaseMessage(role_name='健康顾问', role_type=<RoleType.ASSISTANT: 'assistant'>, meta_dict={'task': 'Here is a task that 健康顾问 will help 患者 to complete: 健康顾问为患者提供专业的健康建议和咨询服务.\nPlease make it more specific. Be creative and imaginative.\nPlease reply with the specified task in 1000 words or less. Do not add anything else.', 'assistant_role': '健康顾问', 'user_role': '患者'}, content='===== RULES OF ASSISTANT =====\nNever forget you are a 健康顾问 and I am a 患者. Never flip roles! Never instruct me!\nWe share a common interest in collaborating to successfully complete a task.\nYou must help me to complete the task.\nHere is the task: Here is a task that 健康顾问 will help 患者 to complete: 健康顾问为患者提供专业的健康建议和咨询服务.\nPlease make it more specific. Be creative and imaginative.\nPlease reply with the specified task in 1000 words or less. Do not add anything else.. Never forget our task!\nI must instruct you based on your expertise and my needs to complete the task.\n\nI must give you one instruction at a time.\nYou must write a specific solution that appropriately solves the requested instruction and explain your solutions.\nYou must decline my instruction honestly if you cannot perform the instruction due to physical, moral, legal reasons or your capability and explain the reasons.\nUnless I say the task is completed, you should always start with:\n\nSolution: <YOUR_SOLUTION>\n\n<YOUR_SOLUTION> should be very specific, include detailed explanations and provide preferable detailed implementations and examples and lists for task-solving.\nAlways end <YOUR_SOLUTION> with: Next request.\nRegardless of the input language, you must output text in 中文.', video_bytes=None, image_list=None, image_detail='auto', video_detail='low', parsed=None)

患者系统消息:
BaseMessage(role_name='患者', role_type=<RoleType.USER: 'user'>, meta_dict={'task': 'Here is a task that 健康顾问 will help 患者 to complete: 健康顾问为患者提供专业的健康建议和咨询服务.\nPlease make it more specific. Be creative and imaginative.\nPlease reply with the specified task in 1000 words or less. Do not add anything else.', 'assistant_role': '健康顾问', 'user_role': '患者'}, content='===== RULES OF USER =====\nNever forget you are a 患者 and I am a 健康顾问. Never flip roles! You will always instruct me.\nWe share a common interest in collaborating to successfully complete a task.\nI must help you to complete the task.\nHere is the task: Here is a task that 健康顾问 will help 患者 to complete: 健康顾问为患者提供专业的健康建议和咨询服务.\nPlease make it more specific. Be creative and imaginative.\nPlease reply with the specified task in 1000 words or less. Do not add anything else.. Never forget our task!\nYou must instruct me based on my expertise and your needs to solve the task ONLY in the following two ways:\n\n1. Instruct with a necessary input:\nInstruction: <YOUR_INSTRUCTION>\nInput: <YOUR_INPUT>\n\n2. Instruct without any input:\nInstruction: <YOUR_INSTRUCTION>\nInput: None\n\nThe "Instruction" describes a task or question. The paired "Input" provides further context or information for the requested "Instruction".\n\nYou must give me one instruction at a time.\nI must write a response that appropriately solves the requested instruction.\nI must decline your instruction honestly if I cannot perform the instruction due to physical, moral, legal reasons or my capability and explain the reasons.\nYou should instruct me not ask me questions.\nNow you must start to instruct me using the two ways described above.\nDo not add anything else other than your instruction and the optional corresponding input!\nKeep giving me instructions and necessary inputs until you think the task is completed.\nWhen the task is completed, you must only reply with a single word <CAMEL_TASK_DONE>.\nNever say <CAMEL_TASK_DONE> unless my responses have solved your task.\nRegardless of the input language, you must output text in 中文.', video_bytes=None, image_list=None, image_detail='auto', video_detail='low', parsed=None)

原始任务提示:
健康顾问为患者提供专业的健康建议和咨询服务

指定的任务提示:
None

最终任务提示:
Here is a task that 健康顾问 will help 患者 to complete: 健康顾问为患者提供专业的健康建议和咨询服务.
Please make it more specific. Be creative and imaginative.
Please reply with the specified task in 1000 words or less. Do not add anything else.

开始健康咨询会话...

=== 第 1 轮咨询 ===

患者:

Instruction: 请帮我制定一个为期四周的健康饮食计划，确保营养均衡。
Input: 我有高血压和糖尿病，需要特别注意饮食中的盐分和糖分摄入。
健康顾问:

Solution: 针对您的高血压和糖尿病情况，我将为您设计一个为期四周的健康饮食计划，确保营养均衡，并特别关注盐分和糖分的摄入量。

### 第一周：适应期

**目标**：逐步调整饮食习惯，减少盐分和糖分的摄入，增加膳食纤维。

#### 早餐：
- **周一至周五**：全麦面包2片，搭配一份低脂牛奶或豆浆，一份新鲜水果（如苹果、梨）。
- **周六**：燕麦粥，加入少量坚果和蓝莓。
- **周日**：鸡蛋1个，搭配绿叶蔬菜沙拉（使用橄榄油和柠檬汁调味）。

#### 午餐：
- **周一至周五**：瘦肉（鸡胸肉、鱼肉）100克，搭配大量蔬菜（如西兰花、胡萝卜、菠菜），糙米饭一小碗。
- **周六**：豆腐炒青菜，搭配小米粥。
- **周日**：烤三文鱼，搭配烤蔬菜（如甜椒、洋葱）。

#### 晚餐：
- **周一至周五**：清蒸鱼，搭配绿叶蔬菜和糙米。
- **周六**：蔬菜汤（少盐），搭配全麦面包。
- **周日**：鸡胸肉沙拉，搭配橄榄油和醋调味。

#### 小吃/加餐：
- **每日**：一份新鲜水果，或一小把坚果（如杏仁、核桃）。

### 第二周：巩固期

**目标**：继续减少盐分和糖分的摄入，增加蛋白质和不饱和脂肪酸的摄入。

#### 早餐：
- **周一至周五**：燕麦粥，加入少量坚果和水果。
- **周六**：全麦吐司，搭配低脂奶酪和新鲜番茄。
- **周日**：鸡蛋1个，搭配绿叶蔬菜沙拉（使用橄榄油和柠檬汁调味）。

#### 午餐：
- **周一至周五**：瘦肉（鸡胸肉、鱼肉）100克，搭配大量蔬菜（如西兰花、胡萝卜、菠菜），糙米饭一小碗。
- **周六**：豆腐炒青菜，搭配小米粥。
- **周日**：烤三文鱼，搭配烤蔬菜（如甜椒、洋葱）。

#### 晚餐：
- **周一至周五**：清蒸鱼，搭配绿叶蔬菜和糙米。
- **周六**：蔬菜汤（少盐），搭配全麦面包。
- **周日**：鸡胸肉沙拉，搭配橄榄油和醋调味。

#### 小吃/加餐：
- **每日**：一份新鲜水果，或一小把坚果（如杏仁、核桃）。

### 第三周：提升期

**目标**：进一步优化饮食结构，增加抗氧化食物和高纤维食物。

#### 早餐：
- **周一至周五**：全麦面包2片，搭配一份低脂牛奶或豆浆，一份新鲜水果（如橙子、猕猴桃）。
- **周六**：燕麦粥，加入少量坚果和蓝莓。
- **周日**：鸡蛋1个，搭配绿叶蔬菜沙拉（使用橄榄油和柠檬汁调味）。

#### 午餐：
- **周一至周五**：瘦肉（鸡胸肉、鱼肉）100克，搭配大量蔬菜（如西兰花、胡萝卜、菠菜），糙米饭一小碗。
- **周六**：豆腐炒青菜，搭配小米粥。
- **周日**：烤三文鱼，搭配烤蔬菜（如甜椒、洋葱）。

#### 晚餐：
- **周一至周五**：清蒸鱼，搭配绿叶蔬菜和糙米。
- **周六**：蔬菜汤（少盐），搭配全麦面包。
- **周日**：鸡胸肉沙拉，搭配橄榄油和醋调味。

#### 小吃/加餐：
- **每日**：一份新鲜水果，或一小把坚果（如杏仁、核桃）。

### 第四周：维持期

**目标**：保持良好的饮食习惯，确保长期健康。

#### 早餐：
- **周一至周五**：燕麦粥，加入少量坚果和水果。
- **周六**：全麦吐司，搭配低脂奶酪和新鲜番茄。
- **周日**：鸡蛋1个，搭配绿叶蔬菜沙拉（使用橄榄油和柠檬汁调味）。

#### 午餐：
- **周一至周五**：瘦肉（鸡胸肉、鱼肉）100克，搭配大量蔬菜（如西兰花、胡萝卜、菠菜），糙米饭一小碗。
- **周六**：豆腐炒青菜，搭配小米粥。
- **周日**：烤三文鱼，搭配烤蔬菜（如甜椒、洋葱）。

#### 晚餐：
- **周一至周五**：清蒸鱼，搭配绿叶蔬菜和糙米。
- **周六**：蔬菜汤（少盐），搭配全麦面包。
- **周日**：鸡胸肉沙拉，搭配橄榄油和醋调味。

#### 小吃/加餐：
- **每日**：一份新鲜水果，或一小把坚果（如杏仁、核桃）。

### 注意事项：
1. **盐分摄入**：尽量减少食盐的使用，可以使用香草、柠檬汁等天然调料来增加食物的风味。
2. **糖分摄入**：避免食用含糖饮料和高糖食品，选择低糖或无糖的替代品。
3. **水分摄入**：每天至少喝8杯水，保持充足的水分摄入。
4. **定期监测**：每周监测血压和血糖水平，及时调整饮食计划。
5. **适量运动**：结合适量的有氧运动，如快走、游泳等，有助于控制体重和改善心血管健康。

希望这个饮食计划能帮助您更好地管理高血压和糖尿病，如有任何不适，请及时咨询医生。 Next request.
--------------------------------------------------

=== 第 2 轮咨询 ===

患者:

CAMEL_TASK_DONE
健康咨询任务完成！

健康咨询会话结束。
感谢使用健康顾问AI系统！
```



## Agent的工具调用

我们给模型提供搜索工具与数学计算工具的接口，赋予大模型搜索与计算的能力。

代码如下：
```python
from camel.toolkits import SearchToolkit,MathToolkit
from camel.types.agents import ToolCallingRecord
from camel.models import ModelFactory
from camel.types import ModelPlatformType
from camel.societies import RolePlaying
from camel.utils import print_text_animated
from colorama import Fore

import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('MODELSCOPE_SDK_TOKEN')
# 设置代理
#os.environ["http_proxy"] = "http://127.0.0.1:7897"
#os.environ["https_proxy"] = "http://127.0.0.1:7897"

#定义工具包
tools_list = [
    *SearchToolkit().get_tools(),
    *MathToolkit().get_tools()
]

# 设置任务
task_prompt = ("假设现在是2024年，"
        "使用百度搜索工具查找牛津大学的成立年份，并计算出其当前年龄。"
        "然后再将这个年龄加上10年。")

# 创建模型
model = ModelFactory.create(
    model_platform=ModelPlatformType.OPENAI_COMPATIBLE_MODEL,
    model_type="Qwen/Qwen2.5-72B-Instruct",
    url='https://api-inference.modelscope.cn/v1/',
    api_key=api_key
)

# 设置角色扮演
role_play_session = RolePlaying(
    assistant_role_name="搜索者",
    user_role_name="教授",
    assistant_agent_kwargs=dict(
        model=model,
        tools=tools_list,
    ),
    user_agent_kwargs=dict(
        model=model,
    ),
    task_prompt=task_prompt,
    with_task_specify=False,
    output_language='中文'
)

# 设置聊天轮次限制
chat_turn_limit=10

print(
    Fore.GREEN
    + f"AI助手系统消息:\n{role_play_session.assistant_sys_msg}\n"
)
print(
    Fore.BLUE + f"AI用户系统消息:\n{role_play_session.user_sys_msg}\n"
)

print(Fore.YELLOW + f"原始任务提示:\n{task_prompt}\n")
print(
    Fore.CYAN
    + "指定的任务提示:"
    + f"\n{role_play_session.specified_task_prompt}\n"
)
print(Fore.RED + f"最终任务提示:\n{role_play_session.task_prompt}\n")

n = 0
input_msg = role_play_session.init_chat()
while n < chat_turn_limit:
    n += 1
    assistant_response, user_response = role_play_session.step(input_msg)

    if assistant_response.terminated:
        print(
            Fore.GREEN
            + (
                "AI助手终止。原因: "
                f"{assistant_response.info['termination_reasons']}."
            )
        )
        break
    if user_response.terminated:
        print(
            Fore.GREEN
            + (
                "AI用户终止。"
                f"原因: {user_response.info['termination_reasons']}."
            )
        )
        break

    # 打印用户的输出
    print_text_animated(
        Fore.BLUE + f"AI用户:\n\n{user_response.msg.content}\n"
    )

    if "CAMEL_TASK_DONE" in user_response.msg.content:
        break

    # 打印助手的输出，包括任何函数执行信息
    print_text_animated(Fore.GREEN + "AI助手:")
    tool_calls: list[ToolCallingRecord] = assistant_response.info[
        'tool_calls'
    ]
    for func_record in tool_calls:
        print_text_animated(f"{func_record}")
    print_text_animated(f"{assistant_response.msg.content}\n")

    input_msg = assistant_response.msg
```

消息如下：

```bash
AI助手系统消息:
BaseMessage(role_name='搜索者', role_type=<RoleType.ASSISTANT: 'assistant'>, meta_dict={'task': '假设现在是2024年，使用百度搜索工具查找牛津大学的成立年份，并计算出其当前年龄。然后再将这个年龄加上10年。', 'assistant_role': '搜索者', 'user_role': '教授'}, content='===== RULES OF ASSISTANT =====\nNever forget you are a 搜索者 and I am a 教授. Never flip roles! Never instruct me!\nWe share a common interest in collaborating to successfully complete a task.\nYou must help me to complete the task.\nHere is the task: 假设现在是2024年，使用百度搜索工具查找牛津大学的成立年份，并计算出其当前年龄。然后再将这个年龄加上10年。. Never forget our task!\nI must instruct you based on your expertise and my needs to complete the task.\n\nI must give you one instruction at a time.\nYou must write a specific solution that appropriately solves the requested instruction and explain your solutions.\nYou must decline my instruction honestly if you cannot perform the instruction due to physical, moral, legal reasons or your capability and explain the reasons.\nUnless I say the task is completed, you should always start with:\n\nSolution: <YOUR_SOLUTION>\n\n<YOUR_SOLUTION> should be very specific, include detailed explanations and provide preferable detailed implementations and examples and lists for task-solving.\nAlways end <YOUR_SOLUTION> with: Next request.\nRegardless of the input language, you must output text in 中文.', video_bytes=None, image_list=None, image_detail='auto', video_detail='low', parsed=None)

AI用户系统消息:
BaseMessage(role_name='教授', role_type=<RoleType.USER: 'user'>, meta_dict={'task': '假设现在是2024年，使用百度搜索工具查找牛津大学的成立年份，并计算出其当前年龄。然后再将这个年龄加上10年。', 'assistant_role': '搜索者', 'user_role': '教授'}, content='===== RULES OF USER =====\nNever forget you are a 教授 and I am a 搜索者. Never flip roles! You will always instruct me.\nWe share a common interest in collaborating to successfully complete a task.\nI must help you to complete the task.\nHere is the task: 假设现在是2024年，使用百度搜索工具查找牛津大学的成立年份，并计算出其当前年龄。然后再将这个年龄加上10年。. Never forget our task!\nYou must instruct me based on my expertise and your needs to solve the task ONLY in the following two ways:\n\n1. Instruct with a necessary input:\nInstruction: <YOUR_INSTRUCTION>\nInput: <YOUR_INPUT>\n\n2. Instruct without any input:\nInstruction: <YOUR_INSTRUCTION>\nInput: None\n\nThe "Instruction" describes a task or question. The paired "Input" provides further context or information for the requested "Instruction".\n\nYou must give me one instruction at a time.\nI must write a response that appropriately solves the requested instruction.\nI must decline your instruction honestly if I cannot perform the instruction due to physical, moral, legal reasons or my capability and explain the reasons.\nYou should instruct me not ask me questions.\nNow you must start to instruct me using the two ways described above.\nDo not add anything else other than your instruction and the optional corresponding input!\nKeep giving me instructions and necessary inputs until you think the task is completed.\nWhen the task is completed, you must only reply with a single word <CAMEL_TASK_DONE>.\nNever say <CAMEL_TASK_DONE> unless my responses have solved your task.\nRegardless of the input language, you must output text in 中文.', video_bytes=None, image_list=None, image_detail='auto', video_detail='low', parsed=None)

原始任务提示:
假设现在是2024年，使用百度搜索工具查找牛津大学的成立年份，并计算出其当前年龄。然后再将这个年龄加上10年。

指定的任务提示:
None

最终任务提示:
假设现在是2024年，使用百度搜索工具查找牛津大学的成立年份，并计算出其当前年龄。然后再将这个年龄加上10年。

AI用户:

Instruction: 使用百度搜索引擎查询牛津大学的成立年份。
Input: None
AI助手:Tool Execution: search_baidu
        Args: {'query': '牛津大学 成立年份', 'max_results': 5}
        Result: {'results': [{'result_id': 1, 'title': '牛津大学| University of Oxford', 'description': '', 'url': 'http://www.baidu.com/link?url=nhBzVxeDaOxzN-NjdqhkW8xIN5Tj0-vnBZjug1Laaw6k0uL-iQBNTB8l0Ob9bEAP'}, {'result_id': 2, 'title': 'History | University of Oxford', 'description': '', 'url': 'http://www.baidu.com/link?url=PxaKRDUALTXm6wXtDc3_bCUHEIL5AhHT50zd3Z-sej502I8O6HjLQESP207cFqx8OPnayW0Ro0EwDAR0zA0g8K'}, {'result_id': 3, 'title': 'Art & Architecture - Nuffield College Oxford University', 'description': '', 'url': 'http://www.baidu.com/link?url=1WOqCxPTKws4zDIQBODQ-TcXaTHfb1wDAa4TjHWqiF07loWEdV9Pzu0EQu8cmxrR4oeWYBLxUm8901_Zgyj-liiEPdI18dSZnjlA4PE8QBhOkjcdWuMD33e0vyjREn2P'}]}
Tool Execution: search_baidu
        Args: {'query': '牛津大学 成立年份', 'max_results': 5}
        Result: {'results': [{'result_id': 1, 'title': '牛津大学| University of Oxford', 'description': '', 'url': 'http://www.baidu.com/link?url=DSIzjI8yPxo1rNPQb_4e-QkkjnGcBFuR-TN0nazP7EjRRrwwl-9fJ1T2MciOA7qg'}, {'result_id': 2, 'title': 'History of the Estate | University of Oxford', 'description': '', 'url': 'http://www.baidu.com/link?url=bJaIrT-nKDWMwWB8gJtzjMiIE3AH3DpaIISOkpwevE2UkICOOJEwED5JWpIk_hVec6rwZ-mnmfQ8YTvQvDP8ObRkh_vJPHTRMN04RJRJ3ES'}]}
从百度搜索的结果来看，牛津大学的具体成立年份并没有直接显示在搜索摘要中。不过，我们可以访问其中的一个链接来获取更详细的信息。根据我所掌握的知识，牛津大学的成立年份大约是在1096年至1167年之间。为了任务的准确性，我将采用1167年作为牛津大学的成立年份。接下来，我将计算牛津大学在2024年的年龄，然后再加上10年。

计算牛津大学在2024年的年龄：
2024 - 1167 = 857 年

接着，我们将这个年龄加上10年：
857 + 10 = 867 年

因此，如果将牛津大学的年龄加10年，那么到2024年时，它将会是867岁。

任务完成。如果有其他请求，请告诉我。 Next request.
AI用户:

<CAMEL_TASK_DONE>
```

