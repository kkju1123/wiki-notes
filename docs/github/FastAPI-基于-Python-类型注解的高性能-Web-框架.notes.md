# FastAPI：基于 Python 类型注解的高性能 Web 框架

*原文: [https://github.com/tiangolo/fastapi](https://github.com/tiangolo/fastapi) · 来源: github · 生成时间: 2026-09-17T08:54:55.663224+00:00*

## 背景

Python 传统 Web 框架如 Flask/Django 大多基于 WSGI，异步支持弱，且请求校验、序列化、文档生成需要大量手写或依赖扩展。随着微服务和机器学习模型服务兴起，需要高性能、开发效率高的 API 框架。FastAPI 于 2018 年出现，利用 Python 3.6+ 类型注解和 ASGI 生态，填补了这一空白。

## 痛点

不用 FastAPI 时，开发者要手动处理请求参数提取、数据校验、错误响应和 OpenAPI 文档，重复代码多且容易出错；同时传统同步框架在高并发或 WebSocket 场景下性能受限。

## 解决办法

FastAPI 将类型注解作为单一事实来源：Pydantic 模型负责请求体和响应体的校验与序列化，并生成 JSON Schema；Starlette/ASGI 提供异步高性能和 WebSocket 支持；依赖注入系统按函数参数声明自动解析路径、查询、头部和依赖。路由、模型和依赖共同生成 OpenAPI 3.1 文档，自动提供 Swagger UI 和 ReDoc。这就像给 API 写了一张带类型的合同，框架在运行时自动执行合同，并免费生成说明书。

## 关键代码示例

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

@app.post("/items/")
async def create_item(item: Item) -> dict:
    return {"item": item.model_dump()}
```

这段代码展示了 FastAPI 的核心用法：Item 继承 BaseModel，类型注解驱动请求体校验和序列化；路由函数声明 item: Item 后，FastAPI 自动从请求体解析并校验 JSON，字段缺失或类型错误会返回 422；返回值 dict 与响应模型配合可进一步过滤。运行后访问 /docs 即可看到自动生成的 Swagger UI。

## 关键流程

1. 安装 FastAPI 和 ASGI 服务器（如 uvicorn）
2. 用 Pydantic BaseModel 定义数据模型
3. 创建 FastAPI 实例并通过装饰器声明路由函数
4. 在函数签名中使用类型注解和 Depends 声明参数与依赖
5. 通过 uvicorn 启动应用，访问 /docs 查看自动文档

## 关键点

- FastAPI 以 Python 类型注解为核心，将静态类型提示转化为运行时数据校验和文档生成，显著减少样板代码和人为错误。
- 基于 Starlette ASGI 框架和 Uvicorn 服务器，支持 async/await、WebSocket 和 HTTP/2，性能接近 NodeJS 与 Go。
- Pydantic 负责数据模型的校验、序列化和 JSON Schema 生成，是自动 OpenAPI 文档的基础。
- 依赖注入系统通过 Depends 和其他参数声明自动解析请求数据，支持依赖缓存和递归依赖，提高代码复用性。
- 完全兼容 OpenAPI 和 JSON Schema，便于与 API 网关、客户端生成器、监控工具等生态集成。

## 对比与权衡

- 相比 Flask，FastAPI 原生支持 async/await、自动数据校验和交互式文档，性能更好；但 Flask 更轻量、生态成熟，适合简单同步应用。
- 相比 Django REST Framework，FastAPI 更轻量、启动更快、异步支持更好；但 Django 自带 ORM、Admin 和用户体系，适合复杂全栈项目。
- 相比 Node.js/Express，FastAPI 性能相当且 Python 类型注解加 Pydantic 提供更强的数据建模和校验能力；但 Node 生态在非阻塞 I/O 和前端同构方面有优势。

## 自测问题

**问: FastAPI 为什么能实现高性能？**

底层基于 Starlette 的 ASGI 应用和 Uvicorn 异步事件循环，避免同步阻塞；路由和请求处理均为异步，支持高并发连接。同时 Pydantic v2 核心用 Rust 编写，数据校验速度快。

**问: FastAPI 如何利用类型注解自动生成文档？**

FastAPI 在应用启动时收集所有路由签名和 Pydantic 模型，将类型注解转换为 JSON Schema，并按照 OpenAPI 规范组装成完整的 API 描述文档；内置 /docs 和 /redoc 页面基于该 Schema 渲染交互式文档。

**问: 依赖注入在 FastAPI 中如何工作？**

通过 Depends(get_db) 声明参数，FastAPI 在收到请求时根据依赖树递归解析每个依赖函数，将返回值注入路由函数；同一请求内的依赖默认缓存，避免重复执行，并支持生成器实现清理逻辑。

**问: FastAPI 中的 Pydantic 模型和 ORM 模型有什么区别？**

Pydantic 模型用于请求/响应数据的校验、序列化和文档，不直接操作数据库；ORM 模型映射数据库表结构。通常使用 response_model 和 model_config 的 from_attributes 来将 ORM 对象转换为 Pydantic 响应模型，避免暴露内部字段。

## 适用场景

- 构建高并发 RESTful API 微服务
- 机器学习模型推理服务的 HTTP 接口封装
- 需要自动生成交互式 API 文档的团队协作项目
- WebSocket 实时通信或异步任务 API

## 标签

`FastAPI` `Python` `Web框架` `API开发` `异步编程`
