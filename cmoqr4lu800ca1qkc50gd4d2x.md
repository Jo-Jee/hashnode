---
title: "Why Korean Loses on LLMs"
datePublished: 2026-05-04T05:21:11.316Z
cuid: cmoqr4lu800ca1qkc50gd4d2x
slug: why-korean-loses-on-llms
cover: https://cdn.hashnode.com/uploads/covers/69c777487cf2706510c47f9a/74da9bd5-017c-499b-a27b-8de4b756bf48.png

---

> Audience: Anyone using an LLM API who has wondered "Why does the same content cost more tokens in Korean?" Feed the same document to a model in English and Korean and the token counts differ by more than 2x. The bill differs by 2x too. This post is about why.

* * *

## 0\. One-page summary

```shell
Tokenizing 1,000 characters of equivalent meaning (cl100k_base):

English   ████                            ~250 tokens (1.0x)

Mandarin  ███████                       ~440 tokens (1.76x)

Cantonese █████████                    ~525 tokens (2.10x)

Japanese  █████████                    ~530 tokens (2.12x)

Korean    ██████████                  ~590 tokens (2.36x)  
```

Three things matter:

1.  **A tokenizer is a function of its training corpus distribution.** GPT-3's corpus was 92.6% English and 0.017% Korean. That ratio gets baked into token efficiency.
    
2.  **Hangul is the most expensive among CJK.** The 11,172 precomposed Hangul syllable blocks don't all fit in the vocab, so byte fallback kicks in often.
    
3.  **This gap *is* the cost, latency, and context-length gap.** APIs bill per token. Putting the same information in Korean fills more than twice the context window.
    

The good news: GPT-4o expanded the vocab from 100k to 200k and the Korean ratio narrowed from **~2.36x to ~1.1x**. The gap is closing, but on top of BPE, Hangul remains structurally disadvantaged.

* * *

## 1\. Why does identical meaning produce different token counts

### 1.1 BPE in one paragraph

Almost every commercial LLM today (GPT, Claude, Llama, Gemini) uses **BPE (Byte Pair Encoding)** or a variant. The algorithm itself is simple:

1.  Split text into a UTF-8 byte sequence.
    
2.  From the training corpus, merge **the byte pair that co-occurs most often** into a single token.
    
3.  Repeat up to the vocab size limit (e.g. 100,000).
    

The resulting vocab packs **frequent expressions into a single token** and **leaves rare expressions split into many byte-level tokens**. What counts as "frequent" is decided entirely by the training corpus.

### 1.2 Language distribution of the training corpus

GPT-3's published training data language ratios (Common Crawl basis):

| Language | Share |
| --- | --- |
| English | 92.65% |
| French | 1.82% |
| German | 1.47% |
| ... |  |
| Japanese | 0.16% |
| **Korean** | **0.017%** |

English patterns like `the`, `tion`, `ing` appear hundreds of millions of times in the corpus, so they earn a single token slot in the vocab. Korean patterns like `습니다`, `하는데`, `안녕` appear at roughly 1/5000 the rate of English equivalents, so they don't make it into the vocab.

What happens to Hangul that doesn't get in? **It gets split into UTF-8 bytes.**

* * *

## 2\. Why Hangul is the most expensive of CJK

### 2.1 The Unicode structure of a Hangul syllable

A Hangul syllable is the composition `consonant (initial) + vowel (medial) + consonant (final)`. The total number of theoretically possible syllable blocks:

```plaintext
19 initials × 21 medials × (27 finals + 1 no-final) = 11,172 syllables
```

Unicode allocates all 11,172 of these as **precomposed** code points in the range **U+AC00 – U+D7A3**. So `한`, `국`, `어` are each a single code point.

In UTF-8:

```plaintext
ASCII 'a'  → 0x61                       (1 byte)
Hangul '한' → 0xED 0x95 0x9C            (3 bytes)
Han  '中'  → 0xE4 0xB8 0xAD             (3 bytes)
Kana 'あ'  → 0xE3 0x81 0x82             (3 bytes)
```

Every CJK character takes 3 bytes. So far so equal.

### 2.2 So why is Korean still worse

The answer is **which characters made it into the vocab**.

