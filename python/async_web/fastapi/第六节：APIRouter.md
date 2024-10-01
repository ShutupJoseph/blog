# 第六节：APIRouter

在`FastAPI`项目中，我们不可能把所有视图的代码都放到`main.py`文件中，这时候就需要将视图进行分类，然后同类的放到一个单独的子路由下。在`FastAPI`中可以使用`APIRouter`类来实现。

示例代码如下：

```python
from fastapi import APIRouter
from typing import List
from schemas import UserResp

router = APIRouter(prefix='/myuser', tags=['user'])


@router.get('/list', response_model=List[UserResp])
async def user_list(page: int=1):
    return [
        UserResp(id=1, username='zhangsan', email='hy@qq.com')
    ]
```

然后在`main.py`中通过`include_router`加载，示例代码如下：

```python
from fastapi import FastAPI
from routers import users, items

app = FastAPI()

app.include_router(users.router)
app.include_router(items.router)
```



> 原文: <https://www.yuque.com/hynever/wms8gi/goc0rb2g6a5fhrzv>