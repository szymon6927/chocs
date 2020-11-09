# Chocs [![Build Status](https://travis-ci.org/kodemore/chocs.svg?branch=master)](https://travis-ci.org/kodemore/chocs) [![codecov](https://codecov.io/gh/kodemore/chocs/branch/master/graph/badge.svg)](https://codecov.io/gh/kodemore/chocs) [![Maintainability](https://api.codeclimate.com/v1/badges/9e3c979283b2361a9174/maintainability)](https://codeclimate.com/github/kodemore/chocs/maintainability)
Chocs is a modern HTTP framework for building AWS HTTP API/REST API and WSGI compatible applications. 
Chocs aims to be small, expressive, and robust. 
It provides an elegant API for writing fault-proof, extensible microservices.  

## Features

 - AWS Serverless integration
 - Elegant and easy API
 - No additional bloat like built-in template engines, session handlers, etc.
 - Compatible with all WSGI servers
 - Loosely coupled components which can be used separately
 - Multipart body parsing
 - Graceful error handling
 - HTTP middleware support
 - Fast routing

## Installation
```
pip install chocs
```

# Usage

## Quick start

```python
from chocs import HttpRequest
from chocs import HttpResponse
from chocs import http
from chocs import serve

@http.get("/hello/{name}")
def hello(request: HttpRequest) -> HttpResponse:
    return HttpResponse(f"Hello {request.path_parameters.get('name')}!")

serve()
```

 > Keep in mind that the `serve()` function is using the `bjoern` package, so make sure you included it in your project 
 > dependencies before using it. You are able to use any WSGI compatible server.

## Running application with Gunicorn (or any other WSGI server)

```python
# myapp.py
from chocs import HttpRequest
from chocs import HttpResponse
from chocs import create_wsgi_handler
from chocs import http

@http.get("/hello/{name}*")
def hello(request: HttpRequest) -> HttpResponse:
    return HttpResponse(f"Hello {request.path_parameters.get('name')}!")

app = create_wsgi_handler(debug=False)
```

```bash
gunicorn -w 4 myapp:app
```

## Running application in AWS Lambda (Http api or rest api)

`handler.py`
```python
import logging

from chocs import HttpRequest
from chocs import HttpResponse
from chocs import http

logger = logging.getLogger()
logger.setLevel(logging.INFO)


@http.get("/hello/{name}")
def hello_handler(request: HttpRequest) -> HttpResponse:
    logger.info("Hello AWS!")
    logger.info(request.attributes.get("aws_context"))
    logger.info(request.attributes.get("aws_event"))
    return HttpResponse(f"Hello {request.path_parameters.get('name')}")


__all__ = ["hello_handler"]
```

`serverless.yml`
```yaml
service: aws-hello-name

provider:
  name: aws
  runtime: python3.8

plugins:
  - serverless-python-requirements

custom:
  pythonRequirements:
    dockerizePip: true

functions:
  hello_name:
    handler: handler.hello_handler
    events:
      - httpApi:
          method: GET
          path: /hello/{name}
```

```bash
serverless deploy
```

## Routing
Chocs is shipped with a built-in routing module. The easiest way to utilise chocs' routing is to use `chocs.router` object.
`chocs.router` is an instance of the module's internal class `chocs.application.HttpApplication`, which provides a simple API 
where each function is a decorator corresponding to an HTTP method.

```python
from chocs import http, HttpResponse, HttpRequest

@http.get("/hello")
def hello(req: HttpRequest) -> HttpResponse:
    ...
```

The above example will assign the hello function to handle a `GET /hello` request. 

Available methods:
- `delete`
- `get`
- `head`
- `options`
- `patch`
- `post`
- `put`
- `trace`

### Parametrized routes

Routes can contain parameterised parts. Parameters must be enclosed within `{` and `}`.

```python
from chocs import http

@http.get("/pet/{id}")
def hello():
    ...
```
Will match the following URIs:
 - `/pet/1`
 - `/pet/abc`
 - `/pet/abc1`
 
### Wildcard routes

Asterisks (`*`) can be used in the route's pattern to match any possible combination. Keep in mind that routes which 
_do not_ contain wildcards are prioritised over routes with wildcards.

```python
from chocs import http

@http.get("/pet/*", id)
def hello():
    ...
```

The above example will match following URIs:
- `/pet/a`
- `/pet/a/b/c`
- `/pet/12jd/fds`

## Defining and using a custom middleware

Middleware are functions or classes that inherit `chocs.Middleware`. Middlewares have access to the request 
object and the next function is used to control middleware stack flow. Successful middleware execution should call
the `next` function *(which accepts a `chocs.HttpRequest` instance and returns `chocs.HttpReponse`)* and return a
valid `chocs.HttpResponse` instance.

Middlewares can perform various tasks:
 - Making changes in request/response objects ending
 - Validating input data
 - Authenticating users
 - End request-response cycle
 - Connecting to external data sources
 
Middlewares are different to functions decorated by `router.*` decorators as they are executed every time a request 
happens and they are not bound to the URI.
 
```python
from chocs import HttpRequest, HttpResponse, serve
from chocs.middleware import MiddlewareHandler

def my_custom_middleware(request: HttpRequest, next: MiddlewareHandler) -> HttpResponse:
    name = request.query_string.get("name", "John")
    return HttpResponse(body=f"Hello {name}")

serve(my_custom_middleware)
```

## Request
`chocs.Request` object is an abstraction around WSGI's environment and `wsgi.input` data with handy interface 
to ease everyday work.

#### `chocs.Request.headers:chocs.HttpHeaders (read-only)`
Keeps parsed headers in dict-like object.

#### `chocs.Request.body:io.BytesIO` 
Raw body data

#### `Request.parsed_body:chocs.HttpMessage`
Depending on the content type it could be one of the following:
 - `chocs.FormHttpMessage`
 - `chocs.JsonHttpMessage`
 - `chocs.MultipartHttpMessage`
 
#### `chocs.Request.cookies:typing.List[chocs.HttpCookie]` 
Request's cookies

#### `chocs.Request.method:chocs.HttpMethod`
The request's method

#### `chocs.Request.path:str`
The request's path

#### `chocs.Request.query_string:chocs.HttpQueryString`
A dict like object with parsed query string with JSON forms support
        
#### `chocs.Request.path_parameters:dict`
Matched route parameters, for example when `/users/john` matches the `/users/{name}` route, parameters will contain a 
`name` key with a value of `john`

#### `chocs.Request.attributes:dict`
Other environmental or custom attributes attached to the request object, eg.: `aws_event` or `aws_context`
when running chocs app as aws lambda.

## Response
`chocs.Response` object is a part of request-response flow and it is required to be returned by all functions
decorated with `router.*` method. Instance of the response class is recognised by `chocs.Application` and used to 
generate real response served to your clients.

#### `chocs.Response.body: io.BytesIO` 
Body served to server's clients.

### `chocs.Response.status_code: chocs.HttpStatus`
Valid response code, instance of `chocs.HttpStatus` enum can be used or just a status code's number.

#### `chocs.Response.cookies:chocs.HttpCookieJar` 
Response's cookies

#### `chocs.Response.write(body: Union[bytes, str, bytearray])`
Write bytes to response body

#### `chocs.Response.close()`
Makes body non-writable.

#### `chocs.Response.writable: bool`
Indicates whether response's body is writable.

## Working with cookies

`chocs.HttpCookieJar` object takes care of cookie handling. It can be accessed in dict-like manner, when item is requested,
instance of `chocs.HttpCookie` is returned to user. 

Cookies can be set either by passing string value to the `chocs.CookieJar`'s key, or by calling `chocs.CookieJar.append` 
method which accepts instance of `chocs.Cookie`.

### Reading client cookies

Cookies can be easily accessed from `chocs.Request.cookies` object which is injected as a parameter to each function 
registered as route handler. Consider the following example:

```python
from chocs import HttpRequest
from chocs import HttpResponse
from chocs import http
from chocs import serve


@http.get("/cookies")
def read_cookies(request: HttpRequest) -> HttpResponse:

    message = "Hello"
    if "user_name" in request.cookies:
        message += f", {str(request.cookies['user_name'])}"
    message += "!"

    return HttpResponse(body=message)

serve()
```

### Setting cookies
```python
from datetime import datetime

from chocs import HttpCookie
from chocs import HttpRequest
from chocs import HttpResponse
from chocs import http
from chocs import serve


@http.get("/cookies")
def read_cookies(request: HttpRequest) -> HttpResponse:
    response = HttpResponse(body="Hi! I have baked some cookies for ya!")
    response.cookies['simple-cookie'] = "Simple cookie for simple people"
    response.cookies.append(HttpCookie("advanced-cookie", "This cookie will expire in 2021-01-01", expires=datetime(2021, 1, 1)))
    return response

serve()
```

# Contributing

## Prerequisites

- libev
- python 3.8
- docker

## Installation

`poetry install`
