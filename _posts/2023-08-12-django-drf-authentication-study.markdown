---
layout:     post
title:      "Django & Django Rest Framework Authentication 詳細解析和狀況模擬"
subtitle:   "如果你也陷入 Django & DRF 的驗證機制中，這能夠將你從泥沼中拉出來。"
date:       2023-08-12 02:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - Django
    - Django Rest Framework
    - DRF
    - Middleware
    - Authentication
    - Authentication Class
    - Authentication Backend
    - Authentication Middleware
    - Session
    - Session Middleware
    - Auth0
---
## 前言

每每在研究 Django 和 Django Rest Framework (DRF) 的 Authentication 都會要花上不少時間來看官方文檔以及網路上五花八門的實作範例。如果範例是單純的 Django 專案那還好一點，如果是加上 DRF 的話，你就會看到明明是同樣要串接的驗證方式 e.g. JWT, Session, External Authentication Server, etc. 卻在不同的地方寫著類似的程式碼。

剛好最近我也要整合 [Auth0](https://auth0.com/) 這個最近正夯的 Auth SaaS 到 Django 和 DRF，其中也是花了不少時間在搞懂「**Django 和 DRF 的驗證機制**」葫蘆裡到底賣的是什麼藥。

本篇不會著重在串接特定的 Authentication Server 的實作上，而是透過列出一些我自己在研究怎麼串接 Auth0 時心中冒出的疑惑，並搭配一些實際的 Django & DRF Source Code 來做出最實際的回答，過程也讓我再次認知到與其去 Google 各式各樣的「**範例**」之前，不如先真的知其所以然，才有能力辨認出最適合當下情況的「**輪子**」而不是深陷泥沼中。

![img](https://i.imgur.com/WUhhqFO.png)

#### 讀這篇文章之前，你至少要 ..

1. 知道什麼是 [Django Middleware](https://docs.djangoproject.com/en/4.2/topics/http/middleware/)
2. 知道基本的 [Django Authentication](https://docs.djangoproject.com/en/dev/topics/auth/)
3. 知道基本的 [DRF Authentication](https://www.django-rest-framework.org/api-guide/authentication/)
4. 看過 DRF 的 `ViewSet`, `Permission` ([docs](https://www.django-rest-framework.org/api-guide/views/))

順帶一提，這篇提到的理解和 Django 或是 DRF 的版本關係比較小，基本上不會有太大的差異 (除了一些細節的實作會依版本不同而有些微差異)，但我可以保證 Django 2, 3, 4 都適用於這篇的觀念。

## 名詞代稱

這些名詞太像了，而且又臭又長，等等會一直提到所以就用些代稱吧：

-  Django -> Django
-  Django Rest Framework -> DRF
-  Django 的 `View` -> Django View ([docs](https://docs.djangoproject.com/en/4.2/topics/http/views/))
-  Django 的 `AUTHENTICATION_BACKENDS` -> Auth Backend ([docs](https://docs.djangoproject.com/en/4.2/topics/auth/customizing/#authentication-backends))
-  DRF 的 `APIView`, `GenericAPIView`, `ViewSet` 都是繼承於 `APIView` -> DRF `APIView` 系列 ([docs](https://www.django-rest-framework.org/api-guide/views/))
-  DRF 的 `AUTHENTICATION_CLASSES` -> Auth Class ([docs](https://www.django-rest-framework.org/api-guide/authentication/))

## 知識點

1. **Django code 裡面常常看到的 [request](https://docs.djangoproject.com/en/4.2/ref/request-response/#httprequest-objects) 和 DRF code 裡面的 [request](https://www.django-rest-framework.org/api-guide/requests/#requests) 不是同一個 Class。 (極度重要)**
2. 純粹的 Django 也能夠作為 REST API Server，但規模大到一定程度後，相較於 DRF 會沒有那麼優雅。
3. Django 收到的**所有** Request 和要送出的 Response 都會經過 Middleware。
4. DRF 是基於 Django 擴展的，**大多數情況**下 Django 是沒辦法意識到 DRF 的存在的。

## 情境劇

假設你今天和我一樣要讓 Django / DRF 能夠串接外部的身分驗證伺服器，此時你的腦袋會有什麼想法和問題？

![panik](https://i.imgur.com/XkNRRdI.jpg)

#### 1. 我們常常用的 `request.user` 是哪來的？

根據冷知識的 4. 我們可以知道，Django 的 **request** 和 DRF 的 **request** 是不一樣的，差在哪呢？

```python
from django.http import HttpRequest  # Django 的 request

from rest_framework.request import Request # DRF 的 request
```

所以在問 `request.user` 是從哪來的，要先確定你指的 `request` 是 Django 的 `HttpRequest` 還是 DRF 的 `Request`。

#### 1.1 Django `HttpRequest` 的 `user` 是從哪來的？

是 Django 的 `AuthenticationMiddleware` ([source code](https://github.com/django/django/blob/stable/4.2.x/django/contrib/auth/middleware.py#L15-L25)) 塞進來的。

```python
# Ref: https://github.com/django/django/blob/stable/4.2.x/django/contrib/auth/middleware.py#L15-L25

from django.contrib import auth
from django.utils.functional import SimpleLazyObject

def get_user(request):
    if not hasattr(request, "_cached_user"):
        request._cached_user = auth.get_user(request)
    return request._cached_user

class AuthenticationMiddleware(MiddlewareMixin):
    def process_request(self, request):
        if not hasattr(request, "session"):
            raise ImproperlyConfigured(
                "The Django authentication middleware requires session "
                "middleware to be installed. Edit your MIDDLEWARE setting to "
                "insert "
                "'django.contrib.sessions.middleware.SessionMiddleware' before "
                "'django.contrib.auth.middleware.AuthenticationMiddleware'."
            )
        request.user = SimpleLazyObject(lambda: get_user(request))
```

可以看到上述 Code 的第 8 行有呼叫 `django.contrib.auth` 的 `get_user(request)`，讓我們繼續看下去：

```python
# Ref: https://github.com/django/django/blob/stable/4.2.x/django/contrib/auth/__init__.py#L182-L223

BACKEND_SESSION_KEY = "_auth_user_backend"

# (...)

def get_user(request):
    """
    Return the user model instance associated with the given request session.
    If no user is retrieved, return an instance of `AnonymousUser`.
    """
    from .models import AnonymousUser

    user = None
    try:
        user_id = _get_user_session_key(request)
        backend_path = request.session[BACKEND_SESSION_KEY]
    except KeyError:
        pass
    else:
        if backend_path in settings.AUTHENTICATION_BACKENDS:
            backend = load_backend(backend_path)
            user = backend.get_user(user_id)
            # (...)
    return user or AnonymousUser()
```

可以看到 `get_user` 這個 function 會去拿 `HttpRequest.session`，試著從裡面 (`BACKEND_SESSION_KEY`) 找出 `backend_path`，如果 `backend_path` 被找到了就會呼叫這個 Auth Backend 的 `.get_user(user_id)` function，最後回傳 `user` 或是 `AnonymousUser()`。

等等，`user_id` 是哪來的？

```python
# Ref: https://github.com/django/django/blob/stable/4.2.x/django/contrib/auth/__init__.py#L57-L60

SESSION_KEY = "_auth_user_id"

# (...)

def _get_user_session_key(request):
    # This value in the session is always serialized to a string, so we need
    # to convert it back to Python whenever we access it.
    return get_user_model()._meta.pk.to_python(request.session[SESSION_KEY])
```

原來也是從 `HttpRequest.session` 裡面的 `SESSION_KEY` 拿出來的。

那 `settings.AUTHENTICATION_BACKENDS` 又是什麼呢？好像有點[眼熟](https://docs.djangoproject.com/en/4.2/ref/settings/#authentication-backends) ..

![img](https://i.imgur.com/Vc9mxht.png)

原來就是常常在 `settings.py` 裡面看到的

```python
# project/settings.py

# (...)

AUTHENTICATION_BACKENDS = ['django.contrib.auth.backends.ModelBackend']

# (...)
```

原來預設情況下是會去用 `ModelBackend.get_user(user_id)` 來拿到我們要的 `User` 這個 Model object，之後再透過 `AuthenticationMiddleware` 去把它塞在每一個進來的 `HttpRequest` 裡。

```python
# Ref: https://github.com/django/django/blob/stable/4.2.x/django/contrib/auth/backends.py#L159-L164

class ModelBackend(BaseBackend):
    """
    Authenticates against settings.AUTH_USER_MODEL.
    """

    # (...)

    def get_user(self, user_id):
        try:
            user = UserModel._default_manager.get(pk=user_id)
        except UserModel.DoesNotExist:
            return None
        return user if self.user_can_authenticate(user) else None
```

這樣就結束了嗎？你應該會覺得 `HttpRequest.session` 怎麼會做了那麼多事，他在哪裡把 `user_id` 偷偷塞進 `HttpRequest.session["SESSION_KEY"]` 和把 `backend_path` 塞進 `HttpRequest.session["BACKEND_SESSION_KEY"]` 的呢？

簡單來說就是要做到以下步驟：
1. 在 `settings.py` 裡面的 `MIDDLEWARES` 裡面加入 `django.contrib.sessions.SessionMiddleware` ([source code](https://github.com/django/django/blob/stable/4.2.x/django/contrib/sessions/middleware.py#L12))
2. **(Optional)** 先用  `django.contrib.auth` 的 `authenticate()` function 搭配 Auth Backend 的 `authenticate()` 來驗證使用者憑證是否合法，合法的話就回傳一個 `User` Model object ([source code](https://github.com/django/django/blob/stable/4.2.x/django/contrib/auth/__init__.py#L63-L91))。
3. 使用 `django.contrib.auth` 的 `login()` 來把 `User` 的 id (`user_id`) 存到 `HttpRequest.session[SESSION_KEY]` 裡。([source code](https://github.com/django/django/blob/stable/4.2.x/django/contrib/auth/__init__.py#L94C33-L144))
4. 於是 `HttpRequest.session[SESSION_KEY]` 裡面就會有 `user_id`，我們就可以透過 `user_id` 來在之後帶著同樣 Session ID 的 `HttpRequest` 拿回 `User` Model object 了。

至於 Django 的 Session 到底做了什麼事可以再去看看 Django 的 [How to use sessions](https://docs.djangoproject.com/en/4.2/topics/http/sessions/)。

至此你已經知道 Django 的 `HttpRequest.user` 到底是從哪來的了，在進入到 DRF 的 `Request.user` 從哪來的階段前，我們先看看 DRF 是怎麼把 Django 的 `HttpRequest` 變成他想要的 `Request` 吧！

<br>
<hr>

#### 2. DRF 是怎麼把 `HttpRequest` 變成 `Request` 的？

通常我們會在 `APIView` 系列裡面也去用到 `request.user`，當然，這邊的 `request` 就是 DRF 的 `Request`，而 `APIView` 系列就是繼承於 Django 的 `View`，在 `APIView` 裡面有非常多重要的 function，甚至你可以說 `APIView` 就是 DRF 的命根子也不為過。

而 `APIView` 裡面有一個 function 叫 `.initialize_request()` 扮演著非常重要的角色，就是他將 Django 的 `HttpRequest` 換成 DRF 的 `Request` 的。

```python
# Ref: https://github.com/encode/django-rest-framework/blob/3.14.0/rest_framework/views.py#L385-L397

class APIView(View):
    # (...)

    def initialize_request(self, request: HttpRequest, *args, **kwargs) -> Request:
        """
        Returns the initial request object.
        """
        parser_context = self.get_parser_context(request)

        return Request(
            request,
            parsers=self.get_parsers(),
            authenticators=self.get_authenticators(),
            negotiator=self.get_content_negotiator(),
            parser_context=parser_context
        )
```

在上面的範例我偷偷加上了 Type Hint，所以看到這邊就可以知道雖然都是 `request`，實則是不同的東西。

你現在已經知道了 `HttpRequest` 和 `Request` 的差異，那讓我們回到最一開始的問題：

#### 2.1 DRF `Request` 的 `user` 是從哪來的？

理所當然，他是從 `Request.user` 拿來的。

```python
class Request:
    # (...)

    # Ref: https://github.com/encode/django-rest-framework/blob/3.14.0/rest_framework/request.py#L219-L228
    @property
    def user(self):
        """
        Returns the user associated with the current request, as authenticated
        by the authentication classes provided to the request.
        """
        if not hasattr(self, '_user'):
            with wrap_attributeerrors():
                self._authenticate()
        return self._user

    # Ref: https://github.com/encode/django-rest-framework/blob/3.14.0/rest_framework/request.py#L373-L390
    def _authenticate(self):
    """
    Attempt to authenticate the request using each authentication instance
    in turn.
    """
    for authenticator in self.authenticators:
        try:
            user_auth_tuple = authenticator.authenticate(self)
        except exceptions.APIException:
            self._not_authenticated()
            raise

        if user_auth_tuple is not None:
            self._authenticator = authenticator
            self.user, self.auth = user_auth_tuple
            return

    self._not_authenticated()
```

單看這些 code，你就可以知道 `Request.user` 和 `Request.auth` 都是由 `authenticator.authenticate(request)` 拿到的，聰明的你應該猜到了，`self.authenticators` 肯定是個非常重要的角色！

讓我們回到 `APIView`。

```python
# Ref: https://github.com/encode/django-rest-framework/blob/3.14.0/rest_framework/views.py#L104C21-L104C21

class APIView(View):
    # (...)
    authentication_classes = api_settings.DEFAULT_AUTHENTICATION_CLASSES
    # (...)


    def get_authenticators(self):
        """
        Instantiates and returns the list of authenticators that this view can use.
        """
        return [auth() for auth in self.authentication_classes]
```

啊，原來 `settings.py` 的另外一個常客 `DEFAULT_AUTHENTICATION_CLASSES` 會被用在這邊：

```python
# project/settings.py

REST_FRAMEWORK = {
    # (...)
    "DEFAULT_AUTHENTICATION_CLASSES": [
        'rest_framework.authentication.BasicAuthentication',
        'rest_framework.authentication.SessionAuthentication'
    ],
    # (...)
}
```

從這邊我們可以知道，`Request.user` 的拿法就是從這些 Auth Class 的 `.authenticate()` 回傳的 `(user, auth)` tuple 來的。

同時，我們也瞭解到了一件非常重要的事：

> Django `HttpRequest.user` 和 DRF `Request.user` 彼此之間的拿法是「**完全**」沒有相依關係的。

<br>
<hr>

#### 3. 如果只是要讓 DRF 能夠驗證使用者身分，是不是可以直接把 `AuthenticationMiddleware` 和 `SessionMiddleware` 拿掉？

對，因為 `AuthenticationMiddleware` 和 `SessionMiddleware` 至始至終關注的都是 Django 的 `HttpRequest.user`，跟要在 `APIView` 系列裡面拿到 `Request.user` 裡面是兩碼子事。

所以你可以這麼說，`AuthenticationMiddleware` 和 `SessionMiddleware` 對於 **DRF 能不能夠驗證使用者身分**是沒有任何依賴關係的，甚至你可以說他拖慢了 DRF 的效能，因為在 DRF 中是用不到 `HttpRequest.user` 的，卻還要浪費時間去做前處理。

*註：DRF 中並不是完全用不到 `HttpRequest.user` 的，你可以選擇用或不用，多數情況下是不需要用到的。*

<br>
<hr>

#### 4. 我想要用 Django 的 Admin Page，`AuthenticationMiddleware` 和 `SessionMiddleware` 還可以拿掉嗎？

如果你想要拿掉這兩個 Middleware 卻不做任何 Workaround，是沒辦法使用 Django 的 Admin Page 的，原因是 Django Admin Page 的驗證機制是高度綁定這兩個 Middleware 的。

細節可以再去看看 [The Django admin site](https://docs.djangoproject.com/en/dev/ref/contrib/admin/)。

<br>
<hr>

#### 5. 那只是要讓 DRF 驗證使用者身分, 是不是只需要實作 Auth Class 就好了?

對，也不對。

大多數人會覺得 Django Authentication 和 DRF Authentication 會有關係，其實是因為 DRF 很貼心的實作了一個和 Django Session & Auth 機制互動的一個 Auth Class 叫 `SessionAuthentication`，讓我們可以用 Django `HttpRequest.session` 來塞相對應的 `User` Model object 到 DRF 的 `Request.user`。

這個 `SessionAuthentication` 做了哪些事來混淆我們的視聽呢？

```python
# Ref: https://github.com/encode/django-rest-framework/blob/3.14.0/rest_framework/authentication.py#L112-L133

class SessionAuthentication(BaseAuthentication):
    """
    Use Django's session framework for authentication.
    """

    def authenticate(self, request):
        """
        Returns a `User` if the request session currently has a logged in user.
        Otherwise returns `None`.
        """

        # Get the session-based user from the underlying HttpRequest object
        user = getattr(request._request, 'user', None)

        # Unauthenticated, CSRF validation not required
        if not user or not user.is_active:
            return None

        self.enforce_csrf(request)

        # CSRF passed with authenticated user
        return (user, None)
```

前面有提到 `Request.user` 是透過 Auth Class 的 `.authenticate()` function 來拿到 `(user, auth)` tuple 的，這邊也是一樣，`SessionAuthentication` 也實作了這個 function，但他裡面的 `user` 怎麼是去拿 `request._request`？

還記得 DRF 的 `Request` 是被 `APIView.initialize_request()` 實例化的嗎？

```python
# Ref: https://github.com/encode/django-rest-framework/blob/3.14.0/rest_framework/views.py#L385-L397

class APIView(View):
    # (...)

    def initialize_request(self, request: HttpRequest, *args, **kwargs) -> Request:
        """
        Returns the initial request object.
        """
        parser_context = self.get_parser_context(request)

        return Request(
            request,
            parsers=self.get_parsers(),
            authenticators=self.get_authenticators(),
            negotiator=self.get_content_negotiator(),
            parser_context=parser_context
        )
```

```python
# Ref: https://github.com/encode/django-rest-framework/blob/c5d9144aef1144825942ddffe0a6af23102ef44a/rest_framework/request.py#L140-L160

class Request:
    """
    Wrapper allowing to enhance a standard `HttpRequest` instance.

    Kwargs:
        - request(HttpRequest). The original request instance.
        - parsers(list/tuple). The parsers to use for parsing the
          request content.
        - authenticators(list/tuple). The authenticators used to try
          authenticating the request's user.
    """

    def __init__(self, request, parsers=None, authenticators=None,
                 negotiator=None, parser_context=None):
        assert isinstance(request, HttpRequest), (
            'The `request` argument must be an instance of '
            '`django.http.HttpRequest`, not `{}.{}`.'
            .format(request.__class__.__module__, request.__class__.__name__)
        )

        self._request = request
        # (...)
```

原來 Django 的 `HttpRequest` 就是在這邊被偷塞進來的！所以 `SessionAuthentication.authenticate()` 中，`request._request` 其實就等於 `Request.HttpRequest` 嘛！也難怪他有辦法可以透過 `request._request.user` 來直接拿到被 `AuthenticationMiddleware` 塞好的 `HttpRequest.user`，進而讓我們在 DRF 的 `APIView` 系列裡可以直接操作 `request.user`。

雖然上面我們提到了：

> Django `HttpRequest.user` 和 DRF `Request.user` 彼此之間的拿法是「**完全**」沒有相依關係的。

但 DRF 實作了 `SessionAuthentication` 讓我們可以拿到 Django `HttpRequest` 的 `User` Model object，是因為這是 `SessionAuthentication` 裡面的實作主動去達成這個目的的，`HttpRequest.user` 和 `Request.user` 依然是沒有相依關係的。

在 DRF 的文檔中也提到了 [`SessionAuthentication` 用到 Django 預設的 Session Backend 來做身分驗證](https://www.django-rest-framework.org/api-guide/authentication/#sessionauthentication)。

![img](https://i.imgur.com/6MuRaHm.png)

<br>
<hr>

#### 6. 只加 Auth Backend 卻沒有任何 Auth Class 時, 是不是在 `APIView` 系列裡只拿得到 AnonymousUser?

對，因為沒有任何 Auth Class 也就意味著 `Request.authenticators` 都會是空的，在 `Request._authenticate()` 裡面也就沒辦法拿到 `User` Model object，這時會繼續執行 `Request._not_authenticated()` function，在裡面 `Request.user` 就會被設為 AnonymousUser。

```python
# Ref: https://github.com/encode/django-rest-framework/blob/3.14.0/rest_framework/request.py#L392-L408

class Request:
    # (...)

    def _not_authenticated(self):
        """
        Set authenticator, user & authtoken representing an unauthenticated request.

        Defaults are None, AnonymousUser & None.
        """
        self._authenticator = None

        if api_settings.UNAUTHENTICATED_USER:
            self.user = api_settings.UNAUTHENTICATED_USER()
        else:
            self.user = None

        if api_settings.UNAUTHENTICATED_TOKEN:
            self.auth = api_settings.UNAUTHENTICATED_TOKEN()
        else:
            self.auth = None
```

<br>
<hr>

#### 7. 如果要在 `APIView` 系列裡面用 `request.user.has_perm`, 需不需要有 Auth Backend？

不需要，因為 DRF 的 `request.user` 的 `request` 是 DRF 的 `Request` class, 要拿到 `Request.user` 不需要透過 Auth Backend, 而是必須透過 **Auth Class**。

<br>
<hr>

#### 8. 所以 `HttpRequest.user` 和 `Request.user` 都一樣是在 Django Settings 裡面設定 `AUTH_USER_MODEL` 的那個 Model Object 嗎？

對。

<br>
<hr>

#### 9. 那我要怎麼檢查 `Request.user` 是不是有某個 Django 內建的權限? 例如很常看到的 `add_MODEL`, `view_MODEL`, `change_MODEL`, `delete_MODEL` 這些 Permission 呢?


去實作 DRF 的 Permission Class 並塞到 `APIView` 系列底下, 或是塞到 Django Settings 裡面的 REST_FRAMEWORK["DEFAULT_PERMISSION_CLASSES"], 就可以檢查 `Request.user` 是否有特定的 Permission code 了。

如果要使用 `User` 有沒有權限有三種方式：

1. 很簡單的直接用 Django ORM 檢查 Permission Code 有沒有出現這個 `User` 的 `.user_permissions` 裡。

    ```python
    from django.contrib.auth.models import Permission
    from rest_framework.permissions import BasePermission

    class HaveTheSpecificPermission(BasePermission):
        APP_LABEL = "YOURAPP"
        CODENAME = "view_MODEL"

        def has_permission(self, request: Request, view) -> bool:
            return request.user.user_permissions.filter(
                content_type__app_label=self.APP_LABEL,
                codename=self.CODENAME
            ).exists()
    ```

   但這種作法在每一次檢查時都會需要進去 Database 做一次 Query，顯然地會造成一些效能上的問題。<br><br>

2. 所以你可以自己把每個使用者的所有 Permission 在被撈出來之後就先 Cache 起來，之後就只要進去 Cache 比對 Permission 存不存在即可。<br><br>

3. 或是選擇加上 Auth Backend，並且直接用 `request.user.has_perm()` 來檢查 Permission 是否存在。
   會需要加上 Auth Backend 的原因是 `request.user.has_perm()` 裡面會去遍歷所有的 Auth Backend，並依序試著呼叫各個 Auth Backend 的 `.has_perm()`。

   舉個 `ModelBackend` 的例子，裡面實作了很多 Permission 相關的 function，也做了 2. 講的 Cache。

    ```python
    # Ref: https://github.com/django/django/blob/stable/4.2.x/django/contrib/auth/backends.py#L35

    class ModelBackend(BaseBackend):
        # (...)

        def _get_user_permissions(self, user_obj):
            return user_obj.user_permissions.all()

        def _get_group_permissions(self, user_obj):
            user_groups_field = get_user_model()._meta.get_field("groups")
            user_groups_query = "group__%s" % user_groups_field.related_query_name()
            return Permission.objects.filter(**{user_groups_query: user_obj})

        def _get_permissions(self, user_obj, obj, from_name):
            """
            Return the permissions of `user_obj` from `from_name`. `from_name` can
            be either "group" or "user" to return permissions from
            `_get_group_permissions` or `_get_user_permissions` respectively.
            """
            if not user_obj.is_active or user_obj.is_anonymous or obj is not None:
                return set()

            perm_cache_name = "_%s_perm_cache" % from_name
            if not hasattr(user_obj, perm_cache_name):
                if user_obj.is_superuser:
                    perms = Permission.objects.all()
                else:
                    perms = getattr(self, "_get_%s_permissions" % from_name)(user_obj)
                perms = perms.values_list("content_type__app_label", "codename").order_by()
                setattr(
                    user_obj, perm_cache_name, {"%s.%s" % (ct, name) for ct, name in perms}
                )
            return getattr(user_obj, perm_cache_name)

        def get_user_permissions(self, user_obj, obj=None):
            """
            Return a set of permission strings the user `user_obj` has from their
            `user_permissions`.
            """
            return self._get_permissions(user_obj, obj, "user")

        def get_group_permissions(self, user_obj, obj=None):
            """
            Return a set of permission strings the user `user_obj` has from the
            groups they belong.
            """
            return self._get_permissions(user_obj, obj, "group")

        def get_all_permissions(self, user_obj, obj=None):
            if not user_obj.is_active or user_obj.is_anonymous or obj is not None:
                return set()
            if not hasattr(user_obj, "_perm_cache"):
                user_obj._perm_cache = super().get_all_permissions(user_obj)
            return user_obj._perm_cache

        def has_perm(self, user_obj, perm, obj=None):
            return user_obj.is_active and super().has_perm(user_obj, perm, obj=obj)
    ```

<br>
<hr>

#### 10. 那 `RemoteUserBackend` 的用途是什麼？

因為他是 Auth Backend，所以主要用的地方是在 Django View，實作原理是讓你可以透過 `HttpRequest.META["REMOTE_USER"]` 來拿到 Request Header 中被 WSGI Server 轉成的 `HTTP_X_AUTH_USER` Header，
並直接信任這個 `HttpRequest.META["REMOTE_USER"]` 所帶的值是可以用來識別 Django User 的值，進而直接拿到這個 `HttpRequest` 該被賦予的 `User` Model Object。

他也可以搭配 DRF 寫好的一個 Auth Class 叫 **[RemoteUserAuthentication](https://www.django-rest-framework.org/api-guide/authentication/#remoteuserauthentication)，**裡面也是透過 `RemoteUserBackend` 去拿到 User Model Object 並接著要賦予給 `Request.user`。

這個 Auth Backend 讓你可以去串接其他 Authentication Service 驗證好的使用者，前提是有人會幫你驗證好原始請求的使用者身分之後會幫你塞使用者的辨別資訊到 Header 裡，因為 `RemoteUserBackend` 是無條件信任 `HTTP_X_AUTH_USER` 的，也因此在架構上也要確認這個 Header 是不會被惡意竄改的，否則會有使用者被假冒的資安疑慮。

<br>
<hr>

## 總結

說了這麼多，應該可以發現難的並不是怎麼串接其他身分驗證伺服器，而是這些 Code 要擺哪裡才會符合 Django 和 DRF 的設計。

![img](https://i.imgur.com/THinlLI.jpg)

對我來說，只要上面那些問題都能夠被解決，基本上要在 Django 和 DRF 內建的身分驗證機制走跳就不會是什麼太大的問題了，如果我的理解和實際狀況有出入也請指正！

另外附上兩個我在研究時意外發現的小驚喜：

1. Django 寫測試時常常用到 `APIClient` 來 call 自己的 API 測試, 並配上 `APIClient().force_authenticate(user)`, 其中原理很可能跟[這邊](https://github.com/encode/django-rest-framework/blob/c5d9144aef1144825942ddffe0a6af23102ef44a/rest_framework/request.py#L176-L180)有關。<br><br>

2. 要覆寫 DRF 預設的 401 或是 403 Response 可以在 Exception Handler 裡面去抓 `exceptions.NotAuthenticated()` 和 `exceptions.PermissionDenied()` 並換成自己的 Exception！

   [因為 `APIView.permission_denied()`  裡面會丟這些 exception 出來](https://github.com/encode/django-rest-framework/blob/c5d9144aef1144825942ddffe0a6af23102ef44a/rest_framework/views.py#L169-L175), 而 `APIView.permission_denied()` 會在 [`APIView.check_permissions()` 發現沒有權限時](https://github.com/encode/django-rest-framework/blob/c5d9144aef1144825942ddffe0a6af23102ef44a/rest_framework/views.py#L169-L175)被呼叫。

## 參考

[Django Documentation](https://docs.djangoproject.com/en/dev/)<br>
[Django REST Framework](https://www.django-rest-framework.org/)<br>
[Non-user authentication in Django and Django Rest Framework (DRF)](https://already-late.medium.com/non-user-authentication-in-django-and-django-rest-framework-drf-febaa23e0c49) *P.S. 超推這篇，講的非常好！*
