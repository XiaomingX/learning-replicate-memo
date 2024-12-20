### 如何构建一个AI搜索引擎（类似Perplexity的开源项目）

#### 1. 项目概述
**TurboSeek** 是一个使用 Together AI 开源大语言模型（LLM）的应用。通过调用 Bing API，TurboSeek 从网络中获取多个来源，并将这些内容总结为一个简洁的答案，展示给用户。

在本教程中，您将学习如何构建 TurboSeek 的核心部分。该应用基于 **Next.js** 和 **Tailwind CSS** 构建，但 Together 的 API 可以与任何语言和框架搭配使用。

---

#### 2. 构建输入框
**目标**：在网页上创建一个用户输入问题的文本框。

**实现步骤**：
1. 在页面中渲染一个`<input>`标签，并使用 React 的`useState`钩子来控制输入的内容。

**代码示例**：
```javascript
// app/page.tsx
function Page() {
  const [question, setQuestion] = useState('');

  return (
    <form>
      <input
        value={question}
        onChange={(e) => setQuestion(e.target.value)}
        placeholder="请提出您的问题"
      />
    </form>
  );
}
```
当用户提交表单时，需执行以下操作：
1. 使用 Bing API 从网络获取信息来源。
2. 将这些信息传递给大语言模型（LLM），总结出一个答案。

---

#### 3. 获取网络信息来源

**目标**：
- 在用户提交问题后，调用 **Bing API** 以获取网页来源。

**实现步骤**：
1. 在 `app/page.tsx` 文件中编写表单的提交处理函数：
```javascript
async function handleSubmit(e) {
  e.preventDefault();

  const response = await fetch("/api/getSources", {
    method: "POST",
    body: JSON.stringify({ question }),
  });

  const sources = await response.json();
}
```
2. 创建 API 路由：
```javascript
// app/api/getSources/route.js
export async function POST(req) {
  const json = await req.json();
  const params = new URLSearchParams({
    q: json.question,
    mkt: "en-US",
    count: "6",
    safeSearch: "Strict",
  });

  const response = await fetch(
    `https://api.bing.microsoft.com/v7.0/search?${params}`,
    {
      method: "GET",
      headers: { "Ocp-Apim-Subscription-Key": process.env["BING_API_KEY"] },
    }
  );
  const { webPages } = await response.json();
  return NextResponse.json(
    webPages.value.map(result => ({ name: result.name, url: result.url }))
  );
}
```
3. 在 `.env.local` 文件中存储您的 Bing API 密钥：
```
BING_API_KEY=您的API密钥
```

---

#### 4. 显示网络信息来源
**目标**：在页面中展示从 Bing 返回的搜索结果。

**实现步骤**：
```javascript
function Page() {
  const [question, setQuestion] = useState("");
  const [sources, setSources] = useState([]);

  async function handleSubmit(e) {
    e.preventDefault();
    const response = await fetch("/api/getSources", {
      method: "POST",
      body: JSON.stringify({ question }),
    });
    const sources = await response.json();
    setSources(sources);
  }

  return (
    <>
      <form onSubmit={handleSubmit}>
        <input
          value={question}
          onChange={(e) => setQuestion(e.target.value)}
          placeholder="请输入问题"
        />
      </form>
      {sources.length > 0 && (
        <div>
          <p>信息来源：</p>
          <ul>
            {sources.map((source) => (
              <li key={source.url}>
                <a href={source.url}>{source.name}</a>
              </li>
            ))}
          </ul>
        </div>
      )}
    </>
  );
}
```

---

#### 5. 获取答案摘要
**目标**：从所有来源中获取内容，并生成一个最终的答案。

**实现步骤**：
1. 获取网页文本内容：
```javascript
import { JSDOM } from "jsdom";
import { Readability } from "@mozilla/readability";

async function getTextFromURL(url) {
  const response = await fetch(url);
  const html = await response.text();
  const dom = new JSDOM(html);
  const reader = new Readability(dom.window.document);
  const article = reader.parse();
  return article.textContent;
}
```
2. 从所有来源中获取网页内容：
```javascript
const results = await Promise.all(
  json.sources.map(async (source) => getTextFromURL(source.url))
);
```
3. 使用 Together AI API 获取最终答案：
```javascript
import { Together } from "togetherai";

const together = new Together();

export async function POST(req) {
  const json = await req.json();
  const context = json.sources.map((src) => src.text).join("\n\n");
  const systemPrompt = `
    给出一个简洁且准确的答案。以下是上下文内容：
    ${context}
  `;

  const runner = await together.chat.completions.stream({
    model: "meta-llama/Meta-Llama-3.1-8B-Instruct-Turbo",
    messages: [
      { role: "system", content: systemPrompt },
      { role: "user", content: json.question },
    ],
  });

  return new Response(runner.toReadableStream());
}
```

---

#### 6. 在前端显示答案
**目标**：将 Together AI 生成的答案显示在网页中。

**实现步骤**：
1. 监听数据流并更新 React 状态：
```javascript
import { ChatCompletionStream } from "together-ai/lib/ChatCompletionStream";

function Page() {
  const [answer, setAnswer] = useState("");
  async function handleSubmit(e) {
    e.preventDefault();
    const answerResponse = await fetch("/api/getAnswer", {
      method: "POST",
      body: JSON.stringify({ question, sources }),
    });
    const runner = ChatCompletionStream.fromReadableStream(answerResponse.body);
    runner.on("content", (delta) => setAnswer((prev) => prev + delta));
  }
  return <p>{answer}</p>;
}
```

---

### 总结
通过上述步骤，您已经了解了如何使用 **Next.js**、**Together AI** 和 **Bing API** 构建一个开源的 AI 搜索引擎。核心步骤包括：
1. 创建用户输入的文本框。
2. 使用 Bing API 获取网页来源。
3. 获取网页内容并生成答案。
4. 在网页中展示答案。

如果想要探索更多，请访问 [TurboSeek 的 GitHub 仓库](https://github.com/Nutlope/turboseek/)。

