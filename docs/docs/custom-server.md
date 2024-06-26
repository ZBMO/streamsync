# Custom server

Streamsync uses Uvicorn and serves the app in the root path i.e. `/`. If you need to use another ASGI-compatible server or fine-tune Uvicorn, you can easily do so.

## Configure webserver

You can tune your server by adding a `server_setup.py` file to the root 
of your application, next to the `main.py` and `ui.json` files.

This file is executed before starting streamsync. It allows you to configure [authentication](./authentication.md),
add your own routes and middlewares on FastAPI. 

```python
# server_setup.py
import typing

import streamsync.serve

if typing.TYPE_CHECKING:
    from fastapi import FastAPI

# Returns the FastAPI application associated with the streamsync server.
asgi_app: FastAPI = streamsync.serve.app

@asgi_app.get("/probes/healthcheck")
def hello():
    return "1"
```

::: warning Use `server_setup.py` in `edit` mode
If you want to use in `edit` mode, 
you can launch `streamsync edit --enable-server-setup <app>`.
:::

## Implement custom server

You can import `streamsync.serve` and use the function `get_asgi_app`. This returns an ASGI app created by FastAPI, which you can choose how to serve.

The following code can serve as a starting point. You can save this code as `serve.py` and run it with `python serve.py`.

```py
import uvicorn
import streamsync.serve

app_path = "." # . for current working directory
mode = "run" # run or edit

asgi_app = streamsync.serve.get_asgi_app(app_path, mode)

uvicorn.run(asgi_app,
    host="0.0.0.0",
    port=5328,
    log_level="warning",
    ws_max_size=streamsync.serve.MAX_WEBSOCKET_MESSAGE_SIZE)
```

Note the inclusion of the imported `ws_max_size` setting. This is important for normal functioning of the framework when dealing with bigger files.

Fine-tuning Uvicorn allows you to set up SSL, configure proxy headers, etc, which can prove vital in complex deployments.

::: tip Use server setup hook
```python
asgi_app = streamsync.serve.get_asgi_app(app_path, mode, enable_server_setup=True)
```
:::

## Multiple apps at once

Streamsync is built using relative paths, so it can be served from any path. This allows multiple apps to be simultaneously served on different paths.

The example below uses the `get_asgi_app` function to obtain two separate Streamsync apps, which are then mounted on different paths, `/app1` and `/app2`, of a FastAPI app.

```py
import uvicorn
import streamsync.serve
from fastapi import FastAPI, Response

root_asgi_app = FastAPI(lifespan=streamsync.serve.lifespan)
sub_asgi_app_1 = streamsync.serve.get_asgi_app("../app1", "run")
sub_asgi_app_2 = streamsync.serve.get_asgi_app("../app2", "run")

root_asgi_app.mount("/app1", sub_asgi_app_1)
root_asgi_app.mount("/app2", sub_asgi_app_2)

@root_asgi_app.get("/")
async def init():
    return Response("""
    <h1>Welcome to the App Hub</h1>
    """)

uvicorn.run(root_asgi_app,
    host="0.0.0.0",
    port=5328,
    log_level="warning",
    ws_max_size=streamsync.serve.MAX_WEBSOCKET_MESSAGE_SIZE)
```