| Language | Corpus share | Frequently used characters | Coverage in vocab |
| --- | --- | --- | --- |
| Mandarin | ~0.16% | ~3,500 common Hanzi | most frequent Hanzi → 1 token |
| Japanese | ~0.16% | ~2,136 Joyo Kanji + 92 kana | partial Kanji + kana → 1 token |
| Korean | **0.017%** | ~2,500 syllable blocks (in practice) | only the most frequent get a token |

Korean's corpus share is **an order of magnitude lower**. On top of that, with 11,172 possible Hangul combinations, syllables that didn't make the vocab get split into their **3 UTF-8 bytes** during tokenization.

Verify with OpenAI's official tokenizer (`cl100k_base`):

```python
import tiktoken
enc = tiktoken.get_encoding("cl100k_base")

enc.encode("Hello, world")
# → [9906, 11, 1917]                              (3 tokens, 12 chars → 0.25 tok/char)

enc.encode("안녕하세요, 세계")
# → [16715, 53899, 23821, 246, 116, 24486, ...]   (10 tokens, 9 chars → 1.11 tok/char)
```

Same greeting, but Korean costs about **4.5x** more tokens than English here.

### 2.3 The same character splits differently

Whether a single character is in the vocab or not decides its token count.

| Character | Frequency | cl100k tokens |
| --- | --- | --- |
| `한` | very common | 1 |
| `국` | very common | 1 |
| `어` | very common | 1 |
| `욱` | moderate | 2 (a merged 2-byte token + 1 byte) |
| `뙇` | rare | 3 (each byte as its own token) |

The Hangul paradox: **Hangul = jamo composition**, but Unicode allocates the whole syllable as a single code point. BPE never sees a learning signal at the jamo level. Decomposing into jamo actually makes things *worse*:

```plaintext
"앤"      → 2 tokens (precomposed)
"ㅇㅐㄴ"  → 7 tokens (jamo-decomposed, even worse)
```

English BPE operates over 26 letters and naturally learns morpheme-like units. Korean BPE is scattered across 11,172 syllable blocks.

* * *

## 3\. What this means in practice

### 3.1 Cost

OpenAI bills both input and output by the token. Processing the same document in Korean:

```plaintext
GPT-4 ($30 / 1M input tokens)

100,000 chars of English  → ~25,000 tokens → $0.75
100,000 chars of Korean   → ~59,000 tokens → $1.77   (2.36x)
```

For a high-volume pipeline this stacks. A service processing 100M tokens a month will see unit cost climb with every Korean-heavy traffic share.

### 3.2 Context window

The same nominal 128k context behaves differently:

```plaintext
GPT-4 Turbo 128k context

  English   ~512KB of text (≈ one full book)
  Korean    ~217KB of text (≈ half of that)
```

For long-document summarization, RAG, or agent systems, **Korean users hit context pressure twice as fast**. The same document fills the window quicker, forcing chunking sooner.

### 3.3 Latency

LLM latency is **token count × per-token latency**. Twice the input means twice the prefill. Twice the output means twice the generation. The reason a Korean response feels slower than the equivalent English response is rarely model speed — **it's just more tokens**.

### 3.4 Model quality

This one is subtler and more fundamental than cost. A token born from byte fallback **carries little semantic content**. To a model that sees `한` as `0xED, 0x95, 0x9C` (three bytes), that character carries almost no meaning. English `the` is a single token with a single embedding that owns the meaning, while Korean meaning is scattered across bytes.

This is one driver (not the only one) of hallucination, awkward postpositional choices, and homophone confusion in Korean LLM output.

* * *

## 4\. Differences across models and tokenizers

### 4.1 OpenAI: cl100k → o200k

| Tokenizer | Vocab size | Models | Korean efficiency |
| --- | --- | --- | --- |
| `r50k_base` | 50,257 | GPT-3 | ~3x worse than English |
| `cl100k_base` | 100,277 | GPT-3.5, GPT-4 | ~2.36x |
| `o200k_base` | 200,019 | GPT-4o, o1 | **~1.1x** |

`o200k_base` doubled the vocab and deliberately included a large slice of non-English (Korean, Japanese, Chinese) tokens. According to a Microsoft Asia analysis, Korean token usage dropped roughly 50% from GPT-4 to GPT-4o.

### 4.2 Anthropic Claude

