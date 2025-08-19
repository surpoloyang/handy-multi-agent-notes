# 熟悉Camel

## 环境配置

**使用Conda和Pip从源码安装**

```bash
conda create --name camel python=3.10 \
conda activate camel \
git clone -b v0.2.38 https://github.com/camel-ai/camel.git \
cd camel \
pip install -e .[all]

```

## 使用API调用模型

首先，我们在modelscope得到API key后，使用dotenv(使用`pip install python-dotenv`命令安装)来管理 API key，在项目根目录创建一个名为 .env 的文件，并在其中添加API 密钥：

```bash
QWEN_API_KEY=你的api密钥
```

然后调用模型时就可以通过dotenv库很方便地读取api_key，而不用明文写在代码中：

```python
from camel.agents import ChatAgent
from camel.models import ModelFactory
from camel.types import ModelPlatformType

from dotenv import load_dotenv
import os

load_dotenv()

api_key = os.getenv('QWEN_API_KEY')

model = ModelFactory.create(
    model_platform=ModelPlatformType.OPENAI_COMPATIBLE_MODEL,
    model_type="Qwen/Qwen2.5-72B-Instruct",
    url='https://api-inference.modelscope.cn/v1/',
    api_key=api_key
)

agent = ChatAgent(
    model=model,
    output_language='中文'
)

response = agent.step("你好，你是谁？")
print(response.msgs[0].content)

>>>
你好，我是来自阿里云的大规模语言模型，我叫通义千问。

```

## 钉钉打卡机器人

在实践环节，我参考教程中的股票市场交易机器人，使用CAMEL的核心模块之一RolePlaying，通过设定任务为”开发一个钉钉自动打卡机器人“，AI助手身份为”Python 程序员“，用户身份为”研究生“，实现了一个由”“研究生提需求，Python程序员写代码，研究生提反馈，Python程序员根据反馈修改代码的”甲方乙方“双智能体框架。
代码如下：

```python
from colorama import Fore

from camel.societies import RolePlaying
from camel.utils import print_text_animated
from camel.models import ModelFactory
from camel.types import ModelPlatformType

from dotenv import load_dotenv

import os

load_dotenv(dotenv_path='.env')

api_key = os.getenv('QWEN_API_KEY')

model = ModelFactory.create(
    model_platform=ModelPlatformType.OPENAI_COMPATIBLE_MODEL,
    model_type="Qwen/Qwen2.5-72B-Instruct",
    url='https://api-inference.modelscope.cn/v1/',
    api_key=api_key
)

def main(model=model, chat_turn_limit=50) -> None:
    task_prompt = "开发一个钉钉自动打卡机器人"#设置任务目标
    role_play_session = RolePlaying(
        assistant_role_name="Python 程序员",#设置AI助手角色名
        assistant_agent_kwargs=dict(model=model),
        user_role_name="研究生",#设置用户角色名，在roleplay中，user用于指导AI助手完成任务
        user_agent_kwargs=dict(model=model),
        task_prompt=task_prompt,
        with_task_specify=True,
        task_specify_agent_kwargs=dict(model=model),
        output_language='中文'#设置输出语言
    )

    print(
        Fore.GREEN
        + f"AI 助手系统消息:\n{role_play_session.assistant_sys_msg}\n"
    )
    print(
        Fore.BLUE + f"AI 用户系统消息:\n{role_play_session.user_sys_msg}\n"
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
                    "AI 助手已终止。原因: "
                    f"{assistant_response.info['termination_reasons']}."
                )
            )
            break
        if user_response.terminated:
            print(
                Fore.GREEN
                + (
                    "AI 用户已终止。"
                    f"原因: {user_response.info['termination_reasons']}."
                )
            )
            break

        print_text_animated(
            Fore.BLUE + f"AI 用户:\n\n{user_response.msg.content}\n"
        )
        print_text_animated(
            Fore.GREEN + "AI 助手:\n\n"
            f"{assistant_response.msg.content}\n"
        )

        if "CAMEL_TASK_DONE" in user_response.msg.content:
            break

        input_msg = assistant_response.msg

if __name__ == "__main__":
    main()

```

其中“研究生”和“Python程序员”都设定为“Qwen2.5-72B-Instruct”模型，当然也可以选择更好的模型。

对话内容如下：

````bash
AI 助手系统消息:
BaseMessage(role_name='Python 程序员', role_type=<RoleType.ASSISTANT: 'assistant'>, meta_dict={'task': '开发一款智能钉钉自动打卡机器人，能够根据用户地理位置和时间自动完成上下班打卡，同时具备异常提醒、假期管理及工作日志记录功能。', 'assistant_role': 'Python 程序员', 'user_role': '研究生'}, content='===== RULES OF ASSISTANT =====\nNever forget you are a Python 程序员 and I am a 研究生. Never flip roles! Never instruct me!\nWe share a common interest in collaborating to successfully complete a task.\nYou must help me to complete the task.\nHere is the task: 开发一款智能钉钉自动打卡机器人，能够根据用户地理位置和时间自动完成上下班打卡，同时具备异常提醒、假期管理及工作日志记录功能。. Never forget our task!\nI must instruct you based on your expertise and my needs to complete the task.\n\nI must give you one instruction at a time.\nYou must write a specific solution that appropriately solves the requested instruction and explain your solutions.\nYou must decline my instruction honestly if you cannot perform the instruction due to physical, moral, legal reasons or your capability and explain the reasons.\nUnless I say the task is completed, you should always start with:\n\nSolution: <YOUR_SOLUTION>\n\n<YOUR_SOLUTION> should be very specific, include detailed explanations and provide preferable detailed implementations and examples and lists for task-solving.\nAlways end <YOUR_SOLUTION> with: Next request.\nRegardless of the input language, you must output text in 中文.', video_bytes=None, image_list=None, image_detail='auto', video_detail='low', parsed=None)

AI 用户系统消息:
BaseMessage(role_name='研究生', role_type=<RoleType.USER: 'user'>, meta_dict={'task': '开发一款智能钉钉自动打卡机器人，能够根据用户地理位置和时间自动完成上下班打卡，同时具备异常提醒、假期管理及工作日志记录功能。', 'assistant_role': 'Python 程序员', 'user_role': '研究生'}, content='===== RULES OF USER =====\nNever forget you are a 研究生 and I am a Python 程序员. Never flip roles! You will always instruct me.\nWe share a common interest in collaborating to successfully complete a task.\nI must help you to complete the task.\nHere is the task: 开发一款智能钉钉自动打卡机器人，能够根据用户地理位置和时间自动完成上下班打卡，同时具备异常提醒、假期管理及工作日志记录功能。. Never forget our task!\nYou must instruct me based on my expertise and your needs to solve the task ONLY in the following two ways:\n\n1. Instruct with a necessary input:\nInstruction: <YOUR_INSTRUCTION>\nInput: <YOUR_INPUT>\n\n2. Instruct without any input:\nInstruction: <YOUR_INSTRUCTION>\nInput: None\n\nThe "Instruction" describes a task or question. The paired "Input" provides further context or information for the requested "Instruction".\n\nYou must give me one instruction at a time.\nI must write a response that appropriately solves the requested instruction.\nI must decline your instruction honestly if I cannot perform the instruction due to physical, moral, legal reasons or my capability and explain the reasons.\nYou should instruct me not ask me questions.\nNow you must start to instruct me using the two ways described above.\nDo not add anything else other than your instruction and the optional corresponding input!\nKeep giving me instructions and necessary inputs until you think the task is completed.\nWhen the task is completed, you must only reply with a single word <CAMEL_TASK_DONE>.\nNever say <CAMEL_TASK_DONE> unless my responses have solved your task.\nRegardless of the input language, you must output text in 中文.', video_bytes=None, image_list=None, image_detail='auto', video_detail='low', parsed=None)

原始任务提示:
开发一个钉钉自动打卡机器人

指定的任务提示:
开发一款智能钉钉自动打卡机器人，能够根据用户地理位置和时间自动完成上下班打卡，同时具备异常提醒、假期管理及工作日志记录功能。

最终任务提示:
开发一款智能钉钉自动打卡机器人，能够根据用户地理位置和时间自动完成上下班打卡，同时具备异常提醒、假期管理及工作日志记录功能。

