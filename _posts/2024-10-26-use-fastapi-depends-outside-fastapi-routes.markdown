---
layout:     post
title:      "在 FastAPI Routes 以外無痛複用 Depends 的方法"
subtitle:   "簡單地在 Worker, CLI, Development Tool, Testing 重用你的 Dependency"
date:       2024-10-26 00:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - FastAPI
    - Depends
    - Dependency
    - Injectable
    - Worker
    - CLI
    - Testing
---
## _🚀 2024/12/21 重要更新 🚀_

距離這篇文章最初版過了兩個月，我想分享一下我的第一個開源套件 - [fastapi-injectable](https://github.com/JasperSui/fastapi-injectable)。

`fastapi-injectable` 會誕生的原因就是因為這篇文章遇到的痛點，他是 Production-ready 的一個套件，讓你能在 FastAPI 的 Routes 以外**無痛地**使用那些帶有 `Depends()` 參數的 Code，話不多說，直接上 Code：

在使用之前，先透過 `pip install fastapi-injectable` 來安裝！

```python
from typing import Annotated

from fastapi import Depends
from fastapi_injectable import injectable

class Database:
    def query(self) -> str:
        return "data"

def get_db() -> Database:
    return Database()

@injectable
def process_data(db: Annotated[Database, Depends(get_db)]) -> str:
    return db.query()

# Use it anywhere!
result = process_data()
print(result) # Output: 'data'
```

沒有 `fastapi-injectable` 以前，是沒辦法快速地直接複用既有的 Code 的，那如果你的整個專案都是基於 FastAPI `Depends()` 做商業邏輯，你就得因為 `Depends()` 沒辦法在 FastAPI 的 Routes 以外使用，去做出很多 Workaround。

但現在不用了！透過 `fastapi-injectable`，以前那些讓你很頭痛的場景都一一化解了！

1. CLI Tools
2. Background Workers
3. REPL Server
4. 任何 FastAPI Routes 以外的地方

如果有任何回饋或想法，歡迎直接發個 Issue 或 PR 來幫助 [fastapi-injectable](https://github.com/JasperSui/fastapi-injectable) 更好，也希望可以動動手指幫忙按個星星✨，我會很開心的！


> __以下接續原文...__
<hr>

## 版本

本文所附的程式碼在以下環境測試過沒問題：

1. Python: `3.12.4`
2. FastAPI: `0.115.0`

## 背景

在 FastAPI 的專案中，我們常常使用到 Dependency 加上 `Depends(get_your_dependant)` 來使用 Dependency Injection 解耦程式碼，但是只要不是在 FastAPI App 中使用到的 `Depends`，光靠 Python 自身是無法使用到他的，也就是說，如果你今天想要在自己的：

1. Worker
2. Command-line Tool (CLI)
3. Testing
4. Development Tool

來複用以下的 Code，是沒辦法的：

```python
from typing import Annotated
from fastapi import Depends

class Brand:
    pass

class Car:
    def __init__(self, brand: Brand) -> None:
        self.brand = brand

def get_brand() -> Brand:
    return Brand()

def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
    return Car(brand)

brand = get_brand()  # No Error since this function doesn't have any dependencies
car = get_car()  # TypeError: get_car() missing 1 required positional argument: 'brand'
```

`Annotated` 是參數的註釋，詳細的介紹可以看 [Python Docs](https://docs.python.org/3/library/typing.html#typing.Annotated)，Python 本身並不會幫他做任何處理，`Annotated` 是給使用者自己定義的，那誰是使用者呢？也就是會經手這個參數處理的人，在我們的情境中，FastAPI 就會是這個 Annotated 的 Annotations (`Depends`) 的使用者，你有幫 `brand` 標示 `Annotated` 並使用 `Depends` 來作為這個參數的 Annotation，FastAPI 讀到了就會幫你做事。

所以 `get_car()` 在純粹的 Python 中，等價於：

```python
# Oops, no one is handling the `brand` for us
def get_car(brand: Brand) -> Car:
    return Car(brand)

car = get_car()  # No wonder we will get TypeError: get_car() missing 1 required positional argument: 'brand'
```


## 問題

那這樣不就代表專案裡寫了一大堆的 `Depends()` 都沒辦法直接拿來用嗎？不是就代表全部 `get_your_dependant()` 要重新寫一遍，這樣很不方便呀！

其實也不只我遇到這個問題，FastAPI 的 Github Discussion 中也有人在討論這個問題 -- [Using Depends() in other functions, outside the endpoint path operation! #7720](https://github.com/fastapi/fastapi/discussions/7720)。

FastAPI 作者 Tiangolo 的想法是如果想在 FastAPI 以外去用到這些物件，最好的方式就是不要引入 FastAPI 的複雜度，而是顯式地實例化這些物件後傳進去，但這番言論顯然不被社群大多數人所接受。

![Tiangolo](https://i.imgur.com/ZxDPwC7.jpeg)

其實我個人能理解作者的想法，畢竟這也是 Python 之禪的其中一條 -- 「顯式優於隱式」(Explicity is better than implicity.)，但回歸到真實世界，我認為至少在這個使用情境能夠快速地複用原有的一大堆 Code，帶來的效益會比完全遵守「顯式優於隱式」這個慣例所帶來的成本還要高上許多 (Z>B)。

## 其他人的解法

大部分人提到的都是想要在 Testing, Worker, CLI, etc. 的地方能夠無痛地直接重用有用到 `Depends` 的 Code，有些人說到要這樣你就要用其他的 DI Framework 像是 [dependency-injector](https://github.com/ets-labs/python-dependency-injector), [injector](https://github.com/python-injector/injector), [pinject](https://github.com/google/pinject)，但對我來說都是隔靴搔癢，要換成他們就代表一樣要把整個專案翻過來寫一遍，沒有解決到真正的痛點 -- 「無痛複用 `Depends`」。

接著我就看到了 [@barapa](https://github.com/barapa) 提供的[解法](https://github.com/fastapi/fastapi/discussions/7720#discussioncomment-8661497)，同時也是我接下來提供的解法的發想來源，在這邊再次感謝他作出了這個非常有意義的貢獻！

先來看看他提供的原始碼：

### [@barapa](https://github.com/barapa) 的版本

```python
import inspect
from contextlib import AsyncExitStack
from functools import wraps
from typing import Any, Callable, Coroutine, TypeVar

from fastapi import Request
from fastapi.dependencies.models import Dependant
from fastapi.dependencies.utils import get_dependant, solve_dependencies
from loguru import logger

T = TypeVar("T")


class DependencyError(Exception):
    """Exception raised for errors during dependency injection."""

    pass


def injectable(
    func: Callable[..., Coroutine[Any, Any, T]],
) -> Callable[..., Coroutine[Any, Any, T]]:
    """
    A decorator to enable FastAPI-style dependency injection for any asynchronous function.
    This allows dependencies defined with FastAPI's Depends mechanism to be automatically
    resolved and injected into CLI tools or other components, not just web endpoints.
    Args:
        func: The asynchronous function to be wrapped, enabling dependency injection.
    Returns:
        The wrapped function with dependencies injected.
    Raises:
        ValueError: If the dependant.call is not a callable function.
        RuntimeError: If the wrapped function is not asynchronous.
        DependencyError: If an error occurs during dependency resolution.
    """

    @wraps(func)
    async def call_with_solved_dependencies(*args: Any, **kwargs: Any) -> T:  # type: ignore
        dependant: Dependant = get_dependant(path="command", call=func)
        if dependant.call is None or not callable(dependant.call):
            raise ValueError("The dependant.call attribute must be a callable.")

        if not inspect.iscoroutinefunction(dependant.call):
            raise RuntimeError("The decorated function must be asynchronous.")

        fake_request = Request({"type": "http", "headers": [], "query_string": ""})
        values: dict[str, Any] = {}
        errors: list[Any] = []

        async with AsyncExitStack() as stack:
            solved_result = await solve_dependencies(
                request=fake_request,
                dependant=dependant,
                async_exit_stack=stack,
            )
            values, errors = solved_result[0], solved_result[1]

            if errors:
                error_details = "\n".join([str(error) for error in errors])
                logger.info(f"Dependency resolution errors: {error_details}")

            return await dependant.call(*args, **{**values, **kwargs})

    return call_with_solved_dependencies

from cyclopts import App
from fastapi import Depends
from loguru import logger

from app.clis.lib.injectable import injectable
from app.deps.settings import Settings, provide_settings

app = App()

@app.default
@injectable
async def example(
    *,
    message: str,
    settings: Settings = Depends(provide_settings),
) -> None:
    """Example command using injectable with cyclopts"""
    logger.info(message)
    logger.warning(settings.generate_db_url())


if __name__ == "__main__":
    app()
```

看起來比較多行，簡單來說就是：

1. 實作一個 Decorator
2. 將被裝飾的 `func` 非同步函數組成 `fastapi.dependencies.utils.solve_dependencies` 看得懂的 `Dependant` Instance 來解析 Dependencies (透過 `fastapi.dependencies.utils.get_dependant` 來組裝)
3. 解析成功的話就直接呼叫原本的 `func` 非同步函數並把解析好的 Dependencies 原封不動地當成參數塞回去

如此就能夠呼叫有 `Depends` 參數的函數而不用自行顯式地宣告 Dependant 實例了。

不得不說這樣的實作非常優雅且簡潔，可以看到他充分利用了 FastAPI 原有公開 Utility Functions (`fastapi.dependencies.utils`) 來達到以上的目的，讓我們 (外部使用者) 能夠不用過多的臆測、自行實作過於深入的黑魔法、用到私有函數或變數，就能達成我們想要的目的，這樣的方式我認為也最大地減少了未來版本更新會造成 Breaking Change 的可能性。

但原先的實作有一些限制是在我的專案中想要改進使它更容易使用的，當前限制有：

1. 無法裝飾同步函數
2. 不支援 `Depends()` 常見的 `use_cache` 參數來讓使用者自行決定是否要拿到 Singleton

## 我的解法

為了讓我的專案能夠更順利地進行，我決定解決以上兩個問題，儘管讓一個 Decorator 能夠支援同步和非同步函數違反了 SRP 和 Python Convention，但回歸到本質來說，我們是要解決當前無法無痛複用 `Depends` 的問題的，而會發生這個問題的情境是在非 FastAPI Web Application 環境底下 e.g. Worker, CLI Tool, etc.，而我的專案常常用到 Worker 和 CLI Tool，所以對我來說犧牲原則帶來的效益遠大於成本，幾經考慮之後才決定採納這個解法。

> 無法裝飾同步函數

解決這個問題很直覺，就是讓 Decorator 能夠判斷傳入的函數為同步或非同步的即可，我們可以透過 `inspect.iscoroutinefunction` 來完成。

> 不支援 `Depends()` 常見的 `use_cache` 參數來讓使用者自行決定是否要拿到 Singleton

稍微複雜了一些，第一個是我們要讓 Decorator "Maker" 能夠傳入參數，這邊參數我一樣叫做 `use_cache` 來保持一致性，接著依照 `use_cache` 的值來決定傳入到 `solve_dependencies` 的 `dependency_cache` 是甚麼，充分利用 `solve_dependencies` 對於 `dependency_cache` 的處理：

### 當 `use_cache=True` 時

傳入的 `dependency_cache` 會是我預先建立好的 Global `_SOLVED_DEPENDENCIES` 字典實例，這個實例會不斷地被 in place 更新，用途是保留當下解析好 Dependencies，下次再被使用時若 `_SOLVED_DEPENDENCIES` 裡已經有被解析過的 Dependency (Cache Hit)，`solve_dependencies` 裡面就會從字典裡直接拿出 Depedency 實例來使用而不用重新實例化。

### 當 `use_cache=False` 時

傳入的 `dependency_cache` 會是 `None`，這樣可以確保每次執行 `solve_dependencies` 時，裡面要用到的 `dependency_cache` 都是它自己重新實例化的一個字典，藉此強制解析動作一定會發生。

綜合以上解法，我的版本誕生了：

### 完整版本

```python
import asyncio
import inspect
import logging
from collections.abc import Callable, Coroutine
from contextlib import AsyncExitStack
from functools import wraps
from typing import Any, TypeVar, cast

from fastapi import Request
from fastapi.dependencies.models import Dependant
from fastapi.dependencies.utils import get_dependant, solve_dependencies

logger = logging.getLogger(__name__)
T = TypeVar("T")
_SOLVED_DEPENDENCIES: dict[tuple[Callable[..., Any], tuple[str]], Any] = {}


class DependencyError(Exception):
    """Exception raised for errors during dependency injection."""

def injectable(  # noqa: C901
    func: Callable[..., T] | Callable[..., Coroutine[Any, Any, T]] | None = None,
    *,
    use_cache: bool = True,
) -> Callable[..., T] | Callable[..., Coroutine[Any, Any, T]]:
    """A decorator to enable FastAPI-style dependency injection for any function (sync or async).

    This allows dependencies defined with FastAPI's Depends mechanism to be automatically
    resolved and injected into CLI tools or other components, not just web endpoints.

    Args:
        func: The function to be wrapped, enabling dependency injection.
        use_cache: Whether to use the dependency cache for the arguments a.k.a sub-dependencies.

    Returns:
        The wrapped function with dependencies injected.

    Raises:
        ValueError: If the dependant.call is not a callable function.
        DependencyError: If an error occurs during dependency resolution.
    """

    def _impl(
        func: Callable[..., T] | Callable[..., Coroutine[Any, Any, T]],
    ) -> Callable[..., T] | Callable[..., Coroutine[Any, Any, T]]:
        is_async = inspect.iscoroutinefunction(func)
        dependency_cache = _SOLVED_DEPENDENCIES if use_cache else None

        async def resolve_dependencies(dependant: Dependant) -> tuple[dict[str, Any], list[Any] | None]:
            fake_request = Request({"type": "http", "headers": [], "query_string": ""})
            async with AsyncExitStack() as stack:
                solved_result = await solve_dependencies(
                    request=fake_request,
                    dependant=dependant,
                    async_exit_stack=stack,
                    embed_body_fields=False,
                    dependency_cache=dependency_cache,
                )
                dep_kwargs = solved_result.values  # noqa: PD011
                if dependency_cache is not None:
                    dependency_cache.update(solved_result.dependency_cache)

            return dep_kwargs, solved_result.errors

        def handle_errors(errors: list[Any] | None) -> None:
            if errors:
                error_details = "\n".join(str(error) for error in errors)
                logger.info(f"Dependency resolution errors: {error_details}")

        def validate_dependant(dependant: Dependant) -> None:
            if dependant.call is None or not callable(dependant.call):
                msg = "The dependant.call attribute must be a callable."
                raise ValueError(msg)

        @wraps(func)
        async def async_call_with_solved_dependencies(*args: Any, **kwargs: Any) -> T:  # noqa: ANN401
            dependant = get_dependant(path="command", call=func)
            validate_dependant(dependant)
            deps, errors = await resolve_dependencies(dependant)
            handle_errors(errors)

            return await cast(Callable[..., Coroutine[Any, Any, T]], dependant.call)(*args, **{**deps, **kwargs})

        @wraps(func)
        def sync_call_with_solved_dependencies(*args: Any, **kwargs: Any) -> T:  # noqa: ANN401
            dependant = get_dependant(path="command", call=func)
            validate_dependant(dependant)
            deps, errors = asyncio.run(resolve_dependencies(dependant))
            handle_errors(errors)

            return cast(Callable[..., T], dependant.call)(*args, **{**deps, **kwargs})

        return async_call_with_solved_dependencies if is_async else sync_call_with_solved_dependencies

    if func is None:
        return _impl  # type: ignore  # noqa: PGH003
    return _impl(func)
```

### 使用案例

還是一樣，我們用先前的 `Car` 和 `Brand` 來當作範例：

```python
from typing import Annotated
from fastapi import Depends

class Brand:
    pass

class Car:
    def __init__(self, brand: Brand) -> None:
        self.brand = brand

def get_brand() -> Brand:
    return Brand()

def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
    return Car(brand)

brand = get_brand()  # No Error since this function doesn't have any dependencies
car = get_car()  # TypeError: get_car() missing 1 required positional argument: 'brand'
```

我們現在有兩種方式可以來達到我們的目的，你可以選擇用 `@injectable` 來裝飾有 `Depends` 參數的函數，或是用 `injectable(func)` 直接包一層。

#### 1. 用 `@injectable` 來裝飾

##### 當 `use_cache=True` (預設行為)

```python
@injectable  # Equals to @injectable(use_cache=True)
def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
    return Car(brand)

car_1 = get_car()  # <__main__.Car at 0x10620cf20>
car_2 = get_car()  # <__main__.Car at 0x10953c140>
car_3 = get_car()  # <__main__.Car at 0x1095b7e00>
assert car_1.brand is car_2.brand is car_3.brand  # True
```

注意：每個 `Car` 實例的 `Brand` 實例的記憶體位址都是一樣的。

##### 當 `use_cache=False`

```python
@injectable(use_cache=False)
def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
    return Car(brand)

car_1 = get_car()  # <__main__.Car at 0x10620cf20>
car_2 = get_car()  # <__main__.Car at 0x10953c140>
car_3 = get_car()  # <__main__.Car at 0x1095b7e00>
assert car_1.brand is not car_2.brand is not car_3.brand  # True
```

注意：每個 `Car` 實例的 `Brand` 都是不同的實例。


#### 2. 用 `injectable(func)` 來包裝

##### 當 `use_cache=True` (預設行為)

```python
def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
    return Car(brand)

injectable_get_car = injectable(get_car)  # Equals to injectable(get_car, use_cache=True)
car_1 = injectable_get_car()  # <__main__.Car at 0x10620cf20>
car_2 = injectable_get_car()  # <__main__.Car at 0x10953c140>
car_3 = injectable_get_car()  # <__main__.Car at 0x1095b7e00>
assert car_1.brand is car_2.brand is car_3.brand  # True
```

##### 當 `use_cache=False`

```python
def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
    return Car(brand)

injectable_get_car = injectable(get_car, use_cache=False)
car_1 = injectable_get_car()  # <__main__.Car at 0x10620cf20>
car_2 = injectable_get_car()  # <__main__.Car at 0x10953c140>
car_3 = injectable_get_car()  # <__main__.Car at 0x1095b7e00>
assert car_1.brand is not car_2.brand is not car_3.brand  # True
```

### 同場加映完整的測試案例

```python
# type: ignore  # noqa: PGH003

from typing import Annotated
from unittest.mock import Mock, patch

import pytest
from fastapi import Depends
from fastapi.dependencies.models import Dependant

# (Paste the injectable here)
# def injectable(...):
#    ...

class Brand:
    pass


class Car:
    def __init__(self, brand: Brand) -> None:
        self.brand = brand


def test_injectable_sync_only_decorator_with_cache() -> None:
    def get_brand() -> Brand:
        return Brand()

    @injectable
    def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
        return Car(brand)

    car_1 = get_car()
    car_2 = get_car()
    car_3 = get_car()
    assert car_1.brand is car_2.brand is car_3.brand


def test_injectable_sync_only_decorator_with_no_cache() -> None:
    def get_brand() -> Brand:
        return Brand()

    @injectable(use_cache=False)
    def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
        return Car(brand)

    car_1 = get_car()
    car_2 = get_car()
    car_3 = get_car()
    assert car_1.brand is not car_2.brand is not car_3.brand


def test_injectable_sync_only_wrap_function_with_cache() -> None:
    def get_brand() -> Brand:
        return Brand()

    def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
        return Car(brand)

    injectable_get_car = injectable(get_car)
    car_1 = injectable_get_car()
    car_2 = injectable_get_car()
    car_3 = injectable_get_car()
    assert car_1.brand is car_2.brand is car_3.brand


def test_injectable_sync_only_wrap_function_with_no_cache() -> None:
    def get_brand() -> Brand:
        return Brand()

    def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
        return Car(brand)

    injectable_get_car = injectable(get_car, use_cache=False)
    car_1 = injectable_get_car()
    car_2 = injectable_get_car()
    car_3 = injectable_get_car()
    assert car_1.brand is not car_2.brand is not car_3.brand


async def test_injectable_async_only_decorator_with_cache() -> None:
    async def get_brand() -> Brand:
        return Brand()

    @injectable
    async def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
        return Car(brand)

    car_1 = await get_car()
    car_2 = await get_car()
    car_3 = await get_car()
    assert car_1.brand is car_2.brand is car_3.brand


async def test_injectable_async_only_decorator_with_no_cache() -> None:
    async def get_brand() -> Brand:
        return Brand()

    @injectable(use_cache=False)
    async def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
        return Car(brand)

    car_1 = await get_car()
    car_2 = await get_car()
    car_3 = await get_car()
    assert car_1.brand is not car_2.brand is not car_3.brand


async def test_injectable_async_only_wrap_function_with_cache() -> None:
    async def get_brand() -> Brand:
        return Brand()

    async def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
        return Car(brand)

    injectable_get_car = injectable(get_car)
    car_1 = await injectable_get_car()
    car_2 = await injectable_get_car()
    car_3 = await injectable_get_car()
    assert car_1.brand is car_2.brand is car_3.brand


async def test_injectable_async_only_wrap_function_with_no_cache() -> None:
    async def get_brand() -> Brand:
        return Brand()

    async def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
        return Car(brand)

    injectable_get_car = injectable(get_car, use_cache=False)
    car_1 = await injectable_get_car()
    car_2 = await injectable_get_car()
    car_3 = await injectable_get_car()
    assert car_1.brand is not car_2.brand is not car_3.brand


async def test_injectable_async_with_sync_decorator_with_cache() -> None:
    def get_brand() -> Brand:
        return Brand()

    @injectable
    async def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
        return Car(brand)

    car_1 = await get_car()
    car_2 = await get_car()
    car_3 = await get_car()
    assert car_1.brand is car_2.brand is car_3.brand


async def test_injectable_async_with_sync_decorator_with_no_cache() -> None:
    def get_brand() -> Brand:
        return Brand()

    @injectable(use_cache=False)
    async def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
        return Car(brand)

    car_1 = await get_car()
    car_2 = await get_car()
    car_3 = await get_car()
    assert car_1.brand is not car_2.brand is not car_3.brand


async def test_injectable_async_with_sync_wrap_function_with_cache() -> None:
    def get_brand() -> Brand:
        return Brand()

    async def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
        return Car(brand)

    injectable_get_car = injectable(get_car)
    car_1 = await injectable_get_car()
    car_2 = await injectable_get_car()
    car_3 = await injectable_get_car()
    assert car_1.brand is car_2.brand is car_3.brand


async def test_injectable_async_with_sync_wrap_function_with_no_cache() -> None:
    def get_brand() -> Brand:
        return Brand()

    async def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
        return Car(brand)

    injectable_get_car = injectable(get_car, use_cache=False)
    car_1 = await injectable_get_car()
    car_2 = await injectable_get_car()
    car_3 = await injectable_get_car()
    assert car_1.brand is not car_2.brand is not car_3.brand


def test_injectable_sync_with_async_decorator_with_cache() -> None:
    async def get_brand() -> Brand:
        return Brand()

    @injectable
    def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
        return Car(brand)

    car_1 = get_car()
    car_2 = get_car()
    car_3 = get_car()
    assert car_1.brand is car_2.brand is car_3.brand


def test_injectable_sync_with_async_decorator_with_no_cache() -> None:
    async def get_brand() -> Brand:
        return Brand()

    @injectable(use_cache=False)
    def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
        return Car(brand)

    car_1 = get_car()
    car_2 = get_car()
    car_3 = get_car()
    assert car_1.brand is not car_2.brand is not car_3.brand


def test_injectable_sync_with_async_wrap_function_with_cache() -> None:
    async def get_brand() -> Brand:
        return Brand()

    def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
        return Car(brand)

    injectable_get_car = injectable(get_car)
    car_1 = injectable_get_car()
    car_2 = injectable_get_car()
    car_3 = injectable_get_car()
    assert car_1.brand is car_2.brand is car_3.brand


def test_injectable_sync_with_async_wrap_function_with_no_cache() -> None:
    async def get_brand() -> Brand:
        return Brand()

    def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
        return Car(brand)

    injectable_get_car = injectable(get_car, use_cache=False)
    car_1 = injectable_get_car()
    car_2 = injectable_get_car()
    car_3 = injectable_get_car()
    assert car_1.brand is not car_2.brand is not car_3.brand
```

### 已知問題

到這邊已經解決了我 95% 的需求，能夠使用到這個 `injectable` 的地方數不勝數，大大的改善了開發的體驗，但是可以看到 `injectable` 中的實作有不少的 `type: ignore` 或 `noqa`，就隱隱約約能猜得到對於型別解析這一塊，目前的實作可能會有些問題，所以如果你有用一些靜態型別檢查器 e.g. `mypy`, `pyright` etc. 或是一些 Python Language Server e.g. `Pylance` ，它們一定會會抱怨這個函數或是用到這個函數的地方有型別上的問題。

原因其實就包含我前面所提到的，這個 `injectable` 的職責太多，且不符合 Python 慣例，加上靜態型別檢查器對於同時有 `Callable[..., T]` 及 `Callable[..., Awaitable[T]]` 的函數註釋支援地還沒有那麼齊全，要修正它是很困難的，但至少改版後的 `injectable` 已經足以滿足我的使用場景，對我來說這樣就夠了。

## 總結

先前都是使用 Django 和 Django Rest Framework 比較多，對於 FastAPI 這麼深入地研究還是比較少的，在深入使用 FastAPI 並瞭解它的設計哲學之後也得到很多不同的啟發。

很高興有這次機會能夠真正去解決到一些在真實專案中會需要解決的問題，也希望這個小工具能夠幫助到你！

## 參考

[FastAPI Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/)<br>
[Using Depends() in other functions, outside the endpoint path operation! #7720](https://github.com/fastapi/fastapi/discussions/7720)<br>
[PEP 492 – Coroutines with async and await syntax](https://peps.python.org/pep-0492/)