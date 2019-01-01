=====================
用aiohttp写异步爬虫
=====================

| `aiohttp`__ 可以用来写异步爬虫以及作为异步服务器，性能会比同步的模块提高不少。
| 本篇介绍如何用 ``aiohttp`` 写异步爬虫。

.. __: https://docs.aiohttp.org

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
| ``esp.text()`` 有1个参数：``encoding``： 

   - ``encoding=None`` 是默认参数，这时相当于上面的 ``resp.encoding = resp.apparent_encoding``\ ，会检测网页的编码。如果结果不对出现乱码，再才需要明确正确的编码：``resp.text(encoding='utf8')``\ 。另外，已知正确的编码也可以显式声明，因为检测编码有点影响性能。

.. note:: 从 ``requests`` 源码可知，检测编码用的是 ``chardet`` 模块：``encoding = chardet.detect(content)['encoding']`` 。

| 有大量url时，每个都开一个会话显然是不好的，于是会写成这样：

.. code::

    import asyncio

    import aiohttp


    async def fetch(session, id_, url):
        headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.100 Safari/537.36'}
        try:
            async with session.get(url, headers=headers, verify_ssl=False) as resp:
                text, content = await resp.text(), await resp.read()
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

# todo
