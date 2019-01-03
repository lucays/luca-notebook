=====================
用aiohttp写爬虫
=====================

| `aiohttp`__ 可以用来写异步爬虫以及作为异步服务器，性能会比同步的模块提高不少。
| 本篇介绍如何用 ``aiohttp`` 写异步爬虫。

.. __: https://docs.aiohttp.org

基本介绍
==========

首先，来看看 ``requests`` 是如何写爬虫的：

.. code::

    import requests
    

    def fetch(url):
        '''
        url: 要请求的网站url
        '''
        headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.100 Safari/537.36'}  # 请求时携带的header
        resp = requests.get(url, headers=headers, verify=False)  # verify忽略https的ssl验证
        resp.encoding = resp.apparent_encoding  # 猜测编码，防乱码
        return resp.text
    

    fetch('https://www.baidu.com')

把上述代码用 ``aiohttp`` 改写：

.. code:: Python

    import asyncio

    import aiohttp


    async def fetch(url):
        headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.100 Safari/537.36'}
        async with aiohttp.ClientSession() as session:
            async with session.get(url, headers=headers, verify_ssl=False) as resp:
                return await resp.text(errors='ignore')
    

    loop = asyncio.get_event_loop()
    loop.run_until_complete(fetch('https://www.baidu.com'))
    loop.close()

| 以上是基本的改写。
| ``resp.text()`` 有1个参数：``encoding``： 

- ``encoding=None`` 是默认参数，这时相当于上面的 ``resp.encoding = resp.apparent_encoding``\ ，会检测网页的编码。如果结果不对出现乱码，再才需要明确正确的编码：``resp.text(encoding='utf8')``\ 。另外，已知正确的编码也可以显式声明，因为检测编码有点影响性能。

.. note:: 从 ``requests`` 源码可知，检测编码用的是 ``chardet`` 模块：``encoding = chardet.detect(content)['encoding']`` 。

批量url爬取
==============

| 有大量url时，每个都开一个会话显然是不好的，于是会写成这样：

.. code::

    import asyncio

    import aiohttp


    async def fetch(session, id_, url):
        headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.100 Safari/537.36'}
        try:
            async with session.get(url, headers=headers, verify_ssl=False) as resp:
                return await resp.text(), await resp.read()
        except Exception:
            print(f"{id_}, url: {url} error happened:")


    async def fetch_all(urls):
        '''
        urls: list[(id_, url)]
        '''
        async with aiohttp.ClientSession() as session:
            datas = await asyncio.gather(*[fetch(session, id_, url) for id_, url in urls], return_exceptions=True)
            # return_exceptions=True 可知从哪个url抛出的异常
            for ind, data in enumerate(urls):
                id_, url = data
                if isinstance(datas[ind], Exception):
                    print(f"{id_}, {url}: ERROR")
            return datas


    urls = [(i, f'https://www.baidu.com/?tn={i}') for i in range(100)]
    loop = asyncio.get_event_loop()
    loop.run_until_complete(fetch_all(urls))
    loop.close()

错误处理
==========

如果url太多，可能会报错 ``ValueError: too many file descriptors in select()`` ，根据 `stackoverflow`__ 所述，``aiohttp`` 默认设置中一次可以打开100个连接，而Windows一次最多只能打开64个 ``socket``，所以可以在 ``fetch_all`` 中添加一行：

.. __: https://stackoverflow.com/questions/47675410/python-asyncio-aiohttp-valueerror-too-many-file-descriptors-in-select-on-win

.. note:: `这篇文章 <https://blog.magentaize.net/fix-python-too-many-file-descriptors-in-select-in-windows/>`__ 指出应该是 ``Python`` 的锅，限制了并发数最多为512。

.. code::

    connector = aiohttp.TCPConnector(limit=60)  # 60小于64。也可以改成其他数
    async with aiohttp.ClientSession(connector=connector) as session:
        ...

另外，也可以用回调解决这个问题。

