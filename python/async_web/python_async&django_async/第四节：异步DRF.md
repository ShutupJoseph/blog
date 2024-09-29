# 第四节：异步DRF

## 一、介绍
Django Rest Framework作为一款专为Django打造的提供API服务的框架，已经是Django开发者必不可少的框架了，但是这款框架目前是不支持异步的。如果想要在异步Django中使用DRF，那么需要借助一个插件adrf 。

adrf 的github地址为：https://github.com/em1208/adrf，支持异步权限验证、异步限速节流、异步视图、异步视图集、异步序列化等功能。以下分别进行讲解。

## 二、安装和配置
要想使用adrf ，在已经安装了django和djangorestframework的基础之上，通过以下命令即可安装adrf ：

```bash
$ pip install adrf==0.1.6
```

然后在项目的settings.py 中添加adrf ：

```python
INSTALLED_APPS = [
    ...
    'adrf',
]
```

## 三、异步API视图
### 1. APIView
要使用异步的API视图，集成时需要从之前rest_framework.views.APIView ，修改为adrf.views.APIView ，其他其他写法与用rest_framework 类似，示例代码如下：

```python
from adrf.views import APIView
from django.http.response import JsonResponse

class IndexView(APIView):
    async def get(self, request):
        return JsonResponse({"message": "请求成功！"})

class MessageView(APIView):
    async def post(self, request):
        title = request.data.get('title')
        content = request.data.get('content')
        email = request.data.get('email')
        print(title, content, email)
        return JsonResponse({"message": "感谢反馈！留言消息已收到！我们将尽快处理！"})
```

如果使用的是函数视图，那么可以通过adrf.decorators.api_view 装饰器来实现异步视图：

```python
from adrf.decorators import api_view

@api_view(['GET'])
async def async_view(request):
    return Response({"message": "This is an async function based view."})
```

### 2. authentication
如要实现自己的验证用户的操作，那么可以通过继承自rest_framework.authentications.BaseAuthentication ，然后将继承后的authenticate 方法写成异步的即可。示例代码如下：

```python
from rest_framework.authentication import BaseAuthentication, get_authorization_header
from rest_framework import exceptions
from django.contrib.auth import get_user_model

User = get_user_model()

class AsyncAuthentication(BaseAuthentication):

    async def authenticate(self, request):
        auth = get_authorization_header(request).split()
        token = auth[1].decode('utf-8')
        if token == 'zhiliao':
            user = await User.objects.afirst()
            setattr(request, 'user', user)
            return user, None
        else:
            raise exceptions.AuthenticationFailed("token验证失败！")
```

那么可以在settings.py中全局配置AsyncAuthentication ，示例代码如下：

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': ['front.authentications.AsyncAuthentication'],
}
```

或者是在视图中通过authentication_classes 属性来配置，示例代码如下：

```python
class UserInfoView(APIView):
    authentication_classes = [AsyncAuthentication]

    async def get(self, request):
        return JsonResponse(data={
            "user": {
                "username": request.user.username,
                "email": request.user.email
            }
        })
```

### 3. permisssion
如果需要对API进行自定义的权限限制，那么可以实现自己权限类，在该类中实现异步的has_permission 方法即可。示例代码如下：

```python
class AsyncPermission:
    async def has_permission(self, request, view) -> bool:
        if random.random() < 0.7:
            return False

        return True

    async def has_object_permission(self, request, view, obj):
        if obj.user == request.user or request.user.is_superuser:
            return True

        return False
```

那么可以通过在settings.py 中全局配置：

```python
REST_FRAMEWORK = {
    "DEFAULT_PERMISSION_CLASSES": ['front.permissions.AsyncPermission']
}
```

或者是在视图中通过permission_clases 来配置：

```python
class UserInfoView(APIView):
    permission_classes = [AsyncPermission]

    async def get(self, request):
        return JsonResponse(data={
            "user": {
                "username": request.user.username,
                "email": request.user.email
            }
        })
```

### 4. throttle
adrf 同样也支持限速节流，通过继承自rest_framework.BaseThrottle ，然后在子类中实现异步的 allow_request方法，即可实现异步版的限速节流。示例代码如下：

```python
class AsyncThrottle(BaseThrottle):
    async def allow_request(self, request, view) -> bool:
        if random.random() < 0.7:
            return False

        return True

    def wait(self):
        return 3
```

同样可以在settings.py 中全局配置：

```python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': ['front.throttles.AsyncThrottle']
}
```

或者是在视图中通过throttle_classes 单独配置：

```python
class UserInfoView(APIView):
    throttle_classes = [AsyncThrottle]

    async def get(self, request):
        return JsonResponse(data={
            "user": {
                "username": request.user.username,
                "email": request.user.email
            }
        })
