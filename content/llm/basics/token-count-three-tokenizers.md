---
title: "按字数估 token，同一篇笔记三个词表差出 63%"
date: 2026-10-03
draft: false
tags: ["token", "上下文窗口", "分词"]
summary: "上下文预算和账单都按 token 算，我却一直按字数估。把同一篇 13,152 字符的笔记用三个词表各数一遍，最贵与最省差 63%，按字符数估的方向还会随词表翻转。"
---

## 那次预算估错了

给 `pico` 调上下文预算的时候，我先按字符数估了一遍。手上那篇笔记 13,152 个字符，我就按这个量级当 token 数记下去。

后来把这篇文章真的送进接口，`usage.input_tokens` 报回来的是 6,214，减掉固定开销之后 6,184。我按字符数记的那个数字，比它大一倍还多。

于是我把同一篇材料用三个词表各数了一遍。

## 材料与数法

材料取 `1大模型基础/01 Transformer 与 LLM 基础(重要).md`，13,152 个字符、28,274 字节。

三个词表：

- `cl100k_base`，OpenAI 上一代词表，本地用 `tiktoken` 数
- `o200k_base`，OpenAI 当前词表，同样本地数
- `deepseek-flash` 自己的词表，走接口读 `usage.input_tokens`

接口那个数字要先减一个常数，理由在第六节。

## 一个汉字要几个 token

取一段 204 个字符的纯中文：

- `cl100k_base`：220 个 token，每个字符 1.08 个
- `o200k_base`：148 个 token，每个字符 0.73 个
- `deepseek-flash`：112 个 token，每个字符 0.55 个

同一段中文，最贵的词表比最省的贵 1.96 倍。`cl100k_base` 对中文的合并不充分，一个汉字经常占满一个 token 还有余。

## 换成英文，三个词表几乎一致

712 个字符的英文段落：

- `cl100k_base`：137 个 token，每个字符 0.19 个
- `o200k_base`：137 个 token，每个字符 0.19 个
- `deepseek-flash`：136 个 token，每个字符 0.19 个

差值落在个位数。BPE 词表是为英文语料训出来的，各家在这上面没有分歧。

## 代码落在中间

286 个字符的 Python 片段：

- `cl100k_base`：91 个 token，每个字符 0.32 个
- `o200k_base`：81 个 token，每个字符 0.28 个
- `deepseek-flash`：81 个 token，每个字符 0.28 个

## 合到一整篇

13,152 个字符的笔记：

- `cl100k_base`：10,111 个 token，每个字符 0.77 个
- `o200k_base`：7,289 个 token，每个字符 0.55 个
- `deepseek-flash`：6,184 个 token，每个字符 0.47 个

最贵与最省之间差 63%。

## 换算系数不通用

把 `cl100k_base` 除以 `deepseek-flash`，三种内容给出三个比例：纯中文 1.96，英文 1.01，代码 1.12。

按字符数当 token 数用，错的方向还会翻转。整篇中英混排的材料，字符数比 token 数多，`cl100k_base` 高估 30%，`deepseek-flash` 高估 113%。换成纯中文，`cl100k_base` 变成低估 7.8%（220 对 204），`deepseek-flash` 仍然高估 82%（112 对 204）。

同一个乘数修不掉这两种错。

## 接口报的 token 数含固定开销

上面 `deepseek-flash` 的数字都减了 30，因为这个接口对空字符串也报 30 个 `input_tokens`：

- 空字符串：30
- 单个汉字：31
- 四个汉字：32
- 单个字母：31
- 四个字母：31

`usage.input_tokens` 等于内容加上 30 上下的固定开销。材料越长这笔开销越可以忽略，材料越短它越离谱。我一开始取的 55 个字符的短句被报成 52 个 token，减掉 30 之后是 22 个，比 `cl100k_base` 的 38 和 `o200k_base` 的 24 都低。

拿 `input_tokens` 直接反推分词效率，短输入端会得到错的结论。

## 复现这两组数字

```python
import json, urllib.request, tiktoken

text = open("1大模型基础/01 Transformer 与 LLM 基础(重要).md", encoding="utf-8").read()

for name in ["cl100k_base", "o200k_base"]:
    enc = tiktoken.get_encoding(name)
    print(name, len(enc.encode(text)), "token /", len(text), "字符")

url = "https://api.deepseek.com/anthropic/v1/messages"
headers = {
    "content-type": "application/json",
    "x-api-key": KEY,
    "anthropic-version": "2023-06-01",
}

def input_tokens(content):
    body = json.dumps({
        "model": "deepseek-flash",
        "max_tokens": 1,
        "messages": [{"role": "user", "content": content}],
    }).encode()
    req = urllib.request.Request(url, data=body, headers=headers)
    return json.loads(urllib.request.urlopen(req).read())["usage"]["input_tokens"]

overhead = input_tokens("")
print("deepseek-flash", input_tokens(text) - overhead, "token /", len(text), "字符", "| 固定开销", overhead)
```

## 这笔误差落到哪两处

计费按 token 算，误差按同样比例变成钱。上下文窗口也按 token 算，误差在这里决定的是资料能不能一次装下。

两种错都不能靠乘一个系数修掉，比例随内容类型和词表一起变。
