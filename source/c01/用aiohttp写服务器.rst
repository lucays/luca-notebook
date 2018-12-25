=====================
用aiohttp写异步爬虫
=====================

| ``aiohttp`` 可以用来写异步爬虫以及作为异步服务器，性能会比同步的模块提高不少。
| 本篇介绍如何用 ``aiohttp`` 写异步爬虫。

首先，来看看 ``requests`` 是如何写爬虫的：

.. code:: Python

    import requests
    
    def get_html(url):
        '''
        url: 要请求的网站url
        '''
        headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.100 Safari/537.36'}  # 请求时携带的header
        resp = requests.get(url, headers=headers, verify=False)  # verify忽略https的ssl验证
        resp.encoding = resp.apparent_encoding  # 猜测编码，防乱码
        return resp.text
    
    get_html('https://www.baidu.com')

把上述代码用 ``aiohttp`` 改写：

.. code:: Python

    import asyncio

    import aiohttp

    async def get_html(url):
        headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.100 Safari/537.36'}
        async with aiohttp.ClientSession() as session:
            async with session.get(url, headers=headers, verify_ssl=False) as resp:
                return await resp.text(errors='ignore')
    
    loop = asyncio.get_event_loop()
    loop.run_until_complete(get_html('https://www.baidu.com'))
    loop.close()
    
# todo...
