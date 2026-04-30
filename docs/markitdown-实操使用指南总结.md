# MarkItDown 实操使用指南总结

## 1. 工具定位

`markitdown` 是 Microsoft 开源的文档转 Markdown 工具，适合把 PDF、Office、HTML、图片、音频等内容转成更适合 LLM、RAG 和文本分析处理的 Markdown。它重点保留标题、列表、表格、链接等结构化信息，而不是追求版式的高保真还原。

## 2. 典型支持格式

常见输入包括：

- PDF
- Word（`.docx`）
- PowerPoint（`.pptx`）
- Excel（`.xlsx` / `.xls`）
- 图片
- 音频
- HTML
- CSV / JSON / XML
- ZIP
- YouTube URL
- EPUB

## 3. 安装建议

推荐使用独立 Python 环境安装，官方要求 Python 3.10+。

`venv` 方式：

```bash
python -m venv .venv
source .venv/bin/activate
pip install 'markitdown[all]'
```

`conda` 方式：

```bash
conda create -n markitdown python=3.12
conda activate markitdown
pip install 'markitdown[all]'
```

完整能力安装：

```bash
pip install 'markitdown[all]'
```

按需安装：

```bash
pip install 'markitdown[pdf,docx,pptx]'
```

如果只处理部分格式，使用 extras 能减少不必要依赖。常见可选依赖包括 `pdf`、`docx`、`pptx`、`xlsx`、`xls`、`audio-transcription`、`youtube-transcription` 等。

## 4. 基本使用方式

命令行最常见的用法是把输入文件直接转换为 Markdown：

```bash
markitdown input.pdf -o output.md
```

也支持标准输出和管道输入：

```bash
markitdown input.pdf > output.md
cat input.pdf | markitdown > output.md
```

常见格式示例：

```bash
markitdown report.docx -o report.md
markitdown slides.pptx -o slides.md
markitdown data.xlsx -o data.md
markitdown page.html -o page.md
markitdown archive.zip -o archive.md
```

对 Word、PPT、Excel、HTML、ZIP 等文件的调用方式相同，只需替换输入文件名。ZIP 的行为是遍历压缩包内容并尝试转换其中的文件。

## 5. Python API

核心调用方式：

```python
from markitdown import MarkItDown

md = MarkItDown(enable_plugins=False)
result = md.convert("test.xlsx")
print(result.text_content)
```

重点是：

- 使用 `MarkItDown(...)` 创建转换器
- 使用 `convert()` 执行转换
- 从 `result.text_content` 读取最终 Markdown 文本

封装成写文件函数的方式也很直接：

```python
from markitdown import MarkItDown
from pathlib import Path

def convert_to_markdown(input_path: str, output_path: str):
    md = MarkItDown(enable_plugins=False)
    result = md.convert(input_path)
    Path(output_path).write_text(result.text_content, encoding="utf-8")


convert_to_markdown("input.pdf", "output.md")
```

这很适合封装到批量处理脚本、知识库预处理流程或服务端任务中。

## 6. LLM 与图片处理

MarkItDown 支持接入大模型，为图片或文档中的图片生成文字描述。对知识库建设尤其有用，因为流程图、截图和图片内容可以被补充为可搜索文本。

示例：

```python
from markitdown import MarkItDown
from openai import OpenAI

client = OpenAI()

md = MarkItDown(
    llm_client=client,
    llm_model="gpt-4o",
    llm_prompt="请描述图片中的主要内容，保留关键信息。"
)

result = md.convert("example.jpg")
print(result.text_content)
```

## 7. 插件与 OCR

插件默认关闭，可通过命令行或 Python 显式启用。

常用场景是配合 `markitdown-ocr` 插件，为 PDF、DOCX、PPTX、XLSX 中的图片增加 OCR 能力，并结合 Vision 模型提取图中文字。

命令行示例：

```bash
markitdown --list-plugins
markitdown document.pdf --use-plugins --llm-client openai --llm-model gpt-4o
```

安装示例：

```bash
pip install markitdown-ocr
pip install openai
```

Python 示例：

```python
from markitdown import MarkItDown
from openai import OpenAI

md = MarkItDown(
    enable_plugins=True,
    llm_client=OpenAI(),
    llm_model="gpt-4o",
)

result = md.convert("document_with_images.pdf")
print(result.text_content)
```

OCR 适合扫描件、截图型文档和图文混排文件。

## 8. Docker 使用

如果不想在本机安装完整 Python 依赖，可以通过 Docker 运行 MarkItDown。适合做环境隔离、批处理或在 CI 中使用。

```bash
docker build -t markitdown:latest .
docker run --rm -i markitdown:latest < ~/your-file.pdf > output.md
```

## 9. 安全注意事项

这是文档里最需要注意的部分之一。

`convert()` 会根据输入触发文件和网络 I/O，因此不适合直接处理不可信用户输入。服务端场景应优先使用更窄权限的接口，例如：

- `convert_local()`：仅处理明确的本地文件
- `convert_response()`：由调用方先自行发起网络请求，再交给 MarkItDown 解析
- `convert_stream()`：自己完全控制输入流

不建议直接这样处理不可信输入：

```python
md.convert(user_input)
```

更安全的本地文件方式：

```python
md.convert_local("/safe/path/file.pdf")
```

实践上还应额外限制：

- 可访问的文件路径范围
- 允许的文件类型
- URL scheme
- 是否允许访问内网地址
- 是否允许访问 metadata service

## 10. 最适合的场景

MarkItDown 最适合以下任务：

- RAG 知识库预处理
- 文档转 Markdown 后交给 AI 总结
- 论文、报告、Office 文件内容提取
- 批量资料清洗
- Agent / LLM 文件读取前的标准化处理

如果目标是“文档内容结构化提取并转成 AI 可读文本”，它很合适；如果目标是“版面级还原”，它不是最佳选择。

## 11. 批量转换脚本

```python
from markitdown import MarkItDown
from pathlib import Path

input_dir = Path("docs")
output_dir = Path("markdown")
output_dir.mkdir(exist_ok=True)

md = MarkItDown(enable_plugins=False)

supported_exts = {".pdf", ".docx", ".pptx", ".xlsx", ".html", ".csv", ".json", ".xml"}

for file in input_dir.iterdir():
    if file.suffix.lower() not in supported_exts:
        continue

    try:
        result = md.convert(str(file))
        output_file = output_dir / f"{file.stem}.md"
        output_file.write_text(result.text_content, encoding="utf-8")
        print(f"转换成功: {file} -> {output_file}")
    except Exception as e:
        print(f"转换失败: {file}, 原因: {e}")
```

目录结构示例：

```text
project/
├── docs/
│   ├── a.pdf
│   ├── b.docx
│   └── c.pptx
├── markdown/
└── convert.py
```

运行方式：

```bash
python convert.py
```

## 12. 一句话结论

`markitdown` 的核心价值不是排版还原，而是把多种异构文档统一转换成结构较清晰、便于大模型消费的 Markdown 文本，是文档预处理链路中的实用工具。