Claude uses its own tokenizer; the exact vocab is private. The API exposes a separate `/messages/count_tokens` endpoint to count tokens. Empirically, Korean efficiency sits between GPT-4 and GPT-4o.

### 4.3 Llama / Gemma family

Open-source models tend to use **SentencePiece**\-based tokenizers. Llama 3 expanded its vocab from 32k to 128k, dramatically improving multilingual efficiency. Models that explicitly target multilingual support (Llama 3, Qwen, Gemma 2) are far more balanced than GPT-3.5-era tokenizers.

### 4.4 Korean-specialized models

Korean-centric models like HyperCLOVA X and EXAONE **design the vocab from the start to represent Korean well**. For the same Korean input, token counts can be 1/2 to 1/3 of GPT-4. For services with heavy Korean traffic, they're worth evaluating.

* * *

## 5\. Practical mitigations

### 5.1 Measure first

Optimization is meaningless without knowing your service's actual Korean/English token ratios. Measure directly.

```python
import tiktoken

enc = tiktoken.get_encoding("o200k_base")  # or cl100k_base

ko_text = open("korean_sample.txt").read()
en_text = open("english_sample.txt").read()

ko_tok = len(enc.encode(ko_text))
en_tok = len(enc.encode(en_text))

print(f"Korean: {ko_tok} tokens / {len(ko_text)} chars = {ko_tok/len(ko_text):.3f} tok/char")
print(f"English: {en_tok} tokens / {len(en_text)} chars = {en_tok/len(en_text):.3f} tok/char")
```

### 5.2 Write system prompts in English

You can't change the language users speak in, but the **system prompt, few-shot examples, and function descriptions** that ride along on every request are pure overhead. Authoring them in English cuts tokens with negligible quality loss — instruction-following is often *more* stable in English since it's the most heavily trained language.

### 5.3 Factor token efficiency into model selection

Tokenizer differences shift the per-request cost of Korean traffic across same-priced tiers. The fundamental reason GPT-4o is cheaper *and* faster than GPT-4 for Korean is the tokenizer change, not just inference speed.

### 5.4 Embeddings deserve more care

Embedding models like `text-embedding-3-large` use the same tokenizer. In RAG indexing, a Korean chunk fits half the content of an English chunk under the same token cap. Consider chunking by character or sentence count rather than tokens.

### 5.5 Prompt caching is language-independent in mechanism, but Korean-favorable in effect

Anthropic and OpenAI both bill prompt caching per token, so **the absolute savings of a single cache hit are larger in Korean**. The more aggressively you use caching, the more the language gap shrinks.

* * *

## 6\. One-line summary

> **A tokenizer is a mirror of its training corpus. With 92% English and 0.02% Korean, English becomes one token and Korean fragments into bytes. Hangul, layered on top with its 11,172 syllable blocks, is the most expensive of the CJK family. The gap shows up as cost, latency, and context length — narrowing with bigger vocabs, but never structurally erased on top of BPE.**

Next time your Korean LLM bill is double the English equivalent, the model didn't get pricier. **The tokenizer is just chopping Korean into smaller pieces.**

* * *

## References

*   Petrov et al. — [Language Model Tokenizers Introduce Unfairness Between Languages (NeurIPS 2023)](https://arxiv.org/pdf/2305.15425)
    
*   Tony Baloney — [Working with Chinese, Japanese, and Korean text in Generative AI pipelines](https://tonybaloney.github.io/posts/cjk-chinese-japanese-korean-llm-ai-best-practices.html)
    
*   OpenAI — [tiktoken (BPE tokenizer implementation)](https://github.com/openai/tiktoken)
    
*   Modal — [What is o200k Harmony? OpenAI's latest tokenizer](https://modal.com/blog/what-is-o200k-harmony)
    
*   arXiv — [Problematic Tokens: Tokenizer Bias in Large Language Models](https://arxiv.org/html/2406.11214)
    
*   arXiv — [How does a Language-Specific Tokenizer affect LLMs?](https://arxiv.org/html/2502.12560v1)
    
*   arXiv — [Optimizing Korean-Centric LLMs via Token Pruning](https://arxiv.org/html/2604.16235)
    
*   ZDNet Korea — [GPT-4o achieves Korean token efficiency (Microsoft analysis)](https://zdnet.co.kr/view/?no=20240430131643)