=====================
用aiohttp写服务器
=====================

| 用\ ``django``\ /\ ``flask``\ 写服务器的文章太多了，这里记录下自己用\ ``aiohttp``\ 写的过程好了。
| 部分内容直接翻译自\ `aiohttp文档`__\ 。

.. __: https://aiohttp.readthedocs.io/en/stable/web_quickstart.html

| 要实现一个服务器，首先要创建请求处理程序
| 既然是用\ ``aiohttp``\ 实现，那么这个程序就得是一个协程，来接受一个\ ``request``\ ，返回一个\ ``response``\ 。

.. code::

    from aiohttp import web


    async def hello(request):
        return web.Response(text='Hello, world')

由于一个服务器必须能处理\ ``url``\ ，我们在这段代码下面创建一个\ ``app``\ 绑定要处理的\ ``url``\ 并运行：

.. code::

    app = web.Application()
    app.add_routes([web.get('/', hello)])  # 绑定hello函数
    web.run_app(app)

运行这段代码，然后在浏览器打开\ http://localhost:8080/\ 会看到网页显示\ ``Hello, world``\ 。

如果喜欢\ ``flask``\ 那样的路由，用装饰器绑定，也可以写成这样：

.. code::

    from aiohttp import web

    routes = web.RouteTableDef()


    @routes.get('/')
    async def hello(request):
        return web.Response(text="Hello, world")


    app = web.Application()
    app.add_routes(routes)
    web.run_app(app)

两段代码结果完全相同。喜欢什么，就写成那样就好了。

| 对于第一种写法，有多个路由时，由于\ ``aiohttp``\ 会合并同一路径的所有后续资源添加，为所有的\ ``HTTP``\ 方法添加唯一的资源，所以添加顺序对性能优化有一些影响。
| 例如下面2种写法：

.. code::

    app.add_routes([web.get('/path1', get_1),
                    web.post('/path1', post_1),
                    web.get('/path2', get_2),
                    web.post('/path2', post_2)]

和

.. code::

    app.add_routes([web.get('/path1', get_1),
                    web.get('/path2', get_2),
                    web.post('/path2', post_2),
                    web.post('/path1', post_1)]

第1种先添加同一路径的所有方法的写法，会更好一些。

文档是这样写的，那么我们看看源码是怎么实现的：

.. image:: ../_static/c01/2-1.jpg

| 860~864行的代表表明，如果\ ``self._resources``\ 末尾元素的\ ``name``\ 和要添加的一样，那么直接返回这个元素，不会添加。
| 直接使用已有的资源，没有新建（再运行865~872行的代码），自然性能就好一些。
| 简单加个\ ``test``\ 函数绑定\ ``/test``\ ，都再加上\ ``post``\ 方法，然后\ ``print()``\ 一下\ ``self._resources``\ ，第1种写法的结果：

.. image:: ../_static/c01/2-2.jpg

| 可以看出，添加\ ``post``\ 方法时直接用了已有的资源。
| 再看第2种写法：

.. image:: ../_static/c01/2-3.jpg

| 每次都创建了新的资源，自然性能就损失了一些。
