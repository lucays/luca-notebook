======================================
用scrapy+aiohttp+aioredis实现代理池
======================================

| 首先给出代码地址：`github/lucays <https://github.com/lucays/IPProxy-Spider>`__。
| 其实就是把《Python3网络爬虫实战》的代理池实现中的\ ``flask+redis``\ 改成了异步模式\ ``aiohttp+aioredis``\ 。

| \ ``scrapy``\ 用来爬取代理，\ ``aiohttp``\ 做服务端，存储用\ ``redis``\ 。
| \ ``scrapy``\ 部分本身没什么特别的，不过\ ``pipeline``\ 部分最后采取了共用\ ``aioredis``\ 实现的\ ``RedisClient``\ 的写法。
| 而这个类改写时，由于\ ``__init__(self)``\ 不能是协程，最后采用以下代码实现：

.. code::

    @classmethod
    async def create(cls):
        """
        __init__不能是协程，写成工厂函数
        """
        self = cls()
        self.pool = await aioredis.create_redis_pool(address=f"redis://{REDIS_HOST}", password=REDIS_PASSWORD, encoding='utf-8')
        self.conn = await self.pool
        return self

使用方法也在这个文件中附了1个\ ``main``\ 函数作为示例：

.. code::

    async def main():
        r = await RedisClient.create()
        counts = await r.get_count()
        print(counts)
        return counts

    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())

| 服务端部分全部用\ ``aiohttp``\ 基于类的视图进行改写，也没特别值得注意的地方。
| 后台定时任务中，为了能用\ ``subprocess``\ 调用另一个文件夹的\ ``scrapy``\ 项目，使用了一个\ ``cd``\ 类改变工作区文件夹路径：

.. code::

    class cd:
        """
        改变目前工作区文件夹路径的上下文管理器
        """
        def __init__(self, newPath):
            self.newPath = os.path.expanduser(newPath)

        def __enter__(self):
            self.savedPath = os.getcwd()
            os.chdir(self.newPath)

        def __exit__(self, etype, value, traceback):
            os.chdir(self.savedPath)

使用起来很简单：

.. code::

    with cd("../"):
        # 现在进入了上一层目录
        pass