回调
=======

对获取的html用 ``lxml`` 等处理时，可以使用回调。上述代码中，添加如下处理函数：

.. code::

    from lxml import etree


    def get_result(future):
        text, content = future.result()  # 调用future.result()获取返回值
        html = etree.HTML(text)
        for i in html.xpath('//h3/a'):
            print(i.xpath('string(.)'), i.xpath('@href')[0])

之后需要改写 ``fetch_all`` 函数：

.. code::

    async def fetch_all(urls):
        '''
        urls: list[(id_, url)]
        '''
        async with aiohttp.ClientSession() as session:
            tasks = []
            for id_, url in urls:
                # 在Python3.7+，asyncio.ensure_future() 改名为 asyncio.create_task()
                task = asyncio.ensure_future(fetch(session, id_, url))
                task.add_done_callback(get_result)
                tasks.append(task)
            datas = await asyncio.gather(*tasks, return_exceptions=True)
            # return_exceptions=True 可知从哪个url抛出的异常
            for ind, data in enumerate(urls):
                id_, url = data
                if isinstance(datas[ind], Exception):
                    print(f"{id_}, {url}: ERROR")
            return datas

.. note:: 对于上面的 错误处理_ 的解决办法，直接在for循环中这么写：

   .. code::

    async def fetch_all(urls, loop):
        '''
        urls: list[(id_, url)]
        '''
        async with aiohttp.ClientSession() as session:
            for id_, url in urls:
                # 在Python3.7+，asyncio.ensure_future() 改名为 asyncio.create_task()
                task = asyncio.ensure_future(fetch(session, id_, url))
                task.add_done_callback(get_result)
                loop.run_until_complete(task)
            return datas
    

    urls = [(i, f'https://www.baidu.com/s?wd=python&pn={10*i}') for i in range(2000)]
    loop = asyncio.get_event_loop()
    fetch_all(urls, loop)
    loop.close()

在Python官方文档中，`add_done_callback`__ 应当仅在底层代码中使用。即使 ``future`` 抛出异常，也会 ``callback``，让异常在 ``future.result()`` 处抛出。并且给这个函数传递参数也不太方便。

.. __: https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.add_done_callback

那么，我们可以自己动手写一个回调函数，也就是改一改上面的回调代码：

.. code::

    from lxml import etree


    def get_result(data):
        text, content = data
        html = etree.HTML(text)
        for i in html.xpath('//h3/a'):
            print(i.xpath('string(.)'), i.xpath('@href')[0])


    async def add_success_callback(future, callback):
        result = await future  # 注意自己写就不是用future.result()这个接口了
        callback(result)


    async def fetch_all(urls):
        '''
        urls: list[(id_, url)]
        '''
        async with aiohttp.ClientSession() as session:
            tasks = []
            for id_, url in urls:
                # 在Python3.7+，asyncio.ensure_future() 改名为 asyncio.create_task()
                task = asyncio.ensure_future(fetch(session, id_, url))
                task = add_success_callback(task, get_result)
                tasks.append(task)
            datas = await asyncio.gather(*tasks, return_exceptions=True)
            # return_exceptions=True 可知从哪个url抛出的异常
            for ind, data in enumerate(urls):
                id_, url = data
                if isinstance(datas[ind], Exception):
                    print(f"{id_}, {url}: ERROR")
            return datas

如果不使用回调，那么可以写进 ``fetch`` 函数中：

.. code::

    async def fetch(session, id_, url):
        headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.100 Safari/537.36'}
        try:
            async with session.get(url, headers=headers, verify_ssl=False) as resp:
                text, content = await resp.text(), await resp.read()
                html = etree.HTML(text)
                for i in html.xpath('//h3/a'):
                    print(i.xpath('string(.)'), i.xpath('@href')[0])
        except Exception:
            print(f"{id_}, url: {url} error happened:")

不过，这样会模糊 ``fetch`` 函数的功能。
