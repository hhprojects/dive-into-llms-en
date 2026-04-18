# Learning LLMs from Scratch: Building GUI Agents

## Tutorial Objectives

1. Understand the technical roadmap and current research state of GUI agent systems
2. Attempt to build adaptive human-computer interaction GUI agents based on the open-source model Qwen2-VL-7B with the OS-Kairos dataset
3. Attempt to perform inference using the constructed GUI agent

## 1. Preparation

### 1.1 Understanding the technical roadmap and research state of GUI agent systems

Read the tutorial: [[Slides](https://github.com/hhprojects/dive-into-llms-en/blob/main/documents/chapter8/GUIagent.pdf)]

### 1.2 Understanding adaptive human-computer interaction GUI agents

Reference paper: [[Paper](https://arxiv.org/abs/2503.16465)]

### 1.3 Dataset preparation

This tutorial uses OS-Kairos as an example, building and inferencing a simple GUI agent based on the open-sourced dataset from this work.

Download the dataset from the download link in the README.md file of [[Data](https://github.com/Wuzheng02/OS-Kairos)] and extract it to your environment.

A sample of the data format is shown below:

```json
    {
        "task": "Open NetEase Cloud Music, search for 'Shape of You', and play the song.",
        "image_path": "/data1/wuzh/cloud_music/images/1736614680.6518524_1.png",
        "list": [
            " Open NetEase Cloud Music  ",
            " Click the search box at the top of the homepage  ",
            " Enter: Shape of You  ",
            " Select the correct search result  ",
            " Click the song name to play  "
        ],
        "now_step": 1,
        "previous_actions": [
            "CLICK <point>[[381,367]]</point>"
        ],
        "score": 5,
        "osatlas_action": "CLICK <point>[[454,87]]</point>",
        "teacher_action": "CLICK <point>[[500,100]]</point>",
        "success": false
    },
```

### 1.4 Model preparation

Download the Qwen2-VL-7B model weights from [[Model](https://huggingface.co/Qwen/Qwen2-VL-7B-Instruct)].

## 1.5 Preparing supervised fine-tuning code

This tutorial uses LLaMa-Factory to perform supervised fine-tuning on Qwen2-VL-7B. First download the source code from [[Code](https://github.com/hiyouga/LLaMA-Factory/)].

## 2. Building GUI Agents

### 2.1 Data preprocessing

Save the following code as get_sharpgpt.py and fill in the paths for Kairos_train.json and the intended location to store the processed training dataset JSON, then run the file.

```python
import json


with open('', 'r', encoding='utf-8') as f:
    data = json.load(f)


preprocessed_data = []
for item in data:
    task = item.get("task", "")
    previous_actions = item.get("previous_actions", "") 
    image_path = item.get("image_path","")
    prompt_text = f"""
    You are now operating in Executable Language Grounding mode. Your goal is to help users accomplish tasks by suggesting executable actions that best fit their needs and give a score. Your skill set includes both basic and custom actions:

    1. Basic Actions
    Basic actions are standardized and available across all platforms. They provide essential functionality and are defined with a specific format, ensuring consistency and reliability. 
    Basic Action 1: CLICK 
        - purpose: Click at the specified position.
        - format: CLICK <point>[[x-axis, y-axis]]</point>
        - example usage: CLICK <point>[[101, 872]]</point>

    Basic Action 2: TYPE
        - purpose: Enter specified text at the designated location.
        - format: TYPE [input text]
        - example usage: TYPE [Shanghai shopping mall]

    Basic Action 3: SCROLL
        - Purpose: SCROLL in the specified direction.
        - Format: SCROLL [direction (UP/DOWN/LEFT/RIGHT)]
        - Example Usage: SCROLL [UP]

    2. Custom Actions
    Custom actions are unique to each user's platform and environment. They allow for flexibility and adaptability, enabling the model to support new and unseen actions defined by users. These actions extend the functionality of the basic set, making the model more versatile and capable of handling specific tasks.

    Custom Action 1: PRESS_BACK
        - purpose: Press a back button to navigate to the previous screen.
        - format: PRESS_BACK
        - example usage: PRESS_BACK

    Custom Action 2: PRESS_HOME
        - purpose: Press a home button to navigate to the home page.
        - format: PRESS_HOME
        - example usage: PRESS_HOME

    Custom Action 3: ENTER
        - purpose: Press the enter button.
        - format: ENTER
        - example usage: ENTER

    Custom Action 4: IMPOSSIBLE
        - purpose: Indicate the task is impossible.
        - format: IMPOSSIBLE
        - example usage: IMPOSSIBLE

    In most cases, task instructions are high-level and abstract. Carefully read the instruction and action history, then perform reasoning to determine the most appropriate next action.

    And I hope you evaluate your action to be scored, giving it a score from 1 to 5. 
    A higher score indicates that you believe this action is more likely to accomplish the current goal for the given screenshot.
    1 means you believe this action definitely cannot achieve the goal.
    2 means you believe this action is very unlikely to achieve the goal.
    3 means you believe this action has a certain chance of achieving the goal.
    4 means you believe this action is very likely to achieve the goal.
    5 means you believe this action will definitely achieve the goal.

    And your final goal, previous actions, and associated screenshot are as follows:
    Final goal: {task}
    previous actions: {previous_actions}
    screenshot:<image>
    Your output must strictly follow the format below, and especially avoid using unnecessary quotation marks or other punctuation marks.(where osatlas action must be one of the action formats I provided and score must be 1 to 5):
    action:
    score:
    """

    action = item.get("osatlas_action", "")
    score = item.get("score", "")
    preprocessed_item = {
        "messages": [
            {
                "role": "user",
                "content": prompt_text,
            },
            {
                "role": "assistant",
                "content": f"action: {action}\nscore:{score}",
            }
        ],
        "images":[image_path]
    }
    preprocessed_data.append(preprocessed_item)


with open('', 'w', encoding='utf-8') as f:
    json.dump(preprocessed_data, f, ensure_ascii=False, indent=4)
```

After this step, we have successfully converted the OS-Kairos training dataset into data in the sharpgpt format, which facilitates the next step of training. Data preprocessing is complete.

## 2.2 Supervised fine-tuning

In the previous step, we obtained the Kairos dataset formatted for LLaMa-Factory training. Next, we need to modify LLaMa-Factory to register the dataset and configure the training information.

First, modify data/dataset_info.json by adding the following content to register the dataset:

```json
"Karios" :{
  "file_name": "Karios_qwenscore.json",
  "formatting": "sharegpt",
  "columns": {
    "messages": "messages",
    "images": "images"
  },
  "tags": {
    "role_tag": "role",
    "content_tag": "content",
    "user_tag": "user",
    "assistant_tag": "assistant"
  }
},   
```

Then modify /examples/train_full/qwen2vl_full_sft.yaml to configure the training information:

```yaml
### model
model_name_or_path: 
stage: sft
do_train: true
finetuning_type: full
deepspeed: examples/deepspeed/ds_z3_config.json

### dataset
dataset: Karios
template: qwen2_vl
cutoff_len: 4096
max_samples: 9999999
#max_samples: 999
overwrite_cache: true
preprocessing_num_workers: 4

### output
output_dir: 
logging_steps: 10
save_steps: 60000
plot_loss: true
overwrite_output_dir: true

### train
per_device_train_batch_size: 2
gradient_accumulation_steps: 2
learning_rate: 1.0e-5
num_train_epochs: 5.0
lr_scheduler_type: cosine
warmup_ratio: 0.1
bf16: true
ddp_timeout: 180000000

### eval
val_size: 0.1
per_device_eval_batch_size: 1
eval_strategy: steps
eval_steps: 20000
```

The above is an example configuration where model_name_or_path is the path to Qwen2-VL-7B and output_dir is the path to store checkpoints and logs.

After configuration, you can set up inference with a sample command line:

```
CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7 FORCE_TORCHRUN=1 llamafactory-cli train examples/train_full/qwen2vl_full_sft.yaml
```

At least 3 A100 GPUs with 80GB memory are required here.

## 3. OS-Kairos Inference Verification

After training is complete, we can perform inference. There are many methods to choose from—you can build inference code using the transformers and torch libraries, or use LLaMa-Factory for one-click inference. The following demonstrates how to use LLaMa-Factory for one-click inference.

First, modify /examples/inference/qwen2_vl.yaml. A sample is shown below:

```yaml
model_name_or_path: 
template: qwen2_vl
```

Where model_name_or_path is the path to the trained checkpoint.

Then you can perform one-click inference via:

```
CUDA_VISIBLE_DEVICES=0 FORCE_TORCHRUN=1 llamafactory-cli webchat examples/inference/qwen2_vl.yaml
```

You can extract images from the OS-Kairos test set or screenshots from your personal phone, and input them to the agent combined with text prompts in the format from get_sharpgpt.py. After inference, OS-Kairos will output the action it believes should be performed currently and the confidence score for that action. The lower the confidence score, the more the current instruction exceeds the capability boundaries of OS-Kairos, and human intervention is needed to perform human-computer interaction.
