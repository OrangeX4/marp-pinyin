---
marp: true
title: 智能拼音输入法
description: 基于 HMM 隐马尔可夫模型的智能拼音输入法
theme: gaia
headingDivider: 3
paginate: true
_paginate: false
---

<style>
  :root {
    --color-background: #fff;
    --color-foreground: #333;
    --color-highlight: #f96;
    --color-dimmed: #888;
  }
</style>

<!-- _class: - lead -->

# <!--fit--> 基于 HMM 隐马尔可夫模型的智能拼音输入法

在线访问：http://1.15.246.22:3367/

[![w:64 h:64](https://icongr.am/octicons/mark-github.svg)](https://github.com/OrangeX4/simple-pinyin)

## 目录

1. 使用介绍
2. 拼音划分
3. 获取语料
4. 隐马尔可夫模型
5. 输入 Emoji
6. Web 前端

## 一、使用介绍

首先要安装依赖：

```sh
git clone https://github.com/OrangeX4/simple-pinyin.git

# python
cd pinyin
pip install -r requirements.txt

# web
cd web
npm install
```

简易拼音输入法提供了两种 UI，分别是 **命令行模式** 和 **前端模式**。

### 首先是 **命令行模式**：

```sh
cd pinyin
python ./__init__.py
```

然后输入 `cli` 就能进入命令行模式的拼音输入法了。

```sh
cli
```

###

![bg cover](https://picgo-1258602555.cos.ap-nanjing.myqcloud.com/2022-07-05-13-37-49.png)


### 其次是 **前端模式**：

```sh
cd pinyin
python ./__init__.py
```

然后输入 `server` 就能启动一个服务器后端。再输入

```sh
cd web
npm run start
```

就能启动一个前端界面了。

###

![bg cover](https://picgo-1258602555.cos.ap-nanjing.myqcloud.com/2022-07-05-13-28-32.png)

###

当然也可以作为 API 选择 **在 Python 中导入该输入法** 并使用：

```python
from pinyin.ime import ime

print(ime('jintian', limit=1))  # 基础功能
print(ime('jintain', limit=1))  # 纠错功能
print(ime('ji\'ntian', limit=1))  # 分词功能
print(ime('jintiantianqibucuo', limit=1))  # 短句功能
print(ime('jttqbc', limit=1))  # 首字母功能
print(ime('xiaolian', limit=1))  # emoji 功能
print(ime('nanjingdaxuerengongzhinengxueyuan', limit=1))  # 南京大学人工智能学院
print(ime('nanjingdx', limit=1))  # 南京大学
```

### 终端会输出

```python
[(('jin', 'tian'), '今天', -8.551883029536198)]
[(('jin', 'tian'), '今天', -8.551883029536198)]
[(('ji', 'ni', 'tan'), '记念堂', -23.44058007221551)]
[(('jin', 'tian', 'tian', 'qi', 'bu', 'cuo'), '今天天气不错', -25.909553919977228)]
[(('j', 't', 't', 'q', 'b', 'c'), '今天天气不错', -28.974507662573167)]
[(('xiao', 'lian'), '笑脸', -11.639523579866978),
(('xiao', 'lian'), '😄', -11.639533579866978)]
[(('nan', 'jing', 'da', 'xve', 'ren', 'gong', 'zhi', 'neng', 'xve', 'yuan'),
'南京大学人工智能学院', -53.58047344465382)]
[(('nan', 'jing', 'd', 'x'), '南京大学', -17.561359659026092)]
```

以下是具体实现过程的介绍。


## 二、拼音划分

首先要解决的问题是，如何对长拼音序列进行划分。

- **普通情况**：完整的、无错的、没有歧义的拼音序列，例如 `kongqi` 只能划分为 `kong'qi`，即「空气」，并且是完整的、无错的、没有歧义的。这种情况处理起来比较简单，只需要按照「声母」和「韵母」的简单划分和匹配即可。

###

- **拼音简写**：用户在输入的时候，往往不会输入完整的拼音，而是输入一部分拼音。而缩写的情况又有几种类别，由于没有输入法语料，我就按照我自己使用的简写方式频率排列，以「中国」举例如下：
    - **先整后简**：`zhong'g`，也即「去尾字母完整划分」，我们在输入一个词语的时候，常常会输入了前一个字的完整拼音，又输入后一个字的开头拼音，输入法就会匹配到对应的词语，不需要输入完整的拼音。
    - **完全简写**：`z'g`，我们只输入词语的拼音首字母，也是比较常见的情况。
    - **先简后整**：`z'guo`，比较少见，一般出现在想要使用「完全简写」的方式输入，但是发现匹配不到，因此再输入后一个字的
    - **部分简写**：`zh'g`，不太常见，但是也存在这种情况。

###

- **顺序错误**：用户在输入拼音的时候，可能会因为打字打得比较快，有一些字的拼音的顺序弄反了，例如「小路」的 `xiao'lu` 打成了 `xaio'lu`，这时候输入法应该给予纠正。
- **存在歧义**：例如 `xianmianguan` 既可以划分为 `xian'mian'guan`，即「鲜面馆」，也可以划分为 `xi'an'mian'guan`，即「西安面馆」，这时候拼音划分就存在着歧义。如果涉及到拼音简写，则歧义会更多，如 `zhongguo` 甚至可以划分为 `z'hong'gu'o`。

###

我们需要将拼音划分分为两个不同的场景，不同场景的应用不同。第一个场景是「用户输入拼音序列划分」，第二个场景是「文字转拼音后划分」，前者用于预测，后者用于训练。

用户输入拼音序列划分只需要使用简单的动态规划即可实现，将所有合法的拼音序列划分方式都给列举出来，然后同时进行预测。

首先是输入拼音序列的划分，可以通过 `from pinyin.cut import cut_pinyin` 引入，具体的实现代码如下，使用了简单的动态规划，并加入了使用 `'` 分词的功能：

###

```python
# 动态规划判断进行拼音划分
def cut_pinyin(pinyin: str, is_intact=False, is_break=True):
    '''
    进行拼音划分, 返回拼音划分结果列表
    pinyin: 待划分的拼音, 并且是无空格字符串, 例如 `kongjian`
    is_intact: 拼音是否需要完整匹配, 默认为 False, 可以使用残缺部分的拼音进行分词
    is_break: 是否开启分隔符, 开启后可以使用 ' 进行分割, 例如 `kong'jian`
    
    return: 拼音划分结果列表, 例如 `cut_pinyin('kongjian', True)`, 
            会返回 `[('kong', 'jian'), ('kong', 'ji', 'an')]`
    '''
    if is_intact:
        pinyin_set = intact_pinyin_set
        ans_dict = intact_cut_pinyin_ans    
    else:
        pinyin_set = all_pinyin_set
        ans_dict = all_cut_pinyin_ans
```

###

```python
    # 如果保存有, 直接返回保存结果
    if pinyin in ans_dict:
        return ans_dict[pinyin]
    # 如果 is_break, 就进行分割
    if is_break and '\'' in pinyin:
        pinyins = pinyin.split('\'')
        components = [cut_pinyin(p, is_intact, False) for p in pinyins]
        ans = components[0]
        for i in range(1, len(components)):
            ans = [p1 + p2 for p1 in ans for p2 in components[i]]
        return ans
    # 如果没有, 递归地动态规划生成
    ans = [] if pinyin not in pinyin_set else [(pinyin,)]
    for i in range(1, len(pinyin)):
        # 进行划分 pinyin[:i], 如果是正确拼音, 就继续动态规划
        if pinyin[:i] in pinyin_set:
            appendices = cut_pinyin(pinyin[i:], is_intact, is_break=False)
            for appendix in appendices:
                ans.append((pinyin[:i],) + appendix)
    ans_dict[pinyin] = ans
    return ans
```

###

在 `cut_pinyin` 函数的基础上，我们可以加入拼音纠错功能，从第二个字母开始依次交换连续的两个字母，看看是否能够进行完整拼音划分，能的话就加入最终结果，进而实现纠错功能。

例如 `jaiot` 会依次以 `ajiot`、`jiaot`、`jaoit`、`jaito` 的次序进行纠错划分尝试，最后找到可行的划分方式 `jiao't`。

###

```python
def cut_pinyin_with_error_correction(pinyin: str):
    '''
    纠错匹配, 从第二个字母开始, 依次交换两个连续字母并进行*完整划分*.
    如果完整划分返回非空列表, 即匹配成功, 并加入到返回字典中.
    pinyin: 待纠错划分的拼音

    return: 返回字典, 字典的 key 为纠错后的拼音序列, value 为匹配成功的划分结果.
            并且会包含一个 key = 'all' 的项, 包括了所有 value.
    '''
    ans = {}
    for i in range(1, len(pinyin) - 1):
        # 避免交换分词符
        if pinyin[i-1] == '\'' or pinyin[i] == '\'' or pinyin[i + 1] == '\'':
            continue
        key = pinyin[:i] + pinyin[i + 1] + pinyin[i] + pinyin[i + 2:]
        value = cut_pinyin(key, is_intact=True)
        if value:
            ans[key] = value
    ans['all'] = [p for t in ans.values() for p in t]
    return ans
```


## 三、获取语料

这里我直接使用了「北京语言大学 BCC 语料库 http://bcc.blcu.edu.cn」的词频语料 `global_wordfreq.release.txt`，最终解释权归北语大数据与教育技术研究所所有。

其中的语料格式大致为：

```text
第	2002074595
的	943370349
了	255733044
在	197672850
是	171296602
我	169391220
```

###

```text
~	44057380
非常	8056541
一直	8013106
不会	8010572
应该	8001472
```

即「词语 + 词频」的组合，不过注意到有一些非中文的词语，例如 `~	44057380`，因此我们需要进行过滤，最后再使用 Python 的生成器功能，我们就能解耦合地进行数据的读入，具体代码位于 `train/dataset.py`。

###

```python
def is_Chinese(word):
    '''
    判断一个字符串是否全由汉字组成, 用于过滤文本
    '''
    return all('\u4e00' <= ch <= '\u9fff' for ch in word)

def iter_word_and_freq():
    """
    词频数据集, 迭代地返回 (word, freq)
    """
    with open(words_path, 'r', encoding='utf-8') as f:
        for line in f:
            try:
                word, frequency = line.split()
                # 进行过滤
                if is_Chinese(word):
                    yield word, int(frequency)
            except Exception as e:
                pass
```


## 四、隐马尔可夫模型

**隐马尔可夫模型** (Hidden Markov Model, HMM) 是一种统计模型，用来描述一个含有隐含未知参数的马尔可夫过程。

隐马尔可夫模型有两个关键的概念：**状态** (state) 和 **观测** (observation)。隐马尔可夫链随机生成的状态的序列，称为状态序列；每个状态生成一个观测，由此产生的随机的观测的序列，称为观测序列。序列的每一个位置又可以看作一个时刻。

###

![w:1080](https://picgo-1258602555.cos.ap-nanjing.myqcloud.com/2022-07-04-20-23-37.png)

对于拼音输入法来说，**状态就是一个个汉字**，**观测就是对应的拼音**。作为状态的汉字是不知道的，唯一知道的只有用户输入的观测，也就是拼音。

###

不过值得一提的是，由于状态转移矩阵 $A$ 和发射矩阵 $B$ 都是 **稀疏矩阵**，即大部分位置均为 $0$，如果用普通的保存方式会十分占据空间，因此我们使用 **JSON 格式** 将 Python 的字典保存下来。后续使用的时候，只需要加载 JSON 文件，就能重新恢复为位于内存中的 Python 字典了。对应生成的 JSON 文件位于 `data/hmm_xxx.json`.

另外，由于后续概率计算数字可能越算越小，导致计算机无法计算，所以我们对所有概率都进行了 **自然对数运算处理**。

### 生成的 `hmm_start.json` 的部分内容：

```json
{
    "一": -5.081293678906249,
    "丁": -9.192433659783104,
    "丂": -19.34085746030175,
    "七": -10.159009715890134,
    "丄": -16.747217610377284,
    "丅": -16.301496324358553,
    "丆": -19.25047339883348,
    "万": -8.7175005342102
}
```

### 生成的 `hmm_transition.json` 的部分内容：

```json
{
    "渗": {
        "入": -2.439070674759006,
        "出": -1.9373713062580147,
        "漏": -2.4294073791727087,
        "透": -0.5143045447650618,
    },
    "渚": {
        "文": -0.50469675623453,
        "港": -2.8828938894856275,
        "湖": -2.9667753734663296,
        "镇": -1.7247845019528727,
    }
}
```

### 生成的 `hmm_emission.json` 的部分内容：

```json
{
    "一": {
        "y": -0.9808292530117262,
        "yi": -0.4700036292457356
    },
    "模": {
        "m": -0.9808292530117262,
        "mo": -0.5430199262500778,
        "mu": -3.12336226291328
    }
}
```

###

训练完隐马尔可夫模型后，我们就要进行预测了。

隐马尔可夫模型的预测问题，也称为解码 (decoding) 问题，就是在已知隐马尔可夫模型 $\lambda=(\pi, A, B)$ 和观测序列 $O=(o_1, o_2, \cdots, o_{T})$ 的情况下，求使得观测序列条件概率 $P(I|O)$ 最大的状态序列 $I=(i_1, i_2, \cdots, i_{T})$. 即给定观测序列，求最有可能的状态序列。

这里我们使用维特比算法 (Viterbi algorithm) 来进行预测。

为了加速维特比算法, 我们要先通过倒查表的方式计算出 `reversed_emission_matrix` 和 `reversed_transition_matrix`.

###

然后是维特比算法的具体代码 [参考](https://github.com/LiuRoy/Pinyin_Demo):

```python
def viterbi(pinyin, limit=10):
    """
    viterbi 算法

    pinyin: 拼音元组, 例如 ('jin', 'tian')

    return: 返回 limit 个最可能的汉字序列, 但是是 1 个全局最优解和 limit - 1 个局部最优解
            并且返回剩余未搜索的拼音
    """
    # 初始化, 找出第一个拼音对应的汉字以及 start 和 emission 概率之积 (对数下为相加)
    char_and_prob = ((ch, start_vector[ch] + reversed_emission_matrix[pinyin[0]][ch])
        for ch in reversed_emission_matrix[pinyin[0]])
    # 取出概率最大的 limit 个
    V = {char: prob for char, prob
        in heapq.nlargest(limit, char_and_prob, key=lambda x: x[1])}
```

###

```python
    for i in range(1, len(pinyin)):
        py = pinyin[i]

        prob_map = {}
        for phrase, prob in V.items():
            previous = phrase[-1]
            if previous in reversed_transition_matrix and py
                in reversed_transition_matrix[previous]:
                    state, new_prob = reversed_transition_matrix[previous][py]
                    prob_map[phrase + state] = new_prob + prob

        if prob_map:
            V = prob_map
        else:
            # 没有概率, 因此没有完全搜索, 返回目前结果和未搜索拼音 pinyin[i:]
            return sorted(V.items(), key=lambda x: x[1], reverse=True), pinyin[i:]
    return sorted(V.items(), key=lambda x: x[1], reverse=True), ''
```

###

最后综合我们的分词功能和维特比算法，即可得到一个较为智能的输入法了。

```python
# 缓存结果, 加快判断
dp = {}
def ime(pinyin: str, limit=7):
    '''
    输入法函数, 综合分词和维特比算法的最终结果
    '''
    if pinyin in dp:
        return dp[pinyin][:limit]
    # 计算结果
    result = []
    # 获取分词结果
    cut = cut_pinyin_with_strategy(normlize_pinyin(pinyin))
```

###

```python
    # 先尝试完整划分
    for pinyin in cut['intact'] + cut['intact_tail']:
        vit, remain_pinyin = viterbi(pinyin)
        if not remain_pinyin:
            result.extend([(pinyin,) + t for t in vit])
    # 如果完整划分的最小拼音大小小于等于 3, 则进行纠错
    if not result or min([len(pinyin) for pinyin in cut['intact'] + cut['intact_tail']]) <= 3:
        for pinyin in cut['error_correction'] + cut['error_correction_tail']:
            vit, remain_pinyin = viterbi(pinyin)
            if not remain_pinyin:
                result.extend([(pinyin,) + t for t in vit])
    # 如果结果为空, 则进行模糊划分
    if not result:
        for pinyin in cut['fuzzy']:
            vit, remain_pinyin = viterbi(pinyin)
            result.extend([(pinyin,) + t for t in vit])
    # 排序并取出前 limit 个
    dp[pinyin] = sorted(result, key=lambda x: x[2], reverse=True)
    return dp[pinyin][:limit]
```

## 五、输入 Emoji

一个现代的输入法，还应该拥有输入 Emoji 的功能。因此我们首先要获取 Emoji 对应的数据。

我从 https://www.emojiall.com/zh-hans/all-emojis 中获取了所有的中文和 emoji 的对应数据，保存在了 `data/emoji.txt` 中。通过代码

###

```python
def gen_emoji_json():
    emoji_dict = {}
    with open(emoji_file_path, 'r', encoding='utf-8') as f:
        emoji = ''
        for line in f:
            line = line.strip()
            # 跳过数字
            if all(ch in '1234567890' for ch in line):
                continue
            if emoji:
                # 去除前缀
                if line.startswith('旗: '):
                    line = line[3:]
                emoji_dict[line] = emoji
                emoji = ''
            else:
                emoji = line

    # save to data/emoji.json
    with open(emoji_json_path, 'w', encoding='utf-8') as f:
        json.dump(emoji_dict, f, ensure_ascii=False, indent=4)


def load_emoji_dict():
    return json.load(open(emoji_json_path, 'r', encoding='utf-8'))
```

###

我生成了如下格式的 emoji 字典，即中文和 emoji 的对应表：

```json
{
    "笑脸": "😄",
    "苦笑": "😅",
    "斜眼笑": "😆",
    "微笑天使": "😇",
    "呵呵": "🙂",
    "倒脸": "🙃",
    "笑得满地打滚": "🤣",
    "花痴": "😍",
    "亲亲": "😗",
    "飞吻": "😘",
    "吐舌脸": "😛",
    "好吃": "😋",
    "想一想": "🤔",
}
```

###

最后我们更新一下 `ime()` 函数，如果第一个中文有对应 emoji 则在第二位加入 emoji 即可。

```python
def ime(pinyin: str, limit=7):
    '''
    输入法函数, 综合分词和维特比算法的最终结果, 并且会加入 emoji
    '''
    def replace_with_emoji(tuples):
        # 如果第一个中文有对应 emoji, 则使用 emoji 将其替换
        if tuples and tuples[0][1] in emoji_dict:
            return [tuples[0], (tuples[0][0], emoji_dict[tuples[0][1]],
                tuples[0][2] - 1e-5)] + tuples[1:-1]
        else:
            return tuples
    if pinyin in dp:
        return replace_with_emoji(dp[pinyin][:limit])
```


## 六、Web 前端

Web 前端采用了 React 框架，个人比较喜欢 Google 家的 Material Design，因此选用了 MUI，一款基于 React 框架的 Material 组件库。

![w:800](https://www.freecodecamp.org/news/content/images/size/w2000/2022/04/featured.jpg)

###

![bg](https://picgo-1258602555.cos.ap-nanjing.myqcloud.com/2022-07-05-13-28-32.png)

###

![](https://picgo-1258602555.cos.ap-nanjing.myqcloud.com/2022-07-05-13-29-51.png)

###

整个 UI 界面非常简单，由三个主要部分组成。

1. 位于左上方的拼音输入法输入框，用以显示当前输入的拼音内容，例如当前为 `xiao'lian`，然后通过拼音实时获取到对应的推荐词列表。输入框的右边还会包括一个 emoji 选择按钮。
2. 位于左下方的推荐词列表，最多显示 7 个。其中还包括匹配 emoji 的显示。
3. 位于右边的文本输入框，会捕获按键并同步输入法输入的内容，可以用于测试输入法的效果。

前端具体的代码比较繁杂，这里也不过多赘述。

# 🎉
<!-- _class: - lead -->

#### Thanks!