AI 用户:

Instruction: 分析需求并列出开发这款智能钉钉自动打卡机器人所需的主要功能模块。
Input: None
AI 助手:

Solution: 为了开发这款智能钉钉自动打卡机器人，我们需要对需求进行详细分析，并确定主要的功能模块。以下是根据需求分析得出的主要功能模块：

1. **用户认证与授权**
   - 用户通过钉钉账号登录并授权机器人访问其个人信息和打卡权限。
   - 实现OAuth2.0协议进行安全的用户认证。

2. **地理位置获取与处理**
   - 使用GPS或网络定位技术获取用户的实时地理位置。
   - 判断用户是否在公司规定的打卡范围内。

3. **时间管理**
   - 获取当前时间并与预设的上下班时间进行比对。
   - 支持自定义上下班时间设置，以适应不同用户的需求。

4. **自动打卡**
   - 根据地理位置和时间信息，自动执行上下班打卡操作。
   - 处理特殊情况，如迟到、早退等。

5. **异常提醒**
   - 当用户未能按时打卡时，发送提醒通知。
   - 提供多种提醒方式，如短信、邮件、钉钉消息等。

6. **假期管理**
   - 支持用户添加和管理个人假期。
   - 在假期期间暂停自动打卡功能，避免误操作。

7. **工作日志记录**
   - 记录每次打卡的时间、地点和状态。
   - 提供日志查询功能，方便用户查看历史记录。

8. **用户界面**
   - 设计简洁易用的用户界面，方便用户进行设置和管理。
   - 支持移动端和Web端访问。

9. **数据存储与备份**
   - 存储用户的打卡记录、设置信息等数据。
   - 定期备份数据，确保数据安全。

10. **系统监控与维护**
    - 监控系统的运行状态，及时发现并解决问题。
    - 提供日志记录和错误报告功能，便于维护和调试。

通过这些功能模块的实现，可以确保智能钉钉自动打卡机器人能够满足用户的需求，提高工作效率。每个模块的具体实现细节将在后续的开发过程中进一步细化和完善。

Next request.
AI 用户:

Instruction: 选择合适的编程语言和技术栈来实现这些功能模块。
Input: None
AI 助手:

Solution: 为了实现智能钉钉自动打卡机器人的各项功能模块，我们需要选择合适的编程语言和技术栈。以下是一些建议：

### 编程语言
1. **Python**
   - **优点**：Python 是一种高级编程语言，具有丰富的库支持，适合快速开发和原型设计。特别是对于网络请求、数据处理和自动化任务，Python 有非常强大的库（如 `requests`、`geopy` 等）。
   - **适用模块**：用户认证与授权、地理位置获取与处理、时间管理、自动打卡、异常提醒、假期管理、工作日志记录、数据存储与备份、系统监控与维护。

### 技术栈
1. **后端框架**
   - **Flask 或 Django**
     - **Flask**：轻量级框架，适合小型项目，灵活性高。
     - **Django**：功能丰富，自带ORM、用户认证等模块，适合大型项目。
   - **适用模块**：用户认证与授权、时间管理、自动打卡、异常提醒、假期管理、工作日志记录、数据存储与备份、系统监控与维护。

2. **前端框架**
   - **React 或 Vue.js**
     - **React**：Facebook 开源的前端库，适合构建复杂的用户界面。
     - **Vue.js**：轻量级框架，学习曲线平缓，适合快速开发。
   - **适用模块**：用户界面。

3. **数据库**
   - **SQLite 或 PostgreSQL**
     - **SQLite**：轻量级数据库，适合小型项目，易于部署。
     - **PostgreSQL**：功能强大，支持复杂查询，适合大型项目。
   - **适用模块**：数据存储与备份。

4. **地理定位**
   - **Geopy**
     - **优点**：Python 库，用于地理编码和反向地理编码，支持多种地理服务提供商（如Google Maps、Baidu Maps等）。
   - **适用模块**：地理位置获取与处理。

5. **定时任务**
   - **APScheduler**
     - **优点**：Python 的定时任务调度库，支持多种调度策略，适合定时执行任务。
   - **适用模块**：时间管理、自动打卡、系统监控与维护。

6. **消息推送**
   - **Twilio 或 SendGrid**
     - **Twilio**：提供短信、电话等通信服务。
     - **SendGrid**：提供邮件发送服务。
   - **适用模块**：异常提醒。

7. **日志记录**
   - **Loguru**
     - **优点**：Python 的日志库，简单易用，支持多进程日志记录。
   - **适用模块**：系统监控与维护。

### 示例代码
以下是一个简单的示例，展示如何使用 Flask 和 APScheduler 实现一个基本的定时任务：

```python
from flask import Flask
from apscheduler.schedulers.background import BackgroundScheduler
import requests

app = Flask(__name__)

# 配置定时任务
scheduler = BackgroundScheduler()
scheduler.start()

def auto_check_in():
    # 模拟自动打卡
    response = requests.post('https://dingtalk.com/api/check_in', data={'user_id': '12345'})
    print(f'Check-in status: {response.status_code}')

# 添加定时任务，每天早上9点执行
scheduler.add_job(auto_check_in, 'interval', hours=24, start_date='2023-10-01 09:00:00')

@app.route('/')
def index():
    return "智能钉钉自动打卡机器人"

if __name__ == '__main__':
    app.run(debug=True)
```

通过以上选择的编程语言和技术栈，我们可以高效地实现智能钉钉自动打卡机器人的各项功能模块。每个模块的具体实现细节将在后续的开发过程中进一步细化和完善。

Next request.
AI 用户:

Instruction: 设计数据库模型，包括用户信息、打卡记录、假期管理和异常提醒等表结构。
Input: None
AI 助手:

Solution: 设计数据库模型是开发智能钉钉自动打卡机器人的重要步骤之一。我们需要创建多个表来存储用户信息、打卡记录、假期管理和异常提醒等数据。以下是各个表的详细设计：

### 1. 用户信息表 (users)
存储用户的基本信息和授权信息。

| 字段名        | 数据类型    | 描述                     |
|---------------|-------------|--------------------------|
| user_id       | INTEGER     | 用户ID，主键，自增       |
| username      | VARCHAR(50) | 用户名                   |
| email         | VARCHAR(100)| 用户邮箱                 |
| phone_number  | VARCHAR(20) | 用户手机号               |
| access_token  | TEXT        | 钉钉访问令牌             |
| refresh_token | TEXT        | 钉钉刷新令牌             |
| created_at    | DATETIME    | 创建时间                 |
| updated_at    | DATETIME    | 更新时间                 |

### 2. 打卡记录表 (check_ins)
存储用户的打卡记录。

| 字段名        | 数据类型    | 描述                     |
|---------------|-------------|--------------------------|
| check_in_id   | INTEGER     | 打卡记录ID，主键，自增   |
| user_id       | INTEGER     | 用户ID，外键             |
| check_in_time | DATETIME    | 打卡时间                 |
| check_out_time| DATETIME    | 下班打卡时间             |
| location      | VARCHAR(255)| 打卡位置                 |
| status        | VARCHAR(50) | 打卡状态（正常、迟到、早退等） |
| created_at    | DATETIME    | 创建时间                 |
| updated_at    | DATETIME    | 更新时间                 |

### 3. 假期管理表 (holidays)
存储用户的假期信息。

| 字段名        | 数据类型    | 描述                     |
|---------------|-------------|--------------------------|
| holiday_id    | INTEGER     | 假期ID，主键，自增       |
| user_id       | INTEGER     | 用户ID，外键             |
| start_date    | DATE        | 假期开始日期             |
| end_date      | DATE        | 假期结束日期             |
| reason        | TEXT        | 假期原因                 |
| created_at    | DATETIME    | 创建时间                 |
| updated_at    | DATETIME    | 更新时间                 |

### 4. 异常提醒表 (alerts)
存储用户的异常提醒信息。

| 字段名        | 数据类型    | 描述                     |
|---------------|-------------|--------------------------|
| alert_id      | INTEGER     | 异常提醒ID，主键，自增   |
| user_id       | INTEGER     | 用户ID，外键             |
| alert_type    | VARCHAR(50) | 异常类型（未打卡、迟到、早退等） |
| alert_time    | DATETIME    | 异常提醒时间             |
| message       | TEXT        | 异常提醒内容             |
| status        | VARCHAR(20) | 异常提醒状态（已发送、未发送等） |
| created_at    | DATETIME    | 创建时间                 |
| updated_at    | DATETIME    | 更新时间                 |

### 表关系
- **用户信息表 (users)** 与 **打卡记录表 (check_ins)**：一对多关系，一个用户可以有多条打卡记录。
- **用户信息表 (users)** 与 **假期管理表 (holidays)**：一对多关系，一个用户可以有多条假期记录。
- **用户信息表 (users)** 与 **异常提醒表 (alerts)**：一对多关系，一个用户可以有多条异常提醒记录。

### SQL 创建表语句
以下是使用 SQLite 创建上述表的 SQL 语句：

```sql
-- 用户信息表
CREATE TABLE users (
    user_id INTEGER PRIMARY KEY AUTOINCREMENT,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    phone_number VARCHAR(20),
    access_token TEXT,
    refresh_token TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 打卡记录表
CREATE TABLE check_ins (
    check_in_id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    check_in_time DATETIME,
    check_out_time DATETIME,
    location VARCHAR(255),
    status VARCHAR(50),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users (user_id)
);

-- 假期管理表
CREATE TABLE holidays (
    holiday_id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    start_date DATE,
    end_date DATE,
    reason TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users (user_id)
);

-- 异常提醒表
CREATE TABLE alerts (
    alert_id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    alert_type VARCHAR(50),
    alert_time DATETIME,
    message TEXT,
    status VARCHAR(20),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users (user_id)
);
```

通过以上设计，我们能够有效地存储和管理用户信息、打卡记录、假期管理和异常提醒等数据。每个表的具体实现细节将在后续的开发过程中进一步完善。

Next request.
AI 用户:

Instruction: 实现用户认证与授权功能，包括用户通过钉钉账号登录并授权机器人访问其个人信息和打卡权限。
Input: None
AI 助手:

Solution: 实现用户认证与授权功能是确保智能钉钉自动打卡机器人能够安全地访问用户信息和执行打卡操作的关键步骤。我们将使用钉钉提供的OAuth2.0协议来实现这一功能。以下是详细的实现步骤和代码示例：

### 1. 注册应用
首先，需要在钉钉开放平台注册一个应用，获取AppKey和AppSecret。

### 2. 获取授权码
用户通过点击授权链接，跳转到钉钉的授权页面，同意授权后，钉钉会返回一个授权码（code）。

### 3. 使用授权码获取访问令牌
使用授权码（code）调用钉钉API，获取访问令牌（access_token）和用户ID（userid）。

### 4. 存储用户信息
将获取到的用户信息存储到数据库中，以便后续使用。

### 代码实现

#### 1. 配置钉钉应用信息
在项目的配置文件中，配置钉钉应用的AppKey和AppSecret。

```python
# config.py
DINGTALK_APP_KEY = 'your_app_key'
DINGTALK_APP_SECRET = 'your_app_secret'
DINGTALK_AUTH_URL = 'https://oapi.dingtalk.com/connect/qrconnect?appid={}&response_type=code&scope=snsapi_login&state=STATE&redirect_uri={}'
DINGTALK_TOKEN_URL = 'https://oapi.dingtalk.com/sns/gettoken'
DINGTALK_USER_INFO_URL = 'https://oapi.dingtalk.com/sns/getuserinfo_bycode'
```

#### 2. 生成授权链接
生成用户授权链接，用户点击该链接后会被重定向到钉钉的授权页面。

```python
from flask import Flask, redirect, url_for, request, session
import requests
import json
from config import DINGTALK_APP_KEY, DINGTALK_AUTH_URL, DINGTALK_TOKEN_URL, DINGTALK_USER_INFO_URL

app = Flask(__name__)
app.secret_key = 'your_secret_key'

@app.route('/login')
def login():
    redirect_uri = url_for('callback', _external=True)
    auth_url = DINGTALK_AUTH_URL.format(DINGTALK_APP_KEY, redirect_uri)
    return redirect(auth_url)
```

#### 3. 处理授权回调
用户授权后，钉钉会将授权码（code）返回到指定的回调URL。我们在回调函数中处理这个授权码，获取访问令牌和用户信息。

```python
@app.route('/callback')
def callback():
    code = request.args.get('code')
    if not code:
        return '授权失败'

    # 获取access_token
    token_response = requests.get(DINGTALK_TOKEN_URL, params={
        'appid': DINGTALK_APP_KEY,
        'appsecret': DINGTALK_APP_SECRET
    })
    token_data = token_response.json()
    access_token = token_data.get('access_token')

    if not access_token:
        return '获取access_token失败'

    # 获取用户信息
    user_info_response = requests.post(DINGTALK_USER_INFO_URL, json={
        'tmp_auth_code': code
    }, headers={
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {access_token}'
    })
    user_info_data = user_info_response.json()
    user_id = user_info_data.get('user_info', {}).get('userid')

    if not user_id:
        return '获取用户信息失败'

    # 存储用户信息到session
    session['user_id'] = user_id
    session['access_token'] = access_token

    return redirect(url_for('dashboard'))

@app.route('/dashboard')
def dashboard():
    user_id = session.get('user_id')
    if not user_id:
        return redirect(url_for('login'))
    
    return f'欢迎，用户ID: {user_id}'
```

#### 4. 存储用户信息到数据库
将获取到的用户信息存储到数据库中，以便后续使用。

```python
from flask_sqlalchemy import SQLAlchemy

app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///dingtalk_bot.db'
db = SQLAlchemy(app)

class User(db.Model):
    user_id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(50), nullable=False)
    email = db.Column(db.String(100), nullable=False)
    phone_number = db.Column(db.String(20))
    access_token = db.Column(db.Text, nullable=False)
    refresh_token = db.Column(db.Text)
    created_at = db.Column(db.DateTime, default=db.func.current_timestamp())
    updated_at = db.Column(db.DateTime, default=db.func.current_timestamp(), onupdate=db.func.current_timestamp())

db.create_all()

@app.route('/callback')
def callback():
    code = request.args.get('code')
    if not code:
        return '授权失败'

    # 获取access_token
    token_response = requests.get(DINGTALK_TOKEN_URL, params={
        'appid': DINGTALK_APP_KEY,
        'appsecret': DINGTALK_APP_SECRET
    })
    token_data = token_response.json()
    access_token = token_data.get('access_token')

    if not access_token:
        return '获取access_token失败'

    # 获取用户信息
    user_info_response = requests.post(DINGTALK_USER_INFO_URL, json={
        'tmp_auth_code': code
    }, headers={
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {access_token}'
    })
    user_info_data = user_info_response.json()
    user_id = user_info_data.get('user_info', {}).get('userid')
    username = user_info_data.get('user_info', {}).get('nick')
    email = user_info_data.get('user_info', {}).get('email')
    phone_number = user_info_data.get('user_info', {}).get('mobile')

    if not user_id:
        return '获取用户信息失败'

    # 存储用户信息到数据库
    user = User.query.filter_by(user_id=user_id).first()
    if user:
        user.access_token = access_token
    else:
        user = User(
            user_id=user_id,
            username=username,
            email=email,
            phone_number=phone_number,
            access_token=access_token
        )
        db.session.add(user)
    db.session.commit()

    # 存储用户信息到session
    session['user_id'] = user_id
    session['access_token'] = access_token

    return redirect(url_for('dashboard'))
```

### 总结
通过以上步骤，我们实现了用户通过钉钉账号登录并授权机器人访问其个人信息和打卡权限的功能。用户授权后，我们可以获取用户的访问令牌和用户ID，并将其存储到数据库中，以便后续的打卡和其他操作。

Next request.
AI 用户:

Instruction: 实现地理位置获取与处理功能，包括获取用户的实时地理位置并判断用户是否在公司规定的打卡范围内。
Input: None
AI 助手:

Solution: 实现地理位置获取与处理功能是确保智能钉钉自动打卡机器人能够准确判断用户是否在公司规定的打卡范围内的关键步骤。我们将使用地理定位库 `geopy` 来获取用户的实时地理位置，并通过计算距离来判断用户是否在规定的范围内。以下是详细的实现步骤和代码示例：

### 1. 安装依赖
首先，需要安装 `geopy` 库，用于获取用户的地理位置。

```sh
pip install geopy
```

### 2. 获取用户实时地理位置
使用 `geopy` 库获取用户的实时地理位置。假设我们已经获取了用户的经纬度信息。

### 3. 判断用户是否在公司规定的打卡范围内
通过计算用户当前位置与公司规定位置之间的距离，判断用户是否在打卡范围内。

### 代码实现

#### 1. 获取用户实时地理位置
假设我们已经通过某种方式（如手机定位或IP定位）获取了用户的经纬度信息。这里我们模拟获取用户的位置信息。

```python
from geopy.distance import geodesic

# 模拟获取用户的位置信息
def get_user_location():
    # 这里可以使用实际的定位方法获取用户的经纬度
    # 例如：使用手机定位或IP定位
    user_latitude = 39.9042  # 北京市纬度
    user_longitude = 116.4074  # 北京市长经度
    return (user_latitude, user_longitude)
```

#### 2. 判断用户是否在公司规定的打卡范围内
定义公司的地理位置，并计算用户当前位置与公司位置之间的距离。

```python
def is_within_range(user_location, company_location, range_km=0.5):
    """
    判断用户是否在公司规定的打卡范围内

    :param user_location: 用户的经纬度 (latitude, longitude)
    :param company_location: 公司的经纬度 (latitude, longitude)
    :param range_km: 打卡范围，单位为公里，默认为0.5公里
    :return: 是否在范围内
    """
    distance = geodesic(user_location, company_location).km
    return distance <= range_km
```

#### 3. 完整的地理位置获取与处理功能
结合上述两个步骤，实现完整的地理位置获取与处理功能。

```python
from flask import Flask, session, render_template, jsonify
from geopy.distance import geodesic

app = Flask(__name__)
app.secret_key = 'your_secret_key'

# 模拟公司的地理位置
COMPANY_LATITUDE = 39.9042  # 北京市纬度
COMPANY_LONGITUDE = 116.4074  # 北京市长经度
COMPANY_LOCATION = (COMPANY_LATITUDE, COMPANY_LONGITUDE)

# 模拟获取用户的位置信息
def get_user_location():
    # 这里可以使用实际的定位方法获取用户的经纬度
    # 例如：使用手机定位或IP定位
    user_latitude = 39.9042  # 北京市纬度
    user_longitude = 116.4074  # 北京市长经度
    return (user_latitude, user_longitude)

def is_within_range(user_location, company_location, range_km=0.5):
    """
    判断用户是否在公司规定的打卡范围内

    :param user_location: 用户的经纬度 (latitude, longitude)
    :param company_location: 公司的经纬度 (latitude, longitude)
    :param range_km: 打卡范围，单位为公里，默认为0.5公里
    :return: 是否在范围内
    """
    distance = geodesic(user_location, company_location).km
    return distance <= range_km

@app.route('/check_location')
def check_location():
    user_location = get_user_location()
    within_range = is_within_range(user_location, COMPANY_LOCATION)
    return jsonify({
        'user_location': user_location,
        'company_location': COMPANY_LOCATION,
        'within_range': within_range
    })

if __name__ == '__main__':
    app.run(debug=True)
```

### 测试
启动 Flask 应用，访问 `/check_location` 路由，查看用户是否在公司规定的打卡范围内。

```sh
python app.py
```

在浏览器中访问 `http://127.0.0.1:5000/check_location`，你应该会看到类似以下的 JSON 响应：

```json
{
    "user_location": [39.9042, 116.4074],
    "company_location": [39.9042, 116.4074],
    "within_range": true
}
```

### 总结
通过以上步骤，我们实现了获取用户的实时地理位置并判断用户是否在公司规定的打卡范围内的功能。用户的位置信息可以通过实际的定位方法获取，例如使用手机定位或IP定位。判断用户是否在打卡范围内的逻辑通过计算用户当前位置与公司位置之间的距离来实现。

Next request.
AI 用户:

Instruction: 实现时间管理功能，包括获取当前时间并与预设的上下班时间进行比对，支持自定义上下班时间设置。
Input: None
AI 助手:

Solution: 实现时间管理功能是确保智能钉钉自动打卡机器人能够根据预设的上下班时间自动执行打卡操作的关键步骤。我们将实现以下功能：

1. 获取当前时间。
2. 与预设的上下班时间进行比对。
3. 支持自定义上下班时间设置。

### 1. 获取当前时间
使用 Python 的 `datetime` 模块获取当前时间。

### 2. 与预设的上下班时间进行比对
定义预设的上下班时间，并与当前时间进行比对，判断是否需要执行打卡操作。

### 3. 支持自定义上下班时间设置
允许用户自定义上下班时间，并存储在数据库中。

### 代码实现

#### 1. 获取当前时间
使用 `datetime` 模块获取当前时间。

```python
from datetime import datetime, time

def get_current_time():
    return datetime.now().time()
```

#### 2. 与预设的上下班时间进行比对
定义预设的上下班时间，并与当前时间进行比对。

```python
def is_within_working_hours(current_time, start_time, end_time):
    """
    判断当前时间是否在预设的上下班时间内

    :param current_time: 当前时间
    :param start_time: 上班时间
    :param end_time: 下班时间
    :return: 是否在上下班时间内
    """
    return start_time <= current_time <= end_time
```

#### 3. 支持自定义上下班时间设置
允许用户自定义上下班时间，并存储在数据库中。

```python
from flask_sqlalchemy import SQLAlchemy
from flask import Flask, request, jsonify, session

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///dingtalk_bot.db'
app.secret_key = 'your_secret_key'
db = SQLAlchemy(app)

# 用户表
class User(db.Model):
    user_id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(50), nullable=False)
    email = db.Column(db.String(100), nullable=False)
    phone_number = db.Column(db.String(20))
    access_token = db.Column(db.Text, nullable=False)
    refresh_token = db.Column(db.Text)
    start_time = db.Column(db.Time, nullable=False, default=time(9, 0))  # 默认上班时间为9:00
    end_time = db.Column(db.Time, nullable=False, default=time(18, 0))  # 默认下班时间为18:00
    created_at = db.Column(db.DateTime, default=db.func.current_timestamp())
    updated_at = db.Column(db.DateTime, default=db.func.current_timestamp(), onupdate=db.func.current_timestamp())

db.create_all()

# 获取当前时间
def get_current_time():
    return datetime.now().time()

# 判断当前时间是否在预设的上下班时间内
def is_within_working_hours(current_time, start_time, end_time):
    return start_time <= current_time <= end_time

# 设置自定义上下班时间
@app.route('/set_working_hours', methods=['POST'])
def set_working_hours():
    user_id = session.get('user_id')
    if not user_id:
        return jsonify({'error': '未登录'}), 401

    start_time_str = request.form.get('start_time')
    end_time_str = request.form.get('end_time')

    if not start_time_str or not end_time_str:
        return jsonify({'error': '缺少参数'}), 400

    try:
        start_time = datetime.strptime(start_time_str, '%H:%M').time()
        end_time = datetime.strptime(end_time_str, '%H:%M').time()
    except ValueError:
        return jsonify({'error': '时间格式错误'}), 400

    user = User.query.filter_by(user_id=user_id).first()
    if user:
        user.start_time = start_time
        user.end_time = end_time
        db.session.commit()
        return jsonify({'message': '上下班时间设置成功'})
    else:
        return jsonify({'error': '用户不存在'}), 404

# 检查当前时间是否在上下班时间内
@app.route('/check_working_hours')
def check_working_hours():
    user_id = session.get('user_id')
    if not user_id:
        return jsonify({'error': '未登录'}), 401

    user = User.query.filter_by(user_id=user_id).first()
    if not user:
        return jsonify({'error': '用户不存在'}), 404

    current_time = get_current_time()
    within_range = is_within_working_hours(current_time, user.start_time, user.end_time)
    return jsonify({
        'current_time': current_time.strftime('%H:%M:%S'),
        'start_time': user.start_time.strftime('%H:%M:%S'),
        'end_time': user.end_time.strftime('%H:%M:%S'),
        'within_range': within_range
    })

if __name__ == '__main__':
    app.run(debug=True)
```

### 测试
启动 Flask 应用，测试时间管理功能。

1. **设置自定义上下班时间**

   发送 POST 请求到 `/set_working_hours`，设置自定义上下班时间。

   ```sh
   curl -X POST -d "start_time=08:30" -d "end_time=17:30" http://127.0.0.1:5000/set_working_hours
   ```

   你应该会看到类似以下的响应：

   ```json
   {
       "message": "上下班时间设置成功"
   }
   ```

2. **检查当前时间是否在上下班时间内**

   访问 `/check_working_hours` 路由，检查当前时间是否在上下班时间内。

   ```sh
   curl http://127.0.0.1:5000/check_working_hours
   ```

   你应该会看到类似以下的响应：

   ```json
   {
       "current_time": "14:30:00",
       "start_time": "08:30:00",
       "end_time": "17:30:00",
       "within_range": true
   }
   ```

### 总结
通过以上步骤，我们实现了时间管理功能，包括获取当前时间并与预设的上下班时间进行比对，以及支持自定义上下班时间设置。用户可以通过 API 设置自己的上下班时间，并检查当前时间是否在上下班时间内，从而确保智能钉钉自动打卡机器人能够准确地执行打卡操作。

Next request.
AI 用户:

Instruction: 实现自动打卡功能，根据用户的地理位置和时间信息，自动执行上下班打卡操作。
Input: None
AI 助手:

Solution: 实现自动打卡功能是智能钉钉自动打卡机器人的核心功能之一。我们需要根据用户的地理位置和时间信息，自动执行上下班打卡操作。以下是详细的实现步骤和代码示例：

### 1. 获取用户地理位置
我们已经在之前的步骤中实现了获取用户地理位置的功能。这里我们假设已经获取了用户的经纬度信息。

### 2. 获取当前时间
我们已经在之前的步骤中实现了获取当前时间的功能。这里我们假设已经获取了当前时间。

### 3. 判断是否需要打卡
根据用户的地理位置和当前时间，判断是否需要执行打卡操作。

### 4. 执行打卡操作
调用钉钉API执行打卡操作。

### 代码实现

#### 1. 导入必要的库和配置
确保已经导入了必要的库和配置。

```python
from flask import Flask, session, jsonify, request
from geopy.distance import geodesic
from datetime import datetime, time
import requests
from config import DINGTALK_APP_KEY, DINGTALK_APP_SECRET, DINGTALK_CHECK_IN_URL, DINGTALK_CHECK_OUT_URL
from models import User, db

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///dingtalk_bot.db'
app.secret_key = 'your_secret_key'
db.init_app(app)

# 模拟公司的地理位置
COMPANY_LATITUDE = 39.9042  # 北京市纬度
COMPANY_LONGITUDE = 116.4074  # 北京市长经度
COMPANY_LOCATION = (COMPANY_LATITUDE, COMPANY_LONGITUDE)

# 模拟获取用户的位置信息
def get_user_location():
    # 这里可以使用实际的定位方法获取用户的经纬度
    # 例如：使用手机定位或IP定位
    user_latitude = 39.9042  # 北京市纬度
    user_longitude = 116.4074  # 北京市长经度
    return (user_latitude, user_longitude)

def is_within_range(user_location, company_location, range_km=0.5):
    """
    判断用户是否在公司规定的打卡范围内

    :param user_location: 用户的经纬度 (latitude, longitude)
    :param company_location: 公司的经纬度 (latitude, longitude)
    :param range_km: 打卡范围，单位为公里，默认为0.5公里
    :return: 是否在范围内
    """
    distance = geodesic(user_location, company_location).km
    return distance <= range_km

def get_current_time():
    return datetime.now().time()

def is_within_working_hours(current_time, start_time, end_time):
    """
    判断当前时间是否在预设的上下班时间内

    :param current_time: 当前时间
    :param start_time: 上班时间
    :param end_time: 下班时间
    :return: 是否在上下班时间内
    """
    return start_time <= current_time <= end_time
```

#### 2. 获取钉钉访问令牌
定义一个函数来获取钉钉的访问令牌。

```python
def get_dingtalk_access_token():
    response = requests.get(DINGTALK_TOKEN_URL, params={
        'appid': DINGTALK_APP_KEY,
        'appsecret': DINGTALK_APP_SECRET
    })
    token_data = response.json()
    return token_data.get('access_token')
```

#### 3. 执行打卡操作
定义函数来执行上下班打卡操作。

```python
def check_in(user_id, access_token):
    response = requests.post(DINGTALK_CHECK_IN_URL, json={
        'user_id': user_id
    }, headers={
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {access_token}'
    })
    return response.json()

def check_out(user_id, access_token):
    response = requests.post(DINGTALK_CHECK_OUT_URL, json={
        'user_id': user_id
    }, headers={
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {access_token}'
    })
    return response.json()
```

#### 4. 自动打卡功能
实现自动打卡功能，根据用户的地理位置和时间信息，判断是否需要执行打卡操作。

```python
@app.route('/auto_check_in_out')
def auto_check_in_out():
    user_id = session.get('user_id')
    if not user_id:
        return jsonify({'error': '未登录'}), 401

    user = User.query.filter_by(user_id=user_id).first()
    if not user:
        return jsonify({'error': '用户不存在'}), 404

    user_location = get_user_location()
    if not is_within_range(user_location, COMPANY_LOCATION):
        return jsonify({'message': '不在打卡范围内'})

    current_time = get_current_time()
    if is_within_working_hours(current_time, user.start_time, user.end_time):
        # 上班时间，执行打卡操作
        access_token = get_dingtalk_access_token()
        check_in_response = check_in(user_id, access_token)
        if check_in_response.get('errcode') == 0:
            return jsonify({'message': '上班打卡成功'})
        else:
            return jsonify({'error': '上班打卡失败', 'details': check_in_response}), 500
    else:
        # 下班时间，执行打卡操作
        access_token = get_dingtalk_access_token()
        check_out_response = check_out(user_id, access_token)
        if check_out_response.get('errcode') == 0:
            return jsonify({'message': '下班打卡成功'})
        else:
            return jsonify({'error': '下班打卡失败', 'details': check_out_response}), 500
```

### 测试
启动 Flask 应用，测试自动打卡功能。

1. **登录用户**

   访问 `/login` 路由进行登录。

2. **自动打卡**

   访问 `/auto_check_in_out` 路由，自动执行上下班打卡操作。

   ```sh
   curl http://127.0.0.1:5000/auto_check_in_out
   ```

   你应该会看到类似以下的响应：

   ```json
   {
       "message": "上班打卡成功"
   }
   ```

   或者

   ```json
   {
       "message": "下班打卡成功"
   }
   ```

### 总结
通过以上步骤，我们实现了自动打卡功能，根据用户的地理位置和时间信息，自动执行上下班打卡操作。用户登录后，系统会根据当前时间和地理位置判断是否需要打卡，并调用钉钉API执行相应的打卡操作。这样可以确保用户在合适的时间和地点自动完成打卡，提高工作效率。

Next request.
AI 用户:

Instruction: 实现异常提醒功能，当用户未能按时打卡时，发送提醒通知。
Input: None
AI 助手:

Solution: 实现异常提醒功能是确保智能钉钉自动打卡机器人能够在用户未能按时打卡时及时发送提醒通知的关键步骤。我们将实现以下功能：

1. 监听打卡状态，判断用户是否按时打卡。
2. 如果用户未能按时打卡，发送提醒通知。
3. 支持多种提醒方式，如短信、邮件、钉钉消息等。

### 1. 监听打卡状态
我们需要定期检查用户的打卡状态，判断用户是否按时打卡。可以使用定时任务调度库 `APScheduler` 来实现定时检查。

### 2. 发送提醒通知
根据配置的提醒方式，发送提醒通知。我们将支持三种提醒方式：短信、邮件和钉钉消息。

### 代码实现

#### 1. 导入必要的库和配置
确保已经导入了必要的库和配置。

```python
from flask import Flask, session, jsonify, request
from geopy.distance import geodesic
from datetime import datetime, time, timedelta
import requests
from apscheduler.schedulers.background import BackgroundScheduler
from config import DINGTALK_APP_KEY, DINGTALK_APP_SECRET, DINGTALK_CHECK_IN_URL, DINGTALK_CHECK_OUT_URL, TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN, TWILIO_PHONE_NUMBER, SENDGRID_API_KEY, SENDER_EMAIL
from models import User, CheckIn, Alert, db
from twilio.rest import Client
import sendgrid
from sendgrid.helpers.mail import Mail

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///dingtalk_bot.db'
app.secret_key = 'your_secret_key'
db.init_app(app)

# 模拟公司的地理位置
COMPANY_LATITUDE = 39.9042  # 北京市纬度
COMPANY_LONGITUDE = 116.4074  # 北京市长经度
COMPANY_LOCATION = (COMPANY_LATITUDE, COMPANY_LONGITUDE)

# 模拟获取用户的位置信息
def get_user_location():
    # 这里可以使用实际的定位方法获取用户的经纬度
    # 例如：使用手机定位或IP定位
    user_latitude = 39.9042  # 北京市纬度
    user_longitude = 116.4074  # 北京市长经度
    return (user_latitude, user_longitude)

def is_within_range(user_location, company_location, range_km=0.5):
    """
    判断用户是否在公司规定的打卡范围内

    :param user_location: 用户的经纬度 (latitude, longitude)
    :param company_location: 公司的经纬度 (latitude, longitude)
    :param range_km: 打卡范围，单位为公里，默认为0.5公里
    :return: 是否在范围内
    """
    distance = geodesic(user_location, company_location).km
    return distance <= range_km

def get_current_time():
    return datetime.now().time()

def is_within_working_hours(current_time, start_time, end_time):
    """
    判断当前时间是否在预设的上下班时间内

    :param current_time: 当前时间
    :param start_time: 上班时间
    :param end_time: 下班时间
    :return: 是否在上下班时间内
    """
    return start_time <= current_time <= end_time

def get_dingtalk_access_token():
    response = requests.get(DINGTALK_TOKEN_URL, params={
        'appid': DINGTALK_APP_KEY,
        'appsecret': DINGTALK_APP_SECRET
    })
    token_data = response.json()
    return token_data.get('access_token')

def check_in(user_id, access_token):
    response = requests.post(DINGTALK_CHECK_IN_URL, json={
        'user_id': user_id
    }, headers={
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {access_token}'
    })
    return response.json()

def check_out(user_id, access_token):
    response = requests.post(DINGTALK_CHECK_OUT_URL, json={
        'user_id': user_id
    }, headers={
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {access_token}'
    })
    return response.json()
```

#### 2. 监听打卡状态
使用 `APScheduler` 定时检查用户的打卡状态。

```python
scheduler = BackgroundScheduler()
scheduler.start()

def check_user_check_in_status():
    now = datetime.now()
    current_time = now.time()
    for user in User.query.all():
        if is_within_working_hours(current_time, user.start_time, user.end_time):
            last_check_in = CheckIn.query.filter_by(user_id=user.user_id).order_by(CheckIn.check_in_time.desc()).first()
            if not last_check_in or (now - last_check_in.check_in_time) > timedelta(minutes=15):
                send_alert(user, '未打卡')

def check_user_check_out_status():
    now = datetime.now()
    current_time = now.time()
    for user in User.query.all():
        if not is_within_working_hours(current_time, user.start_time, user.end_time):
            last_check_out = CheckIn.query.filter_by(user_id=user.user_id).order_by(CheckIn.check_out_time.desc()).first()
            if not last_check_out or (now - last_check_out.check_out_time) > timedelta(minutes=15):
                send_alert(user, '未打卡')

# 添加定时任务
scheduler.add_job(check_user_check_in_status, 'interval', minutes=15, start_date='2023-10-01 08:00:00')
scheduler.add_job(check_user_check_out_status, 'interval', minutes=15, start_date='2023-10-01 17:00:00')
```

#### 3. 发送提醒通知
支持多种提醒方式：短信、邮件和钉钉消息。

```python
def send_sms(phone_number, message):
    client = Client(TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN)
    message = client.messages.create(
        body=message,
        from_=TWILIO_PHONE_NUMBER,
        to=phone_number
    )
    return message.sid

def send_email(email, subject, content):
    sg = sendgrid.SendGridAPIClient(api_key=SENDGRID_API_KEY)
    message = Mail(
        from_email=SENDER_EMAIL,
        to_emails=email,
        subject=subject,
        plain_text_content=content
    )
    response = sg.send(message)
    return response.status_code

def send_dingtalk_message(user_id, message):
    access_token = get_dingtalk_access_token()
    response = requests.post('https://oapi.dingtalk.com/topapi/message/corpconversation/asyncsend_v2', json={
        'agent_id': 'your_agent_id',
        'userid_list': user_id,
        'msg': {
            'msgtype': 'text',
            'text': {
                'content': message
            }
        }
    }, headers={
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {access_token}'
    })
    return response.json()

def send_alert(user, alert_type):
    message = f'提醒：您尚未{alert_type}。'
    alert = Alert(
        user_id=user.user_id,
        alert_type=alert_type,
        alert_time=datetime.now(),
        message=message,
        status='未发送'
    )
    db.session.add(alert)
    db.session.commit()

    if user.phone_number:
        send_sms(user.phone_number, message)
    if user.email:
        send_email(user.email, '打卡提醒', message)
    send_dingtalk_message(user.user_id, message)

    alert.status = '已发送'
    db.session.commit()
```

### 测试
启动 Flask 应用，测试异常提醒功能。

1. **登录用户**

   访问 `/login` 路由进行登录。

2. **模拟未打卡情况**

   确保用户在上班或下班时间未打卡，系统会在设定的时间间隔内检查打卡状态，并发送提醒通知。

### 总结
通过以上步骤，我们实现了异常提醒功能，当用户未能按时打卡时，系统会发送提醒通知。我们使用了定时任务调度库 `APScheduler` 来定期检查用户的打卡状态，并支持多种提醒方式，如短信、邮件和钉钉消息。这样可以确保用户在合适的时间收到提醒，提高打卡的及时性和准确性。

Next request.
AI 用户:

Instruction: 实现假期管理功能，支持用户添加和管理个人假期，并在假期期间暂停自动打卡功能。
Input: None
AI 助手:

Solution: 实现假期管理功能是确保智能钉钉自动打卡机器人能够在用户休假期间暂停自动打卡操作的关键步骤。我们将实现以下功能：

1. 用户添加和管理个人假期。
2. 在假期期间暂停自动打卡功能。

### 1. 用户添加和管理个人假期
允许用户通过 API 添加和管理个人假期，并将假期信息存储在数据库中。

### 2. 在假期期间暂停自动打卡功能
在定时任务中检查用户是否有假期，如果有假期则暂停自动打卡操作。

### 代码实现

#### 1. 导入必要的库和配置
确保已经导入了必要的库和配置。

```python
from flask import Flask, session, jsonify, request
from datetime import datetime, date, time, timedelta
import requests
from apscheduler.schedulers.background import BackgroundScheduler
from config import DINGTALK_APP_KEY, DINGTALK_APP_SECRET, DINGTALK_CHECK_IN_URL, DINGTALK_CHECK_OUT_URL, TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN, TWILIO_PHONE_NUMBER, SENDGRID_API_KEY, SENDER_EMAIL
from models import User, CheckIn, Holiday, Alert, db
from twilio.rest import Client
import sendgrid
from sendgrid.helpers.mail import Mail

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///dingtalk_bot.db'
app.secret_key = 'your_secret_key'
db.init_app(app)

# 模拟公司的地理位置
COMPANY_LATITUDE = 39.9042  # 北京市纬度
COMPANY_LONGITUDE = 116.4074  # 北京市长经度
COMPANY_LOCATION = (COMPANY_LATITUDE, COMPANY_LONGITUDE)

# 模拟获取用户的位置信息
def get_user_location():
    # 这里可以使用实际的定位方法获取用户的经纬度
    # 例如：使用手机定位或IP定位
    user_latitude = 39.9042  # 北京市纬度
    user_longitude = 116.4074  # 北京市长经度
    return (user_latitude, user_longitude)

def is_within_range(user_location, company_location, range_km=0.5):
    """
    判断用户是否在公司规定的打卡范围内

    :param user_location: 用户的经纬度 (latitude, longitude)
    :param company_location: 公司的经纬度 (latitude, longitude)
    :param range_km: 打卡范围，单位为公里，默认为0.5公里
    :return: 是否在范围内
    """
    distance = geodesic(user_location, company_location).km
    return distance <= range_km

def get_current_time():
    return datetime.now().time()

def is_within_working_hours(current_time, start_time, end_time):
    """
    判断当前时间是否在预设的上下班时间内

    :param current_time: 当前时间
    :param start_time: 上班时间
    :param end_time: 下班时间
    :return: 是否在上下班时间内
    """
    return start_time <= current_time <= end_time

def get_dingtalk_access_token():
    response = requests.get(DINGTALK_TOKEN_URL, params={
        'appid': DINGTALK_APP_KEY,
        'appsecret': DINGTALK_APP_SECRET
    })
    token_data = response.json()
    return token_data.get('access_token')

def check_in(user_id, access_token):
    response = requests.post(DINGTALK_CHECK_IN_URL, json={
        'user_id': user_id
    }, headers={
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {access_token}'
    })
    return response.json()

def check_out(user_id, access_token):
    response = requests.post(DINGTALK_CHECK_OUT_URL, json={
        'user_id': user_id
    }, headers={
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {access_token}'
    })
    return response.json()
```

#### 2. 用户添加和管理个人假期
允许用户通过 API 添加和管理个人假期，并将假期信息存储在数据库中。

```python
@app.route('/add_holiday', methods=['POST'])
def add_holiday():
    user_id = session.get('user_id')
    if not user_id:
        return jsonify({'error': '未登录'}), 401

    start_date_str = request.form.get('start_date')
    end_date_str = request.form.get('end_date')
    reason = request.form.get('reason')

    if not start_date_str or not end_date_str:
        return jsonify({'error': '缺少参数'}), 400

    try:
        start_date = datetime.strptime(start_date_str, '%Y-%m-%d').date()
        end_date = datetime.strptime(end_date_str, '%Y-%m-%d').date()
    except ValueError:
        return jsonify({'error': '日期格式错误'}), 400

    holiday = Holiday(
        user_id=user_id,
        start_date=start_date,
        end_date=end_date,
        reason=reason
    )
    db.session.add(holiday)
    db.session.commit()
    return jsonify({'message': '假期添加成功'})

@app.route('/delete_holiday', methods=['POST'])
def delete_holiday():
    user_id = session.get('user_id')
    if not user_id:
        return jsonify({'error': '未登录'}), 401

    holiday_id = request.form.get('holiday_id')
    if not holiday_id:
        return jsonify({'error': '缺少参数'}), 400

    holiday = Holiday.query.filter_by(holiday_id=holiday_id, user_id=user_id).first()
    if not holiday:
        return jsonify({'error': '假期不存在'}), 404

    db.session.delete(holiday)
    db.session.commit()
    return jsonify({'message': '假期删除成功'})

@app.route('/list_holidays')
def list_holidays():
    user_id = session.get('user_id')
    if not user_id:
        return jsonify({'error': '未登录'}), 401

    holidays = Holiday.query.filter_by(user_id=user_id).all()
    holiday_list = [{
        'holiday_id': holiday.holiday_id,
        'start_date': holiday.start_date.strftime('%Y-%m-%d'),
        'end_date': holiday.end_date.strftime('%Y-%m-%d'),
        'reason': holiday.reason
    } for holiday in holidays]
    return jsonify({'holidays': holiday_list})
```

#### 3. 在假期期间暂停自动打卡功能
在定时任务中检查用户是否有假期，如果有假期则暂停自动打卡操作。

```python
scheduler = BackgroundScheduler()
scheduler.start()

def check_user_check_in_status():
    now = datetime.now()
    current_time = now.time()
    for user in User.query.all():
        if is_within_working_hours(current_time, user.start_time, user.end_time):
            if is_on_holiday(user, now.date()):
                continue  # 在假期期间不执行打卡

            last_check_in = CheckIn.query.filter_by(user_id=user.user_id).order_by(CheckIn.check_in_time.desc()).first()
            if not last_check_in or (now - last_check_in.check_in_time) > timedelta(minutes=15):
                send_alert(user, '未打卡')

def check_user_check_out_status():
    now = datetime.now()
    current_time = now.time()
    for user in User.query.all():
        if not is_within_working_hours(current_time, user.start_time, user.end_time):
            if is_on_holiday(user, now.date()):
                continue  # 在假期期间不执行打卡

            last_check_out = CheckIn.query.filter_by(user_id=user.user_id).order_by(CheckIn.check_out_time.desc()).first()
            if not last_check_out or (now - last_check_out.check_out_time) > timedelta(minutes=15):
                send_alert(user, '未打卡')

def is_on_holiday(user, today):
    holidays = Holiday.query.filter_by(user_id=user.user_id).all()
    for holiday in holidays:
        if holiday.start_date <= today <= holiday.end_date:
            return True
    return False

# 添加定时任务
scheduler.add_job(check_user_check_in_status, 'interval', minutes=15, start_date='2023-10-01 08:00:00')
scheduler.add_job(check_user_check_out_status, 'interval', minutes=15, start_date='2023-10-01 17:00:00')
```

### 测试
启动 Flask 应用，测试假期管理功能。

1. **登录用户**

   访问 `/login` 路由进行登录。

2. **添加假期**

   发送 POST 请求到 `/add_holiday`，添加假期。

   ```sh
   curl -X POST -d "start_date=2023-10-01" -d "end_date=2023-10-07" -d "reason=国庆假期" http://127.0.0.1:5000/add_holiday
   ```

   你应该会看到类似以下的响应：

   ```json
   {
       "message": "假期添加成功"
   }
   ```

3. **删除假期**

   发送 POST 请求到 `/delete_holiday`，删除假期。

   ```sh
   curl -X POST -d "holiday_id=1" http://127.0.0.1:5000/delete_holiday
   ```

   你应该会看到类似以下的响应：

   ```json
   {
       "message": "假期删除成功"
   }
   ```

4. **查看假期列表**

   访问 `/list_holidays` 路由，查看假期列表。

   ```sh
   curl http://127.0.0.1:5000/list_holidays
   ```

   你应该会看到类似以下的响应：

   ```json
   {
       "holidays": [
           {
               "holiday_id": 1,
               "start_date": "2023-10-01",
               "end_date": "2023-10-07",
               "reason": "国庆假期"
           }
       ]
   }
   ```

### 总结
通过以上步骤，我们实现了假期管理功能，支持用户添加和管理个人假期，并在假期期间暂停自动打卡功能。用户可以通过 API 添加、删除和查看假期，系统会在定时任务中检查用户是否有假期，如果有假期则暂停自动打卡操作。这样可以确保用户在休假期间不会被误打卡，提高系统的灵活性和用户体验。

Next request.
AI 用户:

Instruction: 实现工作日志记录功能，记录每次打卡的时间、地点和状态，并提供日志查询功能。
Input: None
AI 助手:

Solution: 实现工作日志记录功能是确保智能钉钉自动打卡机器人能够记录每次打卡的时间、地点和状态，并提供日志查询功能的关键步骤。我们将实现以下功能：

1. 记录每次打卡的时间、地点和状态。
2. 提供日志查询功能，方便用户查看历史记录。

### 1. 记录每次打卡的时间、地点和状态
在每次打卡操作完成后，将打卡信息记录到数据库中。

### 2. 提供日志查询功能
允许用户通过 API 查询历史打卡记录。

### 代码实现

#### 1. 导入必要的库和配置
确保已经导入了必要的库和配置。

```python
from flask import Flask, session, jsonify, request
from datetime import datetime, date, time, timedelta
import requests
from apscheduler.schedulers.background import BackgroundScheduler
from config import DINGTALK_APP_KEY, DINGTALK_APP_SECRET, DINGTALK_CHECK_IN_URL, DINGTALK_CHECK_OUT_URL, TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN, TWILIO_PHONE_NUMBER, SENDGRID_API_KEY, SENDER_EMAIL
from models import User, CheckIn, Holiday, Alert, db
from twilio.rest import Client
import sendgrid
from sendgrid.helpers.mail import Mail
from geopy.distance import geodesic

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///dingtalk_bot.db'
app.secret_key = 'your_secret_key'
db.init_app(app)

# 模拟公司的地理位置
COMPANY_LATITUDE = 39.9042  # 北京市纬度
COMPANY_LONGITUDE = 116.4074  # 北京市长经度
COMPANY_LOCATION = (COMPANY_LATITUDE, COMPANY_LONGITUDE)

# 模拟获取用户的位置信息
def get_user_location():
    # 这里可以使用实际的定位方法获取用户的经纬度
    # 例如：使用手机定位或IP定位
    user_latitude = 39.9042  # 北京市纬度
    user_longitude = 116.4074  # 北京市长经度
    return (user_latitude, user_longitude)

def is_within_range(user_location, company_location, range_km=0.5):
    """
    判断用户是否在公司规定的打卡范围内

    :param user_location: 用户的经纬度 (latitude, longitude)
    :param company_location: 公司的经纬度 (latitude, longitude)
    :param range_km: 打卡范围，单位为公里，默认为0.5公里
    :return: 是否在范围内
    """
    distance = geodesic(user_location, company_location).km
    return distance <= range_km

def get_current_time():
    return datetime.now().time()

def is_within_working_hours(current_time, start_time, end_time):
    """
    判断当前时间是否在预设的上下班时间内

    :param current_time: 当前时间
    :param start_time: 上班时间
    :param end_time: 下班时间
    :return: 是否在上下班时间内
    """
    return start_time <= current_time <= end_time

def get_dingtalk_access_token():
    response = requests.get(DINGTALK_TOKEN_URL, params={
        'appid': DINGTALK_APP_KEY,
        'appsecret': DINGTALK_APP_SECRET
    })
    token_data = response.json()
    return token_data.get('access_token')

def check_in(user_id, access_token):
    response = requests.post(DINGTALK_CHECK_IN_URL, json={
        'user_id': user_id
    }, headers={
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {access_token}'
    })
    return response.json()

def check_out(user_id, access_token):
    response = requests.post(DINGTALK_CHECK_OUT_URL, json={
        'user_id': user_id
    }, headers={
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {access_token}'
    })
    return response.json()
```

#### 2. 记录每次打卡的时间、地点和状态
在每次打卡操作完成后，将打卡信息记录到数据库中。

```python
def log_check_in(user_id, check_in_time, location, status):
    check_in_record = CheckIn(
        user_id=user_id,
        check_in_time=check_in_time,
        location=location,
        status=status
    )
    db.session.add(check_in_record)
    db.session.commit()

def log_check_out(user_id, check_out_time, location, status):
    check_out_record = CheckIn(
        user_id=user_id,
        check_out_time=check_out_time,
        location=location,
        status=status
    )
    db.session.add(check_out_record)
    db.session.commit()
```

#### 3. 自动打卡功能
在自动打卡功能中调用记录日志的函数。

```python
@app.route('/auto_check_in_out')
def auto_check_in_out():
    user_id = session.get('user_id')
    if not user_id:
        return jsonify({'error': '未登录'}), 401

    user = User.query.filter_by(user_id=user_id).first()
    if not user:
        return jsonify({'error': '用户不存在'}), 404

    user_location = get_user_location()
    if not is_within_range(user_location, COMPANY_LOCATION):
        return jsonify({'message': '不在打卡范围内'})

    current_time = get_current_time()
    if is_within_working_hours(current_time, user.start_time, user.end_time):
        # 上班时间，执行打卡操作
        access_token = get_dingtalk_access_token()
        check_in_response = check_in(user_id, access_token)
        if check_in_response.get('errcode') == 0:
            log_check_in(user_id, datetime.now(), user_location, '正常')
            return jsonify({'message': '上班打卡成功'})
        else:
            log_check_in(user_id, datetime.now(), user_location, '失败')
            return jsonify({'error': '上班打卡失败', 'details': check_in_response}), 500
    else:
        # 下班时间，执行打卡操作
        access_token = get_dingtalk_access_token()
        check_out_response = check_out(user_id, access_token)
        if check_out_response.get('errcode') == 0:
            log_check_out(user_id, datetime.now(), user_location, '正常')
            return jsonify({'message': '下班打卡成功'})
        else:
            log_check_out(user_id, datetime.now(), user_location, '失败')
            return jsonify({'error': '下班打卡失败', 'details': check_out_response}), 500
```

#### 4. 提供日志查询功能
允许用户通过 API 查询历史打卡记录。

```python
@app.route('/list_check_ins')
def list_check_ins():
    user_id = session.get('user_id')
    if not user_id:
        return jsonify({'error': '未登录'}), 401

    check_ins = CheckIn.query.filter_by(user_id=user_id).all()
    check_in_list = [{
        'check_in_id': check_in.check_in_id,
        'check_in_time': check_in.check_in_time.strftime('%Y-%m-%d %H:%M:%S') if check_in.check_in_time else None,
        'check_out_time': check_in.check_out_time.strftime('%Y-%m-%d %H:%M:%S') if check_in.check_out_time else None,
        'location': check_in.location,
        'status': check_in.status
    } for check_in in check_ins]
    return jsonify({'check_ins': check_in_list})
```

### 测试
启动 Flask 应用，测试工作日志记录功能。

1. **登录用户**

   访问 `/login` 路由进行登录。

2. **自动打卡**

   访问 `/auto_check_in_out` 路由，自动执行上下班打卡操作。

   ```sh
   curl http://127.0.0.1:5000/auto_check_in_out
   ```

   你应该会看到类似以下的响应：

   ```json
   {
       "message": "上班打卡成功"
   }
   ```

   或者

   ```json
   {
       "message": "下班打卡成功"
   }
   ```

3. **查看打卡记录**

   访问 `/list_check_ins` 路由，查看历史打卡记录。

   ```sh
   curl http://127.0.0.1:5000/list_check_ins
   ```

   你应该会看到类似以下的响应：

   ```json
   {
       "check_ins": [
           {
               "check_in_id": 1,
               "check_in_time": "2023-10-01 09:00:00",
               "check_out_time": null,
               "location": "(39.9042, 116.4074)",
               "status": "正常"
           },
           {
               "check_in_id": 2,
               "check_in_time": null,
               "check_out_time": "2023-10-01 18:00:00",
               "location": "(39.9042, 116.4074)",
               "status": "正常"
           }
       ]
   }
   ```

### 总结
通过以上步骤，我们实现了工作日志记录功能，记录每次打卡的时间、地点和状态，并提供了日志查询功能。用户可以通过 API 查看历史打卡记录，方便管理和查询。这样可以确保用户能够随时查看自己的打卡记录，提高系统的透明度和用户体验。

Next request.
AI 用户:

<CAMEL_TASK_DONE>
AI 助手:

好的，我们已经完成了智能钉钉自动打卡机器人的所有主要功能模块的实现，包括：

1. 用户认证与授权
2. 地理位置获取与处理
3. 时间管理
4. 自动打卡
5. 异常提醒
6. 假期管理
7. 工作日志记录

如果你有任何其他需求或需要进一步优化的地方，请告诉我。我们可以继续完善和优化这个项目。

如果任务已经完成，你可以确认并关闭任务。感谢你的合作！

任务已完成。如果有其他任务或需求，请随时告知。

Next request.
````

## 学习心得

在学习 Camel 的过程中，我完成了从环境搭建到实际应用的全流程实践。

在 API 调用模型部分，借助 dotenv 库管理 API 密钥的方式让我意识到，在实际开发中保护敏感信息的重要性。通过代码成功调用 Qwen2.5-72B-Instruct 模型并得到响应，让我对 Camel 调用外部模型的流程有了清晰认知，也感受到了其接口设计的便捷性。

参考股票市场交易机器人的思路，构建 “研究生” 与 “Python 程序员” 的双智能体 RolePlaying 框架，让我直观看到了 AI 间协作完成任务的过程。“研究生” 提出需求和反馈，“Python 程序员” 针对性编写和修改代码，这种模拟甲方乙方的互动模式，不仅实现了开发目标，更让我理解了 Camel 在多智能体协作场景下的强大能力。可以根据需求选择不同模型赋能智能体，也体现了框架的灵活性。

这次学习让我掌握了 Camel 的基本使用方法，也让我对多智能体协作的功能之强大感到震撼。

