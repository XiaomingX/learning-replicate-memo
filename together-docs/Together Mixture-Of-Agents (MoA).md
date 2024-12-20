## 了解 Together MoA

### 什么是 Together MoA？
混合智能体（Mixture of Agents，简称 MoA）是一种创新的方法，旨在利用多个大型语言模型（LLM）集体的优势，以提高性能并达到先进的结果。通过采用分层架构，每一层包含多个 LLM 智能体，MoA 在性能上显著超越了 GPT-4 Omni 在 AlpacaEval 2.0 的得分（57.5%），达到了 65.1%，而且仅使用开源模型！

MoA 的工作原理是，当给定一个提示（例如：告诉我在旧金山做什么最有趣）时，它会将此提示发送到四个不同的开源 LLM。然后，MoA 将这四个 LLM 的结果组合在一起，再发送到一个最终的 LLM，请求其将所有四个响应整合成一个理想的回答。尽管这种方法的速度不及单一 LLM，但在延迟不太重要的场景（如合成数据生成）中，它的效果非常理想。

如果您想快速了解如何使用代码实现 MoA，您可以观看下面的视频。

## 使用 50 行代码实现 Together MoA
要在自己的应用中开始使用 MoA，您需要安装 Together Python 库，获取 Together API 密钥，并运行以下代码，利用我们的聊天补全 API 与开源模型进行交互。

### 安装 Together Python 库
首先，您需要安装 Together 库。打开终端并运行命令：
```bash
pip install together
```

### 获取 Together API 密钥并导出
接下来，获取您的 Together API 密钥，并在终端中导出它：
```bash
export TOGETHER_API_KEY='xxxx'
```

### 运行代码，与我们的聊天补全 API 进行交互
以下实现使用了 2 个层级和 4 个 LLM。我们将定义四个初始 LLM 和一个聚合 LLM，以及我们的提示。同时，我们也会增加一个提示，以确保聚合器能够有效地整合响应。完成这些后，我们将同时向四个 LLM 发送提示，并计算所有结果。最终，这些结果将发送到我们的聚合 LLM，以及一个指示其将响应合并为最终答案的系统提示。

```python
import asyncio
import os
from together import AsyncTogether, Together

client = Together()
async_client = AsyncTogether()

user_prompt = "在旧金山有哪些有趣的活动？"
reference_models = [
    "Qwen/Qwen2-72B-Instruct",
    "Qwen/Qwen1.5-72B-Chat",
    "mistralai/Mixtral-8x22B-Instruct-v0.1",
    "databricks/dbrx-instruct",
]
aggregator_model = "mistralai/Mixtral-8x22B-Instruct-v0.1"
aggregator_system_prompt = """你收到了一组来自不同开源模型的响应，任务是将这些响应合成一个高质量的回复。请仔细评估这些响应，辨别其中可能存在的偏见或错误。确保你的回应不仅仅是重复这些答案，而是提供一个经过提炼、准确和全面的回复。请确保你的回答结构清晰、连贯，并符合最高的准确性和可靠性标准。

模型的响应："""

async def run_llm(model):
    """运行单个 LLM 调用。"""
    response = await async_client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": user_prompt}],
        temperature=0.7,
        max_tokens=512,
    )
    print(model)
    return response.choices[0].message.content

async def main():
    results = await asyncio.gather(*[run_llm(model) for model in reference_models])

    finalStream = client.chat.completions.create(
        model=aggregator_model,
        messages=[
            {"role": "system", "content": aggregator_system_prompt},
            {"role": "user", "content": ",".join(str(element) for element in results)},
        ],
        stream=True,
    )

    for chunk in finalStream:
        print(chunk.choices[0].delta.content or "", end="", flush=True)

asyncio.run(main())
```

## 高级 MoA 示例
在之前的示例中，我们演示了怎样使用 2 层（4 个 LLM 和一个聚合 LLM）实现 MoA。但 MoA 的一个强大之处在于可以通过多个层级以获取更好的响应。接下来，我们将介绍一个包含 3 层以上的 MoA 实现示例。

```python
import asyncio
import os
import together
from together import AsyncTogether, Together

client = Together()
async_client = AsyncTogether()

user_prompt = "在旧金山有哪些好玩的活动？"
reference_models = [
    "Qwen/Qwen2-72B-Instruct",
    "Qwen/Qwen1.5-72B-Chat",
    "mistralai/Mixtral-8x22B-Instruct-v0.1",
    "databricks/dbrx-instruct",
]
aggregator_model = "mistralai/Mixtral-8x22B-Instruct-v0.1"
aggregator_system_prompt = """你收到了一组来自不同开源模型的响应，任务是将这些响应合成一个高质量的回复。请仔细评估这些响应，辨别其中可能存在的偏见或错误。确保你的回应不仅仅是重复这些答案，而是提供一个经过提炼、准确和全面的回复。请确保你的回答结构清晰、连贯，并符合最高的准确性和可靠性标准。

模型的响应："""
layers = 3

def getFinalSystemPrompt(system_prompt, results):
    """为 2 层以上生成系统提示，包含之前的响应以进行合成。"""
    return (
        system_prompt
        + "\n"
        + "\n".join([f"{i+1}. {str(element)}" for i, element in enumerate(results)])
    )

async def run_llm(model, prev_response=None):
    """运行单个 LLM 调用，同时考虑之前的响应和速率限制。"""
    for sleep_time in [1, 2, 4]:
        try:
            messages = (
                [
                    {
                        "role": "system",
                        "content": getFinalSystemPrompt(
                            aggregator_system_prompt, prev_response
                        ),
                    },
                    {"role": "user", "content": user_prompt},
                ]
                if prev_response
                else [{"role": "user", "content": user_prompt}]
            )
            response = await async_client.chat.completions.create(
                model=model,
                messages=messages,
                temperature=0.7,
                max_tokens=512,
            )
            print("模型: ", model)
            break
        except together.error.RateLimitError as e:
            print(e)
            await asyncio.sleep(sleep_time)
    return response.choices[0].message.content

async def main():
    """运行 MoA 过程的主循环。"""
    results = await asyncio.gather(*[run_llm(model) for model in reference_models])

    for _ in range(1, layers - 1):
        results = await asyncio.gather(
            *[run_llm(model, prev_response=results) for model in reference_models]
        )

    finalStream = client.chat.completions.create(
        model=aggregator_model,
        messages=[
            {
                "role": "system",
                "content": getFinalSystemPrompt(aggregator_system_prompt, results),
            },
            {"role": "user", "content": user_prompt},
        ],
        stream=True,
    )
    for chunk in finalStream:
        print(chunk.choices[0].delta.content or "", end="", flush=True)

asyncio.run(main())
```

以上代码展示了如何实现一个多层的 MoA 过程，让您可以生成更高质量的响应。