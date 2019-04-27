===========================
用gensim判断文本相似度
===========================

| 判断文本的相似度可用于识别不同网站发的文章是否相同（转发或通稿等），只对不相似的文章进一步处理可大大提高效率。
| \ ``Python``\ 中可以使用\ `gensim <https://radimrehurek.com/gensim/index.html>`__\ 库来判断。
| 官方的文档中\ `入门部分 <https://radimrehurek.com/gensim/tutorial.html>`__\ 写的实在不怎样，这里记录一下自己踩坑的经历。

清洗数据
=========

很多文章的开头，结尾等会有一些没用的文字或句子，需要去掉：

.. code::

    from lxml import etree
    from lxml.html import clean

    cleaner = clean.Cleaner(style=True, javascript=True, scripts=True, page_structure=False, safe_attrs_only=False, kill_tags=['span'])

    del_words = {
        '编辑', '责编', '记者 ', '摘要：', '摘要 ', '风险自担', '风险请自担', '想了解更多关于', '扫码下载', '（原题为', '依法追究', '严正声明', "新浪股民维权平台", "你投诉，我报道", "文字内容参考：",
        '关键词 ', '2018-', '原标题', '原文', '概不承担', '转载自', '来源：', '仅做参考', '仅供参考', '未经授权', '浏览更多', '金融曝光台', '财经讯', '在线投诉', "并不预示其未来业绩表现",
        '禁止转载', '阅后点赞', '研究员：', '本文首发', '微信公众号', '个人观点', '蓝字关注', '微信号：', '欢迎订阅', '点击右上角分享', '加入我们', '二维码转账', '赞赏功能', "投资需谨慎", '黑猫投诉平台',
        '热门搜索', '为您推荐', '更多评论', '文明上网', '来源:', '作者:', '扫描二维码', '在线咨询：', '扫描或点击关注', '中金在线', '长按二维码', 'Scan QR Code via WeChat', '推荐阅读',
    }
    ending_words = {'长按二维码向我转账', '已推荐到看一看', "风险提示：", "免责声明："}

    def filter_words(sentences: list) ->list:
        '''
        过滤文章中包含无用词的语句

        :param sentences: 由字符串组成的list

        '''
        text = []
        signal = True
        for sentence in sentences:
            # 如果ending_words中的词语出现了，说明已经到结尾部分，之后的所有内容都丢弃。
            for word in ending_words:
                if word in sentence:
                    signal = False
                    break
            if not signal:
                break
            if sentence.strip() and not [word for word in del_words if word in sentence]:
                text.append(sentence.strip())
        return text

    def text_extract(content: str) -> str:
        '''
        如果存储的是html，用lxml提取纯文本
        '''
        if '<' in content and '>' in content and '</' in content:
            if '<html>' in content:
                content = cleaner.clean_html(content)
            html = etree.HTML(content)
            sentences = []
            for i in html.xpath('//text()'):
                if i.strip():
                    sentences.extend(i.strip().split())

            sentences = ''.join(sentences).split('。')
        else:
            sentences = [''.join(sentence.split()) for sentence in content.split('。')]

        return ' '.join(filter_words(sentences)).strip()

假定我们的原始数据是数据库游标\ ``cur``\ ，每次迭代出\ ``(id_, content)``\ 的话，就可以这样处理来获取有效文本：

.. code::

    contents = []
    for id_, content in cur:
        text = text_extract(content)
        # 文章内容为<html><body/></html>或者为空的丢弃
        if len(text) > 25:
            contents.append((id_, text))  # contents即最终的有效文本组成的list

分词
======

这里采用的\ ``jieba``\ 分词：

.. code::

    def tokenization(content: str) -> list:
        '''
        去除文章中特定词性的词

        '''
        # {标点符号、连词、助词、副词、介词、时语素、‘的’、数词、方位词、代词}
        # {'x', 'c', 'u', 'd', 'p', 't', 'uj', 'm', 'f', 'r'}
        stop_flags = {'x', 'c', 'u', 'd', 'p', 't', 'uj', 'm', 'f', 'r'}
        stop_words = {'nbsp', '\u3000', '\xa0'}
        words = pseg.cut(content)
        return [word for word, flag in words if flag not in stop_flags and word not in stop_words]

相似度判断
============

为了把文章转化成向量表示，这里使用词袋表示，具体来说就是每个词出现的次数。连接词和次数就用字典表示。然后，用\ ``doc2bow()``\ 统计词语的出现次数。

.. code::

    from gensim import corpora, models, similarities

    def get_dic_corpus(contents):
        texts = [tokenization(r) for id_, r in contents]

        dic = corpora.Dictionary(texts)
        corpus = [dic.doc2bow(text) for text in texts]
        return dic, corpus

准备需要对比相似度的文章

.. code::

    # 假定用contents的第1篇文章对比，由于contents中每个元素由id和content组成，所以contents[0][1]是第一篇文章的content
    new_doc = contents[0][1]
    new_vec = dic.doc2bow(tokenization(new_doc))

官方文档入门部分给的模型是\ ``tf-idf``\ 模型：

.. code::

    tfidf = models.TfidfModel(corpus)  # 建立tf-idf模型
    index = similarities.MatrixSimilarity(tfidf[corpus], num_features=12)  # 对整个语料库进行转换并编入索引，准备相似性查询
    sims = index[tfidf[new_vec]]  # 查询每个文档的相似度
    print(list(enumerate(sims)))  # [(0, 1.0), (1, 0.19139354), (2, 0.24600551), (3, 0.82094586), (4, 0.0), (5, 0.0), (6, 0.0), (7, 0.0), (8, 0.0)]

上面的结果中，每个值由编号和相似度组成，例如，编号为0的文章与第1篇文章相似度为100%。(因为就是第一篇）

以上就是通过官方文档的入门示例判断中文文本相似度的基本代码，由于某些原因，结果可能为负值或大于1，暂且忽略，这不是重点。

改进
======

| 上面的例子中，\ ``num_features``\ 的取值需要注意，官方文档没有解释为什么是12，在大批量的判断时还使用12就会报错。实际上应该是\ ``num_features=len(dictionary)``\ 。
| 此外，这个模型的准确度实在是令人堪忧，不知道为什么官方使用其作为入门示例，浅尝辄止的话，可能就误以为现在的技术还达不到令人满意的程度。

下面我们换成lsi模型，实际表现很好：

.. code::

    NUM_TOPICS = 350

    lsi = models.LsiModel(corpus, id2word=dic, num_topics=NUM_TOPICS)
    index = similarities.MatrixSimilarity(lsi[corpus])
    sims = index[lsi[new_vec]]
    res = list(enumerate(sims))
    res = sorted(res, key=lambda x: x[1])  # 按相似度大小排序
    print(res)

其实只是换了模型名称而已，但还要注意几个点：

- 官方文档中\ ``LsiModel()``\ 参数用的是\ ``tfidf[corpus]``\ ，实测会导致部分结果不对。
- 官方文档中最初用的\ ``num_topics=2``\ ，后面又介绍了这个值最好在200-500之间即可。

| 但是这样也有问题，这只能判断单篇的结果，其他文章再对比的话，要用\ ``for``\ 循环一篇篇对比吗？
| 此外，显然这个方法是把文章都存在内存中，如果文章很多，每篇又很长，很容易挤爆内存。
| 众所周知，\ ``Python``\ 的\ ``for``\ 循环效率很低。所以，不要这样做。
| \ ``gensim``\ 提供了一个\ `Similarity类 <https://radimrehurek.com/gensim/similarities/docsim.html>`__\ ，来本地化存储所有文章并直接互相对比，这也是我真正最后使用的方法。

| 原文很多地方云里雾里的，比如最基础的这个\ ``similarities.Similarity``\ 的参数，\ ``get_tmpfile("index")``\ 是什么都没讲。
| 实际使用相当简单：

.. code::

    index = similarities.Similarity('index', lsi[corpus], num_features=lsi.num_topics)
    for i in enumerate(index):
        print(i)   # 输出对整组的相似度
    # 或者，直接输出文章id分组

    percentage = 0.95  # 如果判断结果为95%相似，那么划为一组
    for l, degrees in enumerate(index):
        print(contents[l][0], [contents[i][0] for i, similarity in enumerate(degrees) if similarity >= percentage])

| 上述代码中\ ``num_features=lsi.num_topics``\ ，原文给定的是\ ``num_features=len(dictionary)``\ ，在实际使用中，容易出错：\ ``mismatch between supplied and computed number of non-zeros``\ 。
| 这样改是参照了\ `这里 <https://groups.google.com/forum/#!topic/gensim/zYmQGxADztU>`__\ 的做法。文档给出的示例是tf-idf模型下的结果，在lsi模型下就因为传递的数据不对而可能出错。

.. note::

    这样改完仍然有小概率报错，可以捕获异常改用\ ``tf-idf``\ 模型判断就不会报错了，只不过准确率也下去了：

    .. code::

        def similar(contents, dic, corpus, percentage) -> dict:
            '''
            得出相似度
            '''
            lsi = models.LsiModel(corpus, id2word=dic, num_topics=NUM_TOPICS)
            # index = similarities.MatrixSimilarity(lsi[corpus])

            index = similarities.Similarity('index', lsi[corpus], num_features=lsi.num_topics)
            res = {}
            for l, degrees in enumerate(index):
                res[contents[l][0]] = [contents[i][0] for i, similarity in enumerate(degrees) if similarity >= percentage]
            return res

        try:
            res = similar(contents, dic, corpus, percentage)
        except AssertionError:
            corpus = models.TfidfModel(corpus)
            res = similar(contents, dic, corpus, percentage)

接口示例
==========

简单地，用\ ``flask``\ 写个接口作为示例：

.. code::

    from flask import Flask, request

    app = Flask(__name__)


    @app.route('/similar', methods=['POST'])
    def similar_lst():
        if request.method == 'POST':
            ids = request.form.get('ids')
            ids = [int(i.strip()) for i in ids.split(',')]
            if ids:
                percentage = float(request.form.get('percentage'))
                contents = get_content(ids)  # 从数据库获取这些id对应的content，这步略去
                try:
                    res = similar(contents, dic, corpus, percentage)
                except AssertionError:
                    '''
                    def tfidf_model(corpus):
                        return models.TfidfModel(corpus)[corpus]
                    '''
                    corpus = tfidf_model(corpus)
                    res = similar(contents, dic, corpus, percentage)
                return json.dumps(res)


    if __name__ == '__main__':
        # main()
        app.run(host='0.0.0.0', port=80)

最后
======

这篇文章只是完成了一个文本判断的雏形，算是可以使用的地步而已，还可以对停用词做文件配置等来进一步优化处理。
