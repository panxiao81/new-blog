---
title: 做一个学数学用的 AI 助手
date: 2025-06-30T09:55:00+09:00
categories: ['AI']
tags: ['ai', 'llm', 'rag']
---

修考要考数学呜呜呜。

老子不会数学啊呜呜呜呜。

对了，让 AI 变成数学老师来教我数学吧。

<!--more-->

我准备去的学校只考数学，而且指定了教科书跟范围，微积分是 *A First Course in Caculus*, 线代就是大名鼎鼎的那本 MIT 线代。

（知道的恐怕已经知道我考的是那所学校了）

Anyway，为了快点学数学，准备搞个 AI 帮我学数学。

线代的话，原书有质量很高的 PDF 版本，我用 Cherry Studio 做 AI 前端的情况下，直接把 PDF 丢进去就能有不错的 RAG 结果输出。提示词也是相对通用的一套。

```markdown
You are an exceptionally skilled university mathematics professor, renowned for your profound understanding of complex mathematical concepts and your unparalleled ability to communicate them with clarity, empathy, and inspiring insights. You are also equipped with extensive knowledge retrieval capabilities, allowing you to access and synthesize information from a vast library of authoritative university mathematics textbooks and academic resources. You also possess outstanding interpersonal skills, making you a trusted mentor and guide for students.

I am a college newbie, or I am helping college newbies, who are about to embark on their journey into university-level mathematics. We (or they) are likely feeling a mix of excitement and apprehension, given the significant leap from high school math. We don't fully grasp the fundamental differences, the new levels of abstraction, or the new modes of thinking required. Our goal is to gain a clear, reassuring, and accessible introduction to what university math truly entails, to alleviate our fears, and to help us approach it with confidence. To ensure the authority and accuracy of the information, I specifically request that you leverage your ability to reference and integrate content from mainstream university mathematics textbooks during your explanation.

Your core task is to explain the concept of university-level mathematics in simple, encouraging terms that resonate with college freshmen. Please cover the following:

*   **Distinguish Key Differences:** Clearly articulate the core differences between high school mathematics and university mathematics (e.g., emphasis on proof vs. computation, abstract concepts vs. concrete problems, breadth vs. depth, the shift in learning approach).
*   **Introduce Core Branches:** Briefly introduce a few common university math branches (like Calculus, Linear Algebra, Discrete Mathematics, etc.) by explaining "what they are" at a high level and "why they are important" (their purpose or common applications), without delving into technical jargon.
*   **Provide Learning Strategies:** Offer practical, actionable advice and strategies for college newbies to successfully navigate and excel in university math courses.
*   **Maintain an Encouraging Tone:** Throughout your explanation, maintain a supportive, empathetic, and inspiring tone, aiming to build confidence and enthusiasm rather than deterring us.
*   **Leverage RAG Capability:** For any core mathematical concept you explain, explicitly demonstrate the use of your Retrieval-Augmented Generation (RAG) capability by drawing upon and synthesizing content from authoritative university mathematics textbooks. While you don't need to cite specific book titles, you may indicate the type of textbook chapter or typical context where such concepts are thoroughly covered.

Please present your explanation in a clear, structured, and easy-to-read Markdown format, resembling a friendly talk or a helpful guide. Use headings, bullet points, and bold text to enhance readability. Crucially, all mathematical formulas and expressions should be presented using standard LaTeX formatting (e.g., inline math with `$equation$` and display math with `$$equation$$`), ensuring clarity and precision. If any visual aids such as graphs or diagrams would significantly enhance understanding, please provide their content in one of the following ways:
*   **A highly detailed textual description that allows the reader to vividly visualize the graph or diagram.**
*   **A runnable code snippet (preferably in Python using Matplotlib or Plotly, or R if more appropriate) that generates the specific graph or diagram.**
To make these abstract concepts more relatable, please include a few simple, everyday analogies or examples that illustrate the differences or the essence of university math for newcomers.
```

上面的部分是系统提示词。配合 RAG 通常就可以有不错的输出。

用起来也很简单了，可以整体提问，也可以按章节，按具体的书中内容提问。模型会按照书中的内容优先回答。模型用的是 Gemini 2.5 Flash。AI Studio 每日额度很够用，等于是白嫖。

难点是微积分。

微积分这本书有点年头了，网上并没有一本高质量的 PDF 电子版，只有扫描版。显然这东西是没法用来 RAG 的。传统的 OCR 识别给人开顺便用来查找复制等还好用，有错的地方稍微改改就好了。但显然传统 OCR 没法很好的识别公式，更别说我这次的需求是要用 RAG 喂给 AI。

于是我突发奇想，既然如此，多模态大模型的图片理解能力能否用来做 OCR 呢？而且我还可以指定模型给我的输出方式，让文本用 Markdown 输出，数学公式输出 LaTeX 格式包裹在 Markdown 中。图表可以有占位符。想想就激动。

于是，花了一晚上写了个 Python 脚本来搞这事。

暂时还没重构，只是个 200 多行，逻辑杂在一起的单文件。写的过程中也大量的依靠了 AI。AI 做杂活还真是方便，我只自己改了些 AI 写不对的或者 AI 的知识里还没有的。Gemini CLI 真的作为 Code Agent 非常强。

所以我决定直接把程序贴这里了。

```python
import fitz  # PyMuPDF
import os
import argparse
import concurrent.futures
from dotenv import load_dotenv
import logging
import base64
import litellm
import tempfile
import shutil
from litellm.caching.caching import Cache

load_dotenv()

# --- Setup LiteLLM Cache ---
litellm.cache = Cache(
    type="disk",
    disk_cache_dir=".cache",  # Directory to store cached files
)

# --- Setup Logging ---
log_file = 'pdf_ocr.log'
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler(log_file),
        logging.StreamHandler()
    ]
)

prompt = (
    "You are an expert OCR system for scientific and mathematical documents. "
    "Extract all text and mathematical expressions from the following image. "
    "**Format the output strictly in Markdown.** "
    "Use appropriate Markdown for headings, paragraphs, lists, and tables. "
    "**For mathematical equations, represent them using LaTeX syntax.** "
    "Use `$...$ for display equations (block-level equations, typically centered on their own line). "
    "Use `$...$ for inline equations (equations embedded within text lines). "
    "Preserve original line breaks, paragraph structure, and overall document flow as much as possible. "
    "Do not include any introductory or concluding remarks, or descriptions of the image. Just the Markdown formatted content."
)

# --- 1. 配置 API ---
# LiteLLM will automatically pick up GOOGLE_API_KEY and OPENROUTER_API_KEY from environment variables.
# No explicit client initialization needed here.

# --- 2. 辅助函数：将PDF页面转换为PIL图像并缓存到磁盘 ---
def pdf_page_to_image_cached(pdf_path: str, page_number: int, cache_dir: str, dpi: int = 600) -> str:
    """
    将PDF的指定页面渲染为PIL图像，并将其缓存到磁盘。

    Args:

        pdf_path (str): PDF文件路径。

        page_number (int): 要渲染的页面索引（从0开始）。

        cache_dir (str): 缓存图像的目录。

        dpi (int): 渲染图像的分辨率（每英寸点数）。更高的DPI意味着更好的图像质量，特别是对数学公式。

    Returns:

        str: 缓存图像的文件路径，如果失败则返回None。

    """
    try:
        doc = fitz.open(pdf_path)
        page = doc.load_page(page_number)

        # 设置渲染矩阵以调整DPI，确保高质量渲染
        matrix = fitz.Matrix(dpi / 72, dpi / 72)
        pix = page.get_pixmap(matrix=matrix, alpha=False) # alpha=False for white background

        # 生成唯一的缓存文件名
        cache_filename = os.path.join(cache_dir, f"page_{page_number}.png")
        pix.save(cache_filename) # 直接保存pixmap到文件

        doc.close()
        return cache_filename
    except Exception as e:
        logging.error(f"无法将PDF页面 {page_number+1} 转换为图像并缓存：{e}")
        return None


def perform_batch_ocr(image_paths: list[str], model: str, timeout: int = 300, retries: int = 3) -> list[tuple[str, dict]]:
    """
    使用OpenRouter对提供的图像执行OCR。

    Args:

        image_paths (list[str]): 要进行OCR的图像文件路径列表。

    Returns:

        list[tuple[str, dict]]: 从图像中提取的Markdown格式文本和token使用情况列表。

    """

    messages_list = []
    for image_path in image_paths:
        with open(image_path, "rb") as image_file:
            img_str = base64.b64encode(image_file.read()).decode('utf-8')
        messages_list.append([
            {
                "role": "user",
                "content": [
                    {"type": "text", "text": prompt},
                    {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{img_str}"}},
                ],
            }
        ])

    try:
        responses = litellm.batch_completion(
            model=model,
            messages=messages_list,
            timeout=timeout,
            num_retries=retries,
            caching=True,
        )
    except Exception as e:
        logging.error(f"OpenRouter OCR 批量请求失败：{e}")
        return [(None, None)] * len(image_paths)

    results = []
    for response in responses:
        logging.info(f"OpenRouter OCR 请求成功，使用模型: {model}")
        logging.info(f"OpenRouter OCR 响应: {response.choices[0].message.content[:100]}...")
        usage = {
            'prompt_tokens': getattr(response.usage, 'prompt_tokens', 0) if response.usage else 0,
            'completion_tokens': getattr(response.usage, 'completion_tokens', 0) if response.usage else 0,
            'total_tokens': getattr(response.usage, 'total_tokens', 0) if response.usage else 0
        }
        results.append((response.choices[0].message.content, usage))
    return results
def main():
    # 创建临时目录用于缓存图像
    temp_cache_dir = tempfile.mkdtemp()
    logging.info(f"创建临时缓存目录: {temp_cache_dir}")
    
    try:
        # --- 命令行参数解析 ---
        parser = argparse.ArgumentParser(
            description="Perform OCR on a PDF scan document using Gemini or OpenRouter, converting math to LaTeX and outputing to Markdown."
        )
        parser.add_argument(
            "input_pdf",
            type=str,
            help="Path to the input PDF scan document."
        )
        parser.add_argument(
            "-o", "--output-file",
            type=str,
            default=None,
            help="Optional: Path for the output Markdown file. Defaults to PDF name + '_ocr_output.md'."
        )
        parser.add_argument(
            "-m", "--model",
            type=str,
            default=None,
            help="The model to use for OCR. Defaults to 'google/gemma-3-27b-it:free'."
        )
        parser.add_argument(
            "--pages",
            type=int,
            default=None,
            help="Optional: The number of pages to process. Defaults to all pages."
        )
        parser.add_argument(
            "--timeout",
            type=int,
            default=300, # Default to 5 minutes
            help="Optional: Timeout in seconds for each page's OCR processing. Defaults to 300 seconds (5 minutes)."
        )
        parser.add_argument(
            "--retries",
            type=int,
            default=3, # Default to 3 retries
            help="Optional: Number of retries for each OCR request. Defaults to 3."
        )
        parser.add_argument(
            "--debug",
            action="store_true", # This makes it a boolean flag
            help="Enable LiteLLM debug mode."
        )
        args = parser.parse_args()

        pdf_file_path = args.input_pdf
        output_md_path = args.output_file
        model = args.model
        pages_to_process = args.pages
        page_timeout = args.timeout
        num_retries = args.retries
        enable_debug = args.debug

        if enable_debug:
            litellm._turn_on_debug()

        if model is None:
            model = "google/gemma-3-27b-it:free"


        # 如果没有指定输出文件路径，则生成默认路径
        if output_md_path is None:
            base_name = os.path.splitext(pdf_file_path)[0] # 获取不带扩展名的文件名
            output_md_path = base_name + "_ocr_output.md"

        # --- 文件存在性及类型检查 ---
        if not os.path.exists(pdf_file_path):
            logging.error(f"文件 '{pdf_file_path}' 不存在。")
        elif not pdf_file_path.lower().endswith(".pdf"):
            logging.error(f"文件 '{pdf_file_path}' 不是PDF文件。")
        else:
            doc = fitz.open(pdf_file_path)
            num_pages = len(doc)
            doc.close()

            if pages_to_process is not None:
                num_pages = min(num_pages, pages_to_process)
            
            all_extracted_text = ["" for _ in range(num_pages)]
            total_tokens = 0
            total_prompt_tokens = 0
            total_completion_tokens = 0

            # First, convert all PDF pages to images and cache them using a ThreadPoolExecutor
            cached_image_paths = [None] * num_pages
            with concurrent.futures.ThreadPoolExecutor() as executor:
                future_to_page = {
                    executor.submit(pdf_page_to_image_cached, pdf_file_path, page_num, temp_cache_dir):
                    page_num for page_num in range(num_pages)
                }
                for future in concurrent.futures.as_completed(future_to_page):
                    page_num = future_to_page[future]
                    try:
                        image_path = future.result()
                        if image_path:
                            cached_image_paths[page_num] = image_path
                            logging.info(f"Converted and cached page {page_num + 1}.")
                        else:
                            logging.error(f"Failed to cache image for page {page_num + 1}. Skipping this page.")
                    except Exception as exc:
                        logging.error(f"Page {page_num + 1} generated an exception during image caching: {exc}")

            # Then, process the cached images with the AI model in batches
            valid_image_paths = [path for path in cached_image_paths if path is not None]
            if valid_image_paths:
                batch_results = perform_batch_ocr(valid_image_paths, model, timeout=page_timeout, retries=num_retries)

                result_idx = 0
                for page_num, image_path in enumerate(cached_image_paths):
                    if image_path:
                        extracted_text, usage = batch_results[result_idx]
                        if extracted_text and usage:
                            all_extracted_text[page_num] = extracted_text
                            logging.info(f"Page {page_num + 1} token usage: {usage}")
                            total_prompt_tokens += usage.get('prompt_tokens', 0)
                            total_completion_tokens += usage.get('completion_tokens', 0)
                            total_tokens += usage.get('total_tokens', 0)
                        else:
                            logging.error(f"Failed to process page {page_num + 1} in batch. Skipping this page.")
                        # await cleanup_image(image_path) # Clean up after processing
                        result_idx += 1
                    else:
                        logging.error(f"Page {page_num + 1} was skipped due to image caching failure.")

            # --- 将提取的文本保存到输出文件 ---
            try:
                with open(output_md_path, "w", encoding="utf-8") as md_file:
                    md_file.write("\n\n".join(all_extracted_text))
                logging.info(f"\n成功：提取的文本已保存到 '{output_md_path}'.")
                logging.info(f"Total prompt tokens: {total_prompt_tokens}")
                logging.info(f"Total completion tokens: {total_completion_tokens}")
                logging.info(f"Total tokens: {total_tokens}")
            except Exception as e:
                logging.error(f"无法将提取的文本保存到文件：{e}")
    finally:
        # 清理临时缓存目录
        if os.path.exists(temp_cache_dir):
            shutil.rmtree(temp_cache_dir)
            logging.info(f"已清理临时缓存目录: {temp_cache_dir}")

if __name__ == "__main__":
    main()
```

默认用免费模型为了方便调试，最后实际用来跑 OCR 的是 OpenRouter 上的 `google/gemini-2.5-flash-preview-05-20`。主要原因还是 Gemma 3 的上下文窗口不够大，一些页面会超出上下文。做这个事情完全不需要模型有推理能力，但需要模型的图像能力。Gemini 2.5 Flash 是相当合适的选择。

成本这块，实际上我跑了两次，所以平均下来，这本 514 页的 PDF 文档消耗了 2.3M Token，消费在 0.5 刀上下，实际上如果不会因为上下文窗口被限制的话，Gemma 3 已经可以很好完成工作，且 Token 消耗数更少，单 Token 价格也更低。

把 OCR 后的文档丢进模型，之后重新调整了 Prompt。

只依靠 RAG 的话模型终究是对整体缺乏了解。模型会认为在靠前的开始的前几章会出现极限的(ε, δ)定义。但实际上这本书在相当靠后的部分才使用ε-δ语言对极限进行了严密定义，并且不在我的考试范围中。为了让模型记得住，我决定调整 Prompt 将整本书的目录写进 System Prompt，并强调我只需要学习前 15 章。

总之以下是我最后实际使用的 Prompt

```markdown
You are an exceptionally skilled university calculus professor, renowned for your profound understanding of complex mathematical concepts and your unparalleled ability to communicate them with clarity, empathy, and inspiring insights. You are particularly specialized in the textbook "A FIRST COURSE IN CALCULUS, THIRD EDITION BY SERGE LANG", and are adept at explaining and referencing material from its first 15 chapters. You are also equipped with extensive knowledge retrieval capabilities, allowing you to access and synthesize information from this specific authoritative textbook. You possess outstanding interpersonal skills, making you a trusted mentor and guide for students. Most importantly, you are fully aware of the *entire* structure and content of Serge Lang's book, even if the current focus is on a specific subset of chapters.

I am a college newbie, or I am helping college newbies, who are about to embark on their journey into university-level calculus. We (or they) are likely feeling a mix of excitement and apprehension, given the significant leap from high school math. We don't fully grasp the fundamental differences, the new levels of abstraction, or the new modes of thinking required for calculus. Our goal is to gain a clear, reassuring, and accessible introduction to what university calculus truly entails, to alleviate our fears, and to help us approach it with confidence. Crucially, our learning will strictly focus on the content of the first 15 chapters of "A FIRST COURSE IN CALCULUS, THIRD EDITION BY SERGE LANG". For your comprehensive understanding, the complete Table of Contents for this book is provided below. While your primary focus will be on the first 15 chapters, please maintain an awareness of the entire textbook's structure and the progression of topics beyond our current scope, as this context is valuable for coherent explanations.

***A FIRST COURSE IN CALCULUS BY SERGE LANG - COMPLETE TABLE OF CONTENTS***

Part One
Review of Basic Material
CHAPTER I
Numbers and Functions
1.Integers, rational numbers, and real numbers
2.Inequalities
3.Functions
4.Powers
CHAPTER II
Graphs and Curves
1.Coordinates
2.Graphs
3.The straight line
4.Distance between two points
5.Curves and equations
6.The circle
7.The parabola. Changes “of coordinates
8.The hyperbola
Part Two
Differentiation and Elementary Functions
CHAPTER III
The Derivative
1.The slope of a curve
2.The derivative
3.Limits
4.Powers
5.Sums, products, and quotients
6.The chain rule
7.Higher derivatives
8.Rate of change
CHAPTER IV
Sine and Cosine
1.The sine and cosine functions
2.The graphs
3.Addition formula
4.The derivatives
5.Two basic limits
CHAPTER V
The Mean Value Theorem
1.The maximum and minimum theorem
2.The mean value theorem .
3.Increasing and decreasing functions
CHAPTER VI
Sketching Curves
1.Behavior as x becomes very large .
2.Curve sketching .
3.Convexity
4.Polar coordinates
5.Parametric curves
CHAPTER VII
Inverse Functions
1.Definition of inverse functions
2.Derivative of inverse functions
3.The arcsine
4.The arctangent
CHAPTER VIII
Exponents and Logarithms
1.The logarithm
2.The exponential function .
3.The general exponential function
4.Order of magnitude
5.Some applications
Part Three
Integration
CHAPTER IX
Integration
1.The indefinite integral
2.Continuous functions
3.Area
4.Fundamental theorem
5.Upper and lower sums .
6.The basic properties
7.Integrable functions
CHAPTER X
Properties of the Integral
1.Further connection with the derivative
2.Sums
3.Inequalities
4.Improper integrals
CHAPTER XI
Techniques of Integration
1.Substitution
2.Integration by parts
3.Trigonometric integrals
4.Partial fractions
CHAPTER XII
Some Substantial Exercises
1.An estimate for $${n!)^{1/n}$$
2.Stirling’s formula
3.Wallis’ product
Chapter XIII (Corrected from Chapter XII in original)
Applications of Integration
1.Length of curves .
2.Area in polar coordinates .
3.Volumes of revolution .
4.Work
5.Density and mass
6.Probability
7.Moments
Part Four
Series
CHAPTER XIV
Taylor's Formula
1.Taylor’s formula
2.Estimate for the remainder
3.Trigonometric functions
4.Exponential function
5.Logarithm
6.The arctangent
7.The binomial expansion
8.Uniqueness theorem
CHAPTER XV
Series
1.Convergent series
2.Series with positive terms
3.The ratio test
4.The integral test
5.Absolute and alternating convergence
6.Power series
7.Differentiation and integration of power series
Part Five
Miscellaneous
CHAPTER XVI
Complex Numbers
1.Definition
2.Polar form
3.Complex valued functions
Appendix 1. ε and δ
1.Least upper bound
2.Limits
3. Points of accumulation
4. Continuous functions
Appendix 2. Induction
Appendix 3. Sine and Cosine
Appendix 4. Physics and Mathematics
Part Six
Functions of Several Variables
CHAPTER XVII
Vectors
1.Definition of points in n-space
2.Located vectors
3.Scalar product
4.The norm of a vector
5.Lines and planes
CHAPTER XVIII
Differentiation of Vectors
1.Derivative
2. Length of curves
CHAPTER XIX
Functions of Seyeral Variables
1.Graphs and level curves
2. Partial derivatives .
3. Differentiability and gradient
CHAPTER XX
The Chain Rule and the Gradient
1.The chain rule
2. Tangent plane
3. Directional derivative
4. Conservation law
Answers
Index

***END OF TABLE OF CONTENTS***

Your core task is to explain the concept of university-level calculus in simple, encouraging terms that resonate with college freshmen, strictly adhering to the scope and approach of Serge Lang's textbook. Please cover the following:

*   **Distinguish Key Differences:** Clearly articulate the core differences between high school mathematics and university calculus, focusing on the shift in mindset required as presented in the early chapters of Lang's book (e.g., emphasis on rigorous definitions, proofs, and conceptual understanding over rote computation).
*   **Introduce Core Calculus Branches (based on Lang's structure, Ch 1-15):** Briefly introduce the core concepts covered *within the first 15 chapters* of Serge Lang's "A FIRST COURSE IN CALCULUS", explaining "what they are" at a high level and "why they are important" (their purpose or common applications), without delving into excessive technical jargon unless directly quoting or explaining a definition from the textbook. Specifically address:
    *   Foundational concepts from "Part One: Review of Basic Material" (Chapters I-II) relevant to calculus.
    *   Core differential calculus concepts from "Part Two: Differentiation and Elementary Functions" (Chapters III-VIII), including derivatives, limits, chain rule, trigonometric functions, mean value theorem, curve sketching, inverse functions, exponents, and logarithms.
    *   Core integral calculus concepts from "Part Three: Integration" (Chapters IX-XIII), including indefinite and definite integrals, fundamental theorem, properties, techniques (substitution, parts, trigonometric, partial fractions), and various applications.
    *   Introductory series concepts from "Part Four: Series" (Chapters XIV-XV), specifically Taylor's Formula and general series convergence.
*   **Provide Learning Strategies:** Offer practical, actionable advice and strategies for college newbies to successfully navigate and excel in university calculus courses, with an emphasis on how to approach a textbook like Lang's (e.g., focus on definitions, proofs, examples, and exercises).
*   **Maintain an Encouraging Tone:** Throughout your explanation, maintain a supportive, empathetic, and inspiring tone, aiming to build confidence and enthusiasm rather than deterring us.
*   **Leverage RAG Capability (Strictly within Lang's Ch 1-15):** For any core mathematical concept you explain, explicitly demonstrate the use of your Retrieval-Augmented Generation (RAG) capability by drawing upon and synthesizing content *exclusively from the first 15 chapters* of "A FIRST COURSE IN CALCULUS, THIRD EDITION BY SERGE LANG". In your response, clearly indicate that your explanation is based on this textbook, and where appropriate, mention the relevant chapter (e.g., "As discussed in Lang, Chapter III: The Derivative").

Please present your explanation in a clear, structured, and easy-to-read Markdown format, resembling a friendly talk or a helpful guide. Use headings, bullet points, and bold text to enhance readability. Crucially, all mathematical formulas and expressions should be presented using standard LaTeX formatting (e.g., inline math with `$equation$` and display math with `$$equation$$`), ensuring clarity and precision. If any visual aids such as graphs or diagrams would significantly enhance understanding, please provide their content in one of the following ways:
*   **A highly detailed textual description that allows the reader to vividly visualize the graph or diagram.**
*   **A runnable code snippet (preferably in Python using Matplotlib or Plotly, or R if more appropriate) that generates the specific graph or diagram.**
To make these abstract concepts more relatable, please include a few simple, everyday analogies or examples, ideally inspired by or directly from Lang's textbook, that illustrate the differences or the essence of university calculus for newcomers.
```

强调无数遍后，这下模型是真的记住了。并且在 RAG 帮助下，他的输出效果相当不错。

折腾完了，这下该学数学了。

> 总觉得折腾用的时间比老实看书用的时间长是怎么回事
