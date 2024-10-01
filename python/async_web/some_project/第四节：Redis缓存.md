# 第四节：Redis缓存

## 一、安装
Redis服务端的安装，可参考[Django+Vue+Docker企业OA系统](https://study.163.com/course/courseMain.htm?courseId=1213746817&share=2&shareId=1025897964)。

这里说一下Python操作redis的包。我们使用一个支持异步的redis客户端，叫做`redis-py`。安装命令如下：

```shell
$ pip install "redis[hiredis]"
```

更多的文档请参考：[https://github.com/redis/redis-py](https://github.com/redis/redis-py)

redis-py异步支持：[https://redis-py.readthedocs.io/en/v5.0.1/examples/asyncio_examples.html](https://redis-py.readthedocs.io/en/v5.0.1/examples/asyncio_examples.html)





> 原文: <https://www.yuque.com/hynever/shtqfp/hkfxn1hek57yarqn>