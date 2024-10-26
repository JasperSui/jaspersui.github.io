---
layout:     post
title:      "Easily Reusing Depends Outside FastAPI Routes"
subtitle:   "Reuse Your Dependencies in Workers, CLI Tools, Development Tools, and Testing"
date:       2024-10-26 00:00:01
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
## Versions

The code in this article has been tested in the following environment:

1. Python: `3.12.4`
2. FastAPI: `0.115.0`

## Background

In FastAPI projects, we often use dependencies with `Depends(get_your_dependant)` to decouple code through dependency injection. However, outside the FastAPI app, the standard `Depends` function cannot be used directly with pure Python alone. This means if you want to reuse code like this in:

1. Worker
2. Command-line Tool (CLI)
3. Testing
4. Development Tool

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

...it won’t work.

`Annotated` is a type annotation; more on that can be found in the [Python Docs](https://docs.python.org/3/library/typing.html#typing.Annotated). By itself, `Annotated` does nothing in Python—it’s there for the user to handle. In our case, FastAPI acts as the "user" of this annotation, processing dependencies when it finds an `Annotated` field with `Depends`. Without FastAPI, the following `get_car()` function in pure Python is equivalent to:

```python
# Oops, no one is handling the `brand` for us
def get_car(brand: Brand) -> Car:
    return Car(brand)

car = get_car()  # No wonder we will get TypeError: get_car() missing 1 required positional argument: 'brand'
```

## The Issue

Does this mean all the `Depends()` in a project can’t be reused directly? Does it mean every single `get_your_dependant()` has to be rewritten? That would be inconvenient!

I’m not the only one who’s run into this issue; it’s been discussed in a FastAPI Github thread -- [Using Depends() in other functions, outside the endpoint path operation! #7720](https://github.com/fastapi/fastapi/discussions/7720).

FastAPI's creator, Tiangolo, suggests that to use these objects outside of FastAPI, you should avoid the extra complexity of FastAPI and instead explicitly instantiate objects and pass them in. However, this approach has not been widely accepted by the community.

![Tiangolo](https://i.imgur.com/ZxDPwC7.jpeg)

I understand his viewpoint; after all, "Explicit is better than implicit" is part of the Zen of Python. But in real-world applications, the benefit of reusing a lot of existing code quickly outweighs the cost of strictly following this convention (Z>B).

## Other Solutions

Most solutions suggest ways to reuse `Depends` code in Testing, Workers, CLI, etc., often recommending alternative DI frameworks like [dependency-injector](https://github.com/ets-labs/python-dependency-injector), [injector](https://github.com/python-injector/injector), and [pinject](https://github.com/google/pinject). However, these options fall short since they’d require significant rewriting, failing to meet the core need: reusing `Depends` without hassle.

Then I found [@barapa](https://github.com/barapa)’s [solution](https://github.com/fastapi/fastapi/discussions/7720#discussioncomment-8661497), which I’ll expand on below. I’m grateful for their meaningful contribution!

### [@barapa’s](https://github.com/barapa) Version

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

In short, this solution:

1. Implements a decorator
2. Uses `fastapi.dependencies.utils.get_dependant` to transform the decorated `func` (async) function into a `Dependant` instance, parsing dependencies.
3. Upon success, directly calls the original `func` and passes the parsed dependencies as arguments.

This solution cleverly leverages FastAPI’s public utility functions to accomplish the goal without needing private methods or variables, minimizing the risk of breaking changes in future updates.

However, I wanted to make a few improvements to increase usability for my project. Currently, it has some limitations:

1. Doesn’t work on synchronous functions
2. Lacks support for `use_cache` in `Depends()`, which controls Singleton use

## My Solution

To meet my project needs, I modified it to address both limitations. Although supporting both sync and async functions in a decorator goes against SRP and Python convention, in this scenario—working outside of a FastAPI web app in tools like Workers and CLI — it’s worth the tradeoff.

> Doesn’t work on synchronous functions

The fix was simple: have the decorator check if the input function is synchronous or asynchronous using `inspect.iscoroutinefunction`.

> Lacks support for `use_cache` in `Depends()`, which controls Singleton use

I added a `use_cache` parameter to the decorator maker function, following `solve_dependencies`’ handling of `dependency_cache`.

### When `use_cache=True`

A global dictionary `_SOLVED_DEPENDENCIES` holds parsed dependencies. If a dependency is already cached, `solve_dependencies` retrieves it without reinitializing.

### When `use_cache=False`

A new `dependency_cache` dictionary is created each time to ensure dependencies are always re-solved.

### Complete Code

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

### Usage Example

Let’s use the previous `Car` and `Brand` example:

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

We now have two ways to achieve our goal: you can use `@injectable` to decorate a function with `Depends` parameters, or wrap it directly with `injectable(func)`.

#### 1. Using `@injectable` as a Decorator

##### `use_cache=True` (Default)

```python
@injectable  # Equals to @injectable(use_cache=True)
def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
    return Car(brand)

car_1 = get_car()  # <__main__.Car at 0x10620cf20>
car_2 = get_car()  # <__main__.Car at 0x10953c140>
car_3 = get_car()  # <__main__.Car at 0x1095b7e00>
assert car_1.brand is car_2.brand is car_3.brand  # True
```

Note: Each `Car` instance shares the same `Brand` instance.

##### `use_cache=False`

```python
@injectable(use_cache=False)
def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
    return Car(brand)

car_1 = get_car()  # <__main__.Car at 0x10620cf20>
car_2 = get_car()  # <__main__.Car at 0x10953c140>
car_3 = get_car()  # <__main__.Car at 0x1095b7e00>
assert car_1.brand is not car_2.brand is not car_3.brand  # True
```

Each `Car` instance has a unique `Brand` instance.

#### 2. Using `injectable(func)` as a Wrapper

##### `use_cache=True` (Default)

```python
def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
    return Car(brand)

injectable_get_car = injectable(get_car)  # Equals to injectable(get_car, use_cache=True)
car_1 = injectable_get_car()  # <__main__.Car at 0x10620cf20>
car_2 = injectable_get_car()  # <__main__.Car at 0x10953c140>
car_3 = injectable_get_car()  # <__main__.Car at 0x1095b7e00>
assert car_1.brand is car_2.brand is car_3.brand  # True
```

##### `use_cache=False`

```python
def get_car(brand: Annotated[Brand, Depends(get_brand)]) -> Car:
    return Car(brand)

injectable_get_car = injectable(get_car, use_cache=False)
car_1 = injectable_get_car()  # <__main__.Car at 0x10620cf20>
car_2 = injectable_get_car()  # <__main__.Car at 0x10953c140>
car_3 = injectable_get_car()  # <__main__.Car at 0x1095b7e00>
assert car_1.brand is not car_2.brand is not car_3.brand  # True
```

### Plus: Complete Test Case

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

### Known Issues

This solution meets 95% of my needs, greatly improving my development experience. However, the `injectable` function includes several `type: ignore` and `noqa` comments, suggesting type annotation issues that might not pass with static type checkers like `mypy` or `pyright`.

The reasons include the combined handling of `Callable[..., T]` and `Callable[..., Awaitable[T]]`, which is not fully supported by type checkers. For now, though, it fully meets my project’s requirements.

## Summary

Having primarily worked with Django and DRF, I haven’t delved as deeply into FastAPI until now. Gaining insight into its design philosophy has been enlightening.

I’m glad I had the chance to tackle this real-world issue, and I hope this tool helps you too!

## References

[FastAPI Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/)<br>
[Using Depends() in other functions, outside the endpoint path operation! #7720](https://github.com/fastapi/fastapi/discussions/7720)<br>
[PEP 492 – Coroutines with async and await syntax](https://peps.python.org/pep-0492/)

