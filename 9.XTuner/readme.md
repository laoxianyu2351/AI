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
