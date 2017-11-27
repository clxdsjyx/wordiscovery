
# 词频、互信息、信息熵发现中文新词

**新词发现**任务是中文自然语言处理的重要步骤。**新词**有“新”就有“旧”，属于一个相对个概念，在相对的领域（金融、医疗），在相对的时间（过去、现在）都存在新词。[文本挖掘](https://zh.wikipedia.org/wiki/文本挖掘)会先将文本[分词](https://zh.wikipedia.org/wiki/中文自动分词)，而通用分词器精度不过，通常需要添加**自定义字典**补足精度，所以发现新词并加入字典，成为文本挖掘的一个重要工作。

[**单词**](https://zh.wikipedia.org/wiki/單詞)的定义，来自维基百科的定义如下：
>在语言学中，**单词**（又称为词、词语、单字；英语对应用语为“word”）是能独立运用并含有语义内容或语用内容（即具有表面含义或实际含义）的最小单位。单词的集合称为词汇、术语，例如：所有中文单词统称为“中文词汇”，医学上专用的词统称为“医学术语”等。词典是为词语提供音韵、词义解释、例句、用法等等的工具书，有的词典只修录特殊领域的词汇。

单从语义角度，“苹果“的法语是"pomme"，而“土豆”的法语是"pomme de terre"，若按上面的定义，“土豆”是要被拆的面目全非，但"pomme de terre"是却是表达“土豆”这个语义的最小单位；在机构名中，这类问题出现的更频繁，"Paris 3"是"巴黎第三大学"的简称，如果"Paris"和"3"分别表示地名和数字，那这两个就无法表达“巴黎第三大学”的语义。而中文也有类似的例子，“北京大学”的”北京“和”大学“都可以作为一个最小单位来使用，分别表示”地方名“和“大学”，如果这样分词，那么就可以理解为“北京的大学”了，所以“北京大学”是一个表达语义的最小单位。前几年有部电影《夏洛特烦恼》，我们是要理解为“夏洛特 烦恼“还是”夏洛 特 烦恼“，这就是很经典的分词问题。

但是从语用角度，这些问题似乎能被解决，我们知道"pomme de terre"在日常生活中一般作为“土豆”而不是“土里的苹果”，在巴黎学习都知道“Paris 3”，就像我们提到“北京大学”特指那所著名的高等学府一样。看过电影《夏洛特烦恼》的观众很容易的就能区分这个标题应该看为“夏洛 特 烦恼”。

发现新词的方法，《[互联网时代的社会语言学：基于SNS的文本数据挖掘](http://www.matrix67.com/blog/archives/5044]) 》一文，里面提到的给每一个文本串计算**文本片段**的**凝固程度**和文本串对外的使用**自由度**，通过设定阈值来将文本串分类为词和非词两类。原文给了十分通俗易懂的例子来解释凝固度和自动度。这里放上计算方法。这个方法还有许多地方需要优化，在之后的实践中慢慢调整了。

## 文本片段

**文本片段**，最常用的方法就是[n元语法(ngram)](https://zh.wikipedia.org/wiki/N元语法)，将分本分成多个n长度的文本片段。数据结构，这里采用Trie树的方案，这个方案是简单容易实现，而且用Python的字典做Hash索引实现起来也很优美，唯独的一个问题是所有的数据都存在内存中，这会使得内存占用量非常大，如果要把这个工程化使用，还需要采用其他方案，比如硬盘检索。
<a href="https://upload.wikimedia.org/wikipedia/commons/b/be/Trie_example.svg
" target="_blank"><img src="https://upload.wikimedia.org/wikipedia/commons/b/be/Trie_example.svg" 
alt="IMAGE ALT TEXT HERE" width="240" height="180" border="10" /></a>


```python
class TrieNode(object):
    def __init__(self, 
                 frequence=0, 
                 children_frequence=0, 
                 parent=None):

        self.parent = parent
        self.frequence = frequence
        self.children = {} 
        self.children_frequence = children_frequence

    def insert(self, char):
        self.children_frequence += 1
        self.children[char] = self.children.get(char, TrieNode(parent=self))
        self.children[char].frequence += 1
        return  self.children[char]
        
    def fetch(self, char):
        return self.children[char]
    
class TrieTree(object):
    def __init__(self, size=6):
        self._root = TrieNode()
        self.size = size
        
    def get_root(self):
        return self._root
    
    def insert(self, chunk):
        node = self._root
        for char in chunk:
            node = node.insert(char)
        if len(chunk) < self.size:
            # add symbol "EOS" at end of line trunck
            node.insert("EOS")

    def fetch(self, chunk):
        node = self._root
        for char in chunk:
            node = node.fetch(char)
        return node
```

Trie树的结构上，我添加了几个参数，parent，frequence，children_frequence，他们分别是：
- parent，当前节点的父节点，如果是“树根”的时候，这个父节点为空；
- frequence，当前节点出现的频次，在Trie树上，也可以表示某个文本片段的频次，比如"中国"，“国”这个节点的frequence是100的时候，“中国”俩字也出现了100次。这个可以作为最后的词频过滤用。
- children_frequence，当前接点下有子节点的"frequence"的总和。比如在刚才的例子上加上“中间”出现了99次，那么“中”这个节点的children_frequence的值是199次。
这样的构造让第二部分的计算更加方面。

这个任务中需要构建两棵Trie树，表示正向和反向两个字符片段集。

## 自由度

**自由度**，使用信息论中的[信息熵](https://zh.wikipedia.org/wiki/熵_(信息论))构建文本片段左右熵，公式[1]。熵越大，表示该片段和左右邻字符相互关系的不稳定性越高，那么越有可能作为独立的片段使用。公式[1]第一个等号后面的I(x)表示x的自信息。
\[
\begin{align} 
 H(X) = \sum_{i} {\mathrm{P}(x_i)\,\mathrm{I}(x_i)} = -\sum_{i} {\mathrm{P}(x_i) \log \mathrm{P}(x_i)} [1]
\end{align}
\]



```python
def calc_entropy(chunks, ngram):
    """计算信息熵
    Args:
        chunks，是所有数据的文本片段
        ngram，是Trie树
    Return:
        word2entropy，返回一个包含每个chunk和对应信息熵的字典。
    """
    def entropy(sample, total):
        """Entropy"""
        s = float(sample)
        t = float(total)
        result = - s/t * math.log(s/t)
        return result

    def parse(chunk, ngram):
        node = ngram.fetch(chunk)
        total = node.children_frequence
        return sum([entropy(sub_node.frequence, 
                           total) for sub_node in node.children.values()])

    word2entropy = {}
    for chunk in chunks:
        word2entropy[chunk] = parse(chunk, ngram)   
    return word2entropy
```

## 凝固度

**凝固度**，用信息论中的**互信息**表示，公式[2]。在概率论中，如果x跟y不相关，则p(x,y)=p(x)p(y)。二者相关性越大，则p(x,y)就相比于p(x)p(y)越大。用后面的式子可能更好理解，在y出现的情况下x出现的条件概率p(x|y)除以x本身出现的概率p(x)，自然就表示x跟y的相关程度。 
\begin{align} 
I(x;y) = \log\frac{p(x,y)}{p(x)p(y)} = \log\frac{p(x|y)}{p(x)} = \log\frac{p(y|x)}{p(y)} [2]
\end{align}

这里比较容易产生一个概念的混淆，维基百科将式[2]定义为[点互信息](https://en.wikipedia.org/wiki/Pointwise_mutual_information)，[互信息](https://zh.wikipedia.org/wiki/互信息)的定义如下：
\begin{align} 
I(X;Y) = \sum_{y \in Y} \sum_{x \in X} 
                 p(x,y) \log{ \left(\frac{p(x,y)}{p(x)\,p(y)}
                              \right) }\ [3]
\end{align}
在傅祖芸编著的《信息论——基础理论与应用（第4版）》的绪论中，把式[2]定义为互信息，而式[3]定义为平均互信息，就像信息熵指的是**平均自信息**。


```python
    def calc_mutualinfo(self, chunks, ngram):
        """计算互信息
        Args:
            chunks，是所有数据的文本片段
            ngram，是Trie树
        Return:
            word2mutualinfo，返回一个包含每个chunk和对应互信息的字典。
        """
        def parse(chunk, root):
            sub_node_y_x = ngram.fetch(chunk)
            node = sub_node_y_x.parent
            sub_node_y = root.children[chunk[-1]]
            
            # 这里采用互信息log(p(y|x)/p(y))的计算方法
            prob_y_x = float(sub_node_y_x.frequence) / node.children_frequence
            prob_y = float(sub_node_y.frequence) / root.children_frequence
            mutualinfo = math.log(prob_y_x / prob_y)
            return mutualinfo, sub_node_y_x.frequence
        
        word2mutualinfo = {}  
        root = ngram.get_root()
        for chunk in chunks:
            word2mutualinfo[chunk] = parse(chunk, root)
        return word2mutualinfo
```

## 过滤

最终计算得出互信息、信息熵，甚至也统计了词频，最后一步就是根据阈值对词进行过滤。


```python
def _fetch_final(fw_entropy，
                 bw_entropy,
                 fw_mi,
                 bw_mi
                 entropy_threshold=0.8,
                 mutualinfo_threshold=7,
                 freq_threshold=10):
        final = {}
        for k, v in fw_entropy.items():
            last_node = self.fw_ngram
            if k[::-1] in bw_mi and k in fw_mi:
                mi_min = min(fw_mi[k][0], bw_mi[k[::-1]][0])
                word_prob = min(fw_mi[k][1], bw_mi[k[::-1]][1])
                if mi_min < mutualinfo_threshold:
                    continue
            else:
                continue
            if word_prob < freq_threshold:
                 continue
            if k[::-1] in bw_entropy:
                en_min = min(v, bw_entropy[k[::-1]])
                if en_min < entropy_threshold:
                    continue
            else:
                continue
            final[k] = (word_prob, mi_min, en_min)
        return final
```

## 结果

最终，通过这个方法对这次十九大的开幕发言做的一个词汇发现，ngram的n=10，结果按词频排序输出，可以发现这次十九大谈了许多内容，不一一说了。这个结果还存在不少问题，比如“二〇”，这在阈值的设置上还不够准确，可以尝试使用机器学习的方法来获取阈值。

经济|70 改革|69 我们|64 必须|61 领导|60 完善|57 历史|44 不断|43 群众|43 教育|43 战略|42 思想|40 世界|39 问题|37 提高|37 组织|36 监督|35 加快|35 依法|34 精神|33 团结|33 复兴|32 保障|31 奋斗|30 根本|29 环境|29 军队|29 开放|27 服务|27 理论|26 干部|26 创造|26 基础|25 意识|25 维护|25 协商|24 解决|24 贯彻|23 斗争|23 目标|21 统筹|20 始终|19 方式|19 水平|19 科学|19 利益|19 市场|19 基层|19 积极|18 马克思|18 反对|18 道路|18 自然|18 增长|17 科技|17 稳定|17 原则|17 两岸|17 取得|16 质量|16 农村|16 矛盾|16 协调|15 巩固|15 收入|15 绿色|15 自觉|15 方针|15 纪律|15 长期|15 保证|15 同胞|15 命运|14 美好生活|14 五年|14 传统|14 繁荣|14 没有|14 使命|13 广泛|13 日益|13 价值|13 健康|13 资源|13 参与|13 突出|13 腐败|13 充分|13 梦想|13 任何|13 二〇|13 代表|12 阶段|12 深刻|12 布局|12 区域|12 贸易|12 核心|12 城乡|12 生态文明|12 工程|12 任务|12 地区|12 责任|12 认识|12 胜利|11 贡献|11 覆盖|11 生态环境|11 具有|11 面临|11 各种|11 培育|11 企业|11 继续|10 团结带领|10 提升|10 明显|10 弘扬|10 脱贫|10 贫困|10 标准|10 注重|10 基本实现|10 培养|10 青年|10


## 代码下载地址

git clone https://github.com/KunFly/new-word-discovery.git
