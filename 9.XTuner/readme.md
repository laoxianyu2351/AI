# XTuner

## 基础任务

* 使用 XTuner 微调 InternLM2-Chat-7B 实现自己的小助手认知，如下图所示（图中的 `尖米`需替换成自己的昵称），记录复现过程并截图。

### 使用 conda 先构建一个 Python-3.10 的虚拟环境

```
cd ~
git clone https://github.com/InternLM/Tutorial.git -b camp4
mkdir -p /root/finetune && cd /root/finetune
conda create -n xtuner-env python=3.10 -y
conda activate xtuner-env
```

![1733315217904](image/readme/1733315217904.png)

![1733315636444](image/readme/1733315636444.png)

### 安装 XTuner

```
git clone https://github.com/InternLM/xtuner.git
cd /root/finetune/xtuner

pip install  -e '.[all]'
pip install torch==2.4.1 torchvision==0.19.1 torchaudio==2.4.1 --index-url https://download.pytorch.org/whl/cu121
pip install transformers==4.39.0
```

 这里安装的时间非常长……需要耐心等待（如果不指定版本的话会更快，里面有几个指定版本导致装了卸载，卸载了安装）

完成后可以测试一下有没有安装上。

```
xtuner list-cfg
```

没有报错就是正常的。

![1733486982824](image/readme/1733486982824.png)

### 修改提供的数据

创建一个新的文件夹用于存储微调数据

```
mkdir -p /root/finetune/data && cd /root/finetune/data
cp -r /root/Tutorial/data/assistant_Tuner.jsonl  /root/finetune/data
```

完成后对应目录应该会有红框的文件

![1733487065122](image/readme/1733487065122.png)

然后我们需要将里面的数据修改一下，目前有的数据里面都是'尖米'的智能助手，我们要改成自己的，可以简单粗暴地全量替换，课程这里给我们提供了一个脚本，让我们自己去选择更换。

```
import json
import argparse
from tqdm import tqdm

def process_line(line, old_text, new_text):
    # 解析 JSON 行
    data = json.loads(line)
  
    # 递归函数来处理嵌套的字典和列表
    def replace_text(obj):
        if isinstance(obj, dict):
            return {k: replace_text(v) for k, v in obj.items()}
        elif isinstance(obj, list):
            return [replace_text(item) for item in obj]
        elif isinstance(obj, str):
            return obj.replace(old_text, new_text)
        else:
            return obj
  
    # 处理整个 JSON 对象
    processed_data = replace_text(data)
  
    # 将处理后的对象转回 JSON 字符串
    return json.dumps(processed_data, ensure_ascii=False)

def main(input_file, output_file, old_text, new_text):
    with open(input_file, 'r', encoding='utf-8') as infile, \
         open(output_file, 'w', encoding='utf-8') as outfile:
  
        # 计算总行数用于进度条
        total_lines = sum(1 for _ in infile)
        infile.seek(0)  # 重置文件指针到开头
  
        # 使用 tqdm 创建进度条
        for line in tqdm(infile, total=total_lines, desc="Processing"):
            processed_line = process_line(line.strip(), old_text, new_text)
            outfile.write(processed_line + '\n')

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Replace text in a JSONL file.")
    parser.add_argument("input_file", help="Input JSONL file to process")
    parser.add_argument("output_file", help="Output file for processed JSONL")
    parser.add_argument("--old_text", default="尖米", help="Text to be replaced")
    parser.add_argument("--new_text", default="咸鱼", help="Text to replace with")
    args = parser.parse_args()

    main(args.input_file, args.output_file, args.old_text, args.new_text)
```

弄好后运行我们就可以生成带我们自己的ID的数据了

```
python change_script.py ./assistant_Tuner.jsonl ./assistant_Tuner_change.jsonl
```

我们可以打开看看。
![1733487263529](image/readme/1733487263529.png)
这里面都是将尖米替换成了我们想要的名字了。

数据准备就绪了

### 训练启动

开发机里面本身就有一个我们所需要的模型，因此我们直接可以复制进来，然后开始训练，也就不用下载了。

```
mkdir /root/finetune/models

ln -s /root/share/new_models/Shanghai_AI_Laboratory/internlm2_5-7b-chat /root/finetune/models/internlm2_5-7b-chat
```

另外我们也需要小小的修改一下训练脚本
先获取到官方的。

```
cd /root/finetune
mkdir ./config
cd config
xtuner copy-cfg internlm2_5_chat_7b_qlora_alpaca_e3 ./
```

修改我直接贴我完成修改的截图吧。
![1733487493028](image/readme/1733487493028.png)
![1733487522664](image/readme/1733487522664.png)
![1733487558191](image/readme/1733487558191.png)

接下来就可以开始训练了

```
cd /root/finetune
conda activate xtuner-env

xtuner train ./config/internlm2_5_chat_7b_qlora_alpaca_e3_copy.py --deepspeed deepspeed_zero2 --work-dir ./work_dirs/assistTuner
```

这一步训练是比较花时间的，建议多等等，效果会比较好，我等了很久……（后面忘记关开发机了，所以后面我也不晓得等了多久……）

通过查看last_checkpoint文件可以知道我们的训练中loss最少的是哪个模型，模型的文件位置所在。
![1733487778551](image/readme/1733487778551.png)

训练出来的是pth文件，我们还要转换一下，转为HF的格式。

```
xtuner convert pth_to_hf ./internlm2_5_chat_7b_qlora_alpaca_e3_copy.py /root/finetune/work_dirs/assistTuner/iter_864.pth ./hf
```

这里每个人因为最好的训练模型是不一样的，所以命令也会有差异。

然后转换完成后应该是这样的。
![1733487921716](image/readme/1733487921716.png)

我们还需要将这个adapter合并进大模型里去。

```
xtuner convert merge /root/finetune/models/internlm2_5-7b-chat ./hf ./merged --max-shard-size 2GB
```

这里我遇到了爆显存的乌龙，其实是因为之前意外又跑了一次训练，导致显存不够，程序跑不了。后面使用 `kill -9 <进程ID>`结束掉了之前的多余训练……

文件内容如图
![1733488053681](image/readme/1733488053681.png)

最后是本地的webui展示，这里在 `~/Tutorial/tools/L1_XTuner_code`也有现成的，我们拿来小小改一下~

```
cd ~/Tutorial/tools/L1_XTuner_code
```

![1733488167803](image/readme/1733488167803.png)

最后启动并做好端口映射

```
streamlit run /root/Tutorial/tools/L1_XTuner_code/xtuner_streamlit_demo.py
```

端口映射8501
![1733488331039](image/readme/1733488331039.png)

测试一下有没有自我认知（是不是'咸鱼'）:
![1733488750455](image/readme/1733488750455.png)

完成基础任务。

* 进阶任务将自我认知的模型上传到 HuggingFace/Modelscope/魔乐平台，并将应用部署到 HuggingFace/Modelscope/魔乐平台
* * 参与社区共建，获取浦语 api 创建自己的数据用于微调

其实我们这边自我认知的大模型已经有了，接下来就是将模型上传部署。这边因为大模型在开发机，我实测开发机没办法用git clone命令将HF社区的space克隆下来（网络问题）。
所以我换了一个平台，我决定使用**Modelscope**。

先在平台上生成一个仓库
![1733488660220](image/readme/1733488660220.png)

接下来就是本地克隆下来，然后将模型放进去，再推过去。

下面是我git放在缓存区的……（这几个权重文件真的太大了……花了我好久时间）
![1733488948277](image/readme/1733488948277.png)

……小丑了，需要token
![1733489250959](image/readme/1733489250959.png)

需要使用以下命令：

```
<!-- 这里其实就是之前mass课程里面的那个变通 -->
git remote set-url origin https://<ID>:<token>Q@modelscope.cn/<repo>
```

嘻嘻，搞定了，传起来咯，不过文件特别大，我后面还是直接放传上去的网页吧。
![1733489485663](image/readme/1733489485663.png)

完成了~
![1733490010159](image/readme/1733490010159.png)

接下来就是获取自己的训练数据，由于我实在没看明白getdata文件里面有什么问题，但就是跑不出来，所以自己写一个mini版的……貌似不知道问题，浦语貌似没办法在messages里面同时添加两个角色？？？我加上去会导致返回的回答是什么都没有……

```
from openai import OpenAI
import time

base_id_prompt ="Role: 问答机器人\n\n## Profile\n- author: 尖米\n- version: 1.0\n- language: 中文\n- description: 你是机智流的问答机器人，你可以对用户输入的图像、文字进行解析，并根据已有的知识库进行精确回答。\n\n## Skills\n1. 图像识别与解析：能够识别用户上传的图像，并提取其中的关键信息。\n2. 自然语言处理：能够理解并解析用户输入的文字信息，准确把握用户意图。\n3. 知识库应用：根据解析结果，查询知识库，提供准确、相关的答案。\n4. 多轮对话：支持与用户进行多轮对话，提供连续性、上下文相关的回答。\n\n## Rules\n1. 必须充分理解用户输入的图像和文字内容。\n2. 回答需要简洁明了，避免过于复杂或含糊的表述。\n3. 在回答过程中，优先查询和引用公司已有的知识库。\n4. 对于无法回答的问题，需要引导用户提供更多信息或寻求人工客服帮助。\n\n## Workflows\n1. 接收并分析用户输入的图像或文字信息。\n2. 基于图像识别或自然语言处理技术，提取关键信息。\n3. 查询知识库，匹配相关信息。\n4. 向用户提供精准、相关的回答。\n5. 如有必要，进行多轮对话，确保问题得到有效解决。\n\n## Init\n欢迎使用机智流的问答机器人，请输入您的问题，我将尽力为您提供帮助。\n",
results = []
client = OpenAI(
    api_key="eyJ0eXBlIjoiSldUIiwiYWxnIjoiSFM1MTIifQ.eyJqdGkiOiIxNzAwODc5MCIsInJvbCI6IlJPTEVfUkVHSVNURVIiLCJpc3MiOiJPcGVuWExhYiIsImlhdCI6MTczMzQ5NTQxMSwiY2xpZW50SWQiOiJlYm1ydm9kNnlvMG5semFlazF5cCIsInBob25lIjoiMTgwMTg4Nzc4NjIiLCJ1dWlkIjoiNmJmN2RjYzEtMzA3ZC00NDJiLTk3NjMtMWMzNDg3ZTFkOGRhIiwiZW1haWwiOiIiLCJleHAiOjE3NDkwNDc0MTF9.qFrYB2Z8sMf3YWb9s2YNqxPQyQ6Bh6ls6TUCgZpboOndaPJ-JSySzoqlANvg8z-PYgGkDo4nXOzaWEvWnUFYCQ",  # 此处传token，不带Bearer
    base_url="https://internlm-chat.intern-ai.org.cn/puyu/api/v1/",
)

with open("/root/Tutorial/tools/L1_XTuner_code/Q_list1.txt","r") as f:
    questions = f.read().split('\n')

for question in questions:
    chat_rsp = client.chat.completions.create(
        model="internlm2.5-latest",
        messages=[
                    # {"role": "system", "content": base_id_prompt},
                    {"role": "user", "content": question}
                ],
    )

    # for choice in chat_rsp.choices:
        # print(choice.message.content)
    result = "{\"conversation\":[{\"system\":\"%s\",\"input\":\"%s\",\"output\":\"%s\"}]}" % (base_id_prompt,question,chat_rsp.choices[0].message.content.replace("\n",""))
    results.append(result)
    time.sleep(1)
        

with open("/root/Tutorial/data/my.jsonl","w+") as f :
    for result in results:
        f.write(result+"\n")

```

其实训练数据基本应该是这样弄出来的吧……但是我这边能跑是能跑，过了一段时间后还是会出现问题。他会跳出来访问过于频繁……
所以我放弃了，实测是能有数据的，只要问题不多的话，一点一点来是可以的

我这边取了一点点巧，做了个7个问题的mini文件，生成的数据如图：
![1733541494539](image/readme/1733541494539.png)