```

**注意：在DjangoRestFramework中，已经内置了许多****permission**** 、****throttle**** 的类，这些类由于不涉及到****I/O**** 操作，所以可以直接使用。而****authentication**** 模块有很多都是需要查找数据库的，那么针对这些类，就不能直接使用，而应该自定义了。**

## 四、异步视图集
异步视图集中，所有方法都必须是异步的。示例代码如下：

```python
from django.contrib.auth import get_user_model
from rest_framework.response import Response

from adrf.viewsets import ViewSet

User = get_user_model()

class AsyncViewSet(ViewSet):

    async def list(self, request):
        return Response(
            {"message": "This is the async `list` method of the viewset."}
        )

    async def retrieve(self, request, pk):
        user = await User.objects.filter(pk=pk).afirst()
        return Response({"user_pk": user and user.pk})
```

在urls.py 中，还是可以使用DefaultRouter 批量生成路由。示例代码如下：

```python
from django.urls import path, include
from rest_framework import routers

from . import views

router = routers.DefaultRouter()
router.register(r"user", views.AsyncViewSet, basename="async")

urlpatterns = [
    path("", include(router.urls)),
]
```

## 五、异步序列化
序列化类承担起两个角色，一个是校验前端传的数据是否符合格式，另外一个是将ORM对象转换为字典，方便通过JSON的格式返回给前端。接下来分别进行讲解。

### 1. 校验表单数据
我们以登录API为例，假设要校验username 和password ，那么可以编写类似以下的AsyncLoginSerializer ：

```python
from adrf.serializers import Serializer
from rest_framework import fields
import re

class AsyncLoginSerializer(Serializer):
    username = fields.CharField(max_length=200)
    password = fields.CharField(min_length=6, max_length=200)

    def validate(self, attrs):
        username = attrs.get('username')
        pattern = re.compile(r'^[a-zA-Z][a-zA-Z0-9_]{4,19}$')
        if pattern.match(username):
            return attrs
        else:
            raise fields.ValidationError("用户名不符合规则！")
```

由于adrf 没有对validate_[field] 、validate 方法编写异步处理的逻辑，**因此自定义验证相关的方法只能是同步的，从而不能在校验的逻辑中出现****I/O**** 阻塞式代码**。当然也可以定义成异步的，但是adrf的作者并没有处理异步验证的逻辑，如果要将validate_[field]和validate方法定义成异步的，那么就必须在视图函数中调用await serializer.validated_data 才能执行validate 函数中的代码，且需要通过await field才能获取到对应字段的值，所以不建议将验证方法定义成异步的。

在视图函数中调用Serializer 相关的代码，则和同步模式下是一样的，示例代码如下：

```python
from .serializers import AsyncLoginSerializer

class LoginView(APIView):
    async def post(self, request):
        serializer = AsyncLoginSerializer(data=request.data)
        if serializer.is_valid():
            validated_data = serializer.validated_data
            print('用户名：', validated_data.get('username'))
            print('密码：', validated_data.get('password'))
            return JsonResponse({'message': "登录成功！"})
        else:
            print(serializer.errors)
            return JsonResponse({"message": '登录失败！'}, status=status.HTTP_400_BAD_REQUEST)
```

另外，adrf 也提供了asave 、aupdate 、acreate 方法，可以重写该方法，然后在表单数据校验成功后，通过类似await asave() 的方式保存、更新或创建新数据。

### 2. 将ORM对象转换为字典
由于将ORM转换为字典过程中可能会涉及到SQL查询（阻塞式I/O），所以adrf 提供了异步的用于获取转换后数据的属性adata 。这里我们以序列化用户信息为例，来说明其用法。我们创建一个UserSerializer 类，代码如下：

```python
class UserSerializer(Serializer):
    username = fields.CharField(max_length=200)
    email = fields.EmailField()
    is_active = fields.BooleanField()
    is_staff = fields.BooleanField()
```

然后在视图中，可以将从QuerySet<User> 转换为字典，示例代码如下：

```python
class UserListView(APIView):
    async def get(self, request):
        queryset = User.objects.all()
        serializer = UserSerializer(queryset, many=True)
        data = await serializer.adata
        return JsonResponse({"users": data})
```

或者我们也可以继承ModelSerializer 类来创建序列化类，示例代码如下：

```python
class UserModelSerializer(ModelSerializer):
    class Meta:
        model = User
        fields = ['username', 'email', 'is_active', 'is_staff']
```



> 原文: <https://www.yuque.com/hynever/async-django/nq55zc0ipzkyv62m>