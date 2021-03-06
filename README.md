# Lua binding for libpomelo2

The [libpomelo2][] bindings for  Lua 5.1/5.2/5.3 and LuaJIT.

libpomelo2 is a client library for NetEase's [pomelo][].
pomelo is a fast, scalable, distributed game server framework for Node.js.


[libpomelo2]: https://github.com/NetEase/libpomelo2
[pomelo]: https://github.com/NetEase/pomelo

## Attention

### Polling

**The Lua binding use libpomelo2 in polling mode. That is client can't receive any event unless
polling function `pomelo.poll()` are called.**

### Memory Management

This binding library use Lua GC to free pomelo client. But currently libpomelo2's `pc_client_cleanup`
does not remove client from receive further events. So if client object at Lua side get destroyed
too early, it will certainly lead program crash.

**The solution is extend lifecycle for Lua client object. So don't keep the Lua client object only in
temporary variables. And reuse client objects if possible.**

## Requirements

This package shipped with a copy of libpomelo2.
But the OpenSSL code in libpomelo2/deps was removed for smaller package size.
If you want you OpenSSL support, you needed to install OpenSSL yourself.

You can also replace the shipped libpomelo2 with your own copy, next section will
instruct how to do this.

## Install Using Luarocks

    luarocks install pomelo

or if you like to use your own version of libpomelo2, use:

    luarocks install pomelo LIBPOMELO2_DIR=path/to/libpomelo2/dir


## Install Manually

Because pomelo is usually used as mobile game server. So Lua bindings for
libpomelo2 is mainly used in mobile environment and static link is preferred.
In this case, luarocks can't help. To add this Lua binding for you game:

1. setup OpenSSL for your project if you needs OpenSSL support.
2. add libpomelo2 to you project and make it compile and link to you project.
3. just drop `lua-pomelo.c` and `lua-pomelo.h` in to you project.
4. add luaopen_pomelo to your package.preload, see follow code.
5. don't forget to call `pomelo.poll()` method each frame update.

```c
#include <lua.h>
#include <lauxlib.h>
#include "lua-pomelo.h"

static const luaL_Reg modules[] = {
    { "pomelo", luaopen_pomelo },

    { NULL, NULL }
};

void preload_lua_modules(lua_State *L)
{
    // load extensions
    const luaL_Reg* lib = modules;
    lua_getglobal(L, "package");
    lua_getfield(L, -1, "preload");
    for (; lib->func; lib++)
    {
        lua_pushcfunction(L, lib->func);
        lua_setfield(L, -2, lib->name);
    }
    lua_pop(L, 2);
}

void your_init_code_for_lua_vm(void)
{
  lua_State* L = ....; // create your lua engine

  preload_lua_modules(L);

  // run your startup script
  // ...
}
```


## Usage


```Lua
local pomelo = require('pomelo')
```

### Library APIs

**pomelo.configure([opts])**

Update options for the libpomelo2.

Follow fields for `opts` are supported:

Global options:

* `.log`: string, the log level for the libpomelo2.
* `.cafile`: string, path to CA file.
* `.capath`: string, path to CA directory.

Client options:

* `.conn_timeout`: optional, default 30 seconds
* `.enable_reconn`: optional, default true
* `.reconn_max_retry`: optional, 'ALWAYS' or a positive integer. default 'ALWAYS'
* `.reconn_delay`: optional, default to 2
* `.reconn_delay_max`: default to 30
* `.reconn_exp_backoff`: optional, default to true
* `.transport_name`: one of 'TCP', 'TLS', 'DUMMY', or an integer id of you customized transport

```Lua
pomelo.configure({
  log='WARN', -- log level, optional, one of 'DEBUG', 'INFO', 'WARN', 'ERROR', 'DISABLE', default to 'DISABLE'
  cafile = 'path/to/ca/file', -- optional
  capath = 'path/to/ca/path', -- optional

  conn_timeout = 30, -- optional, default 30 seconds
  enable_reconn = true, -- optional, default true
  reconn_max_retry = 'ALWAYS', -- optional, 'ALWAYS' or a positive integer. default 'ALWAYS'
  reconn_delay = 2, -- integer, optional, default to 2
  reconn_delay_max = 30, -- integer, optional, default to 30
  reconn_exp_backoff = true,-- boolean, optional, default to true
  transport_name = "TCP" -- 'TCP', 'TLS', 'DUMMY', or an integer id of you customized transport
})
```

Note: Client options must set before first call to `pomelo.connect()`.
Note:  If you don't provide `cafile` and `capath`, the `cafile` and `capath` are
unchanged. If you provide provide one of them, the another one is set as NULL.

**pomelo.version()**

Returns the pomelo version string. The version string is `x.y.z-release`.


**pomelo.connect(host, port[, handshake_opts])**

Connect to server.

* host: server address.
* port: server port.
* handshake_opts: optional, see libpomelo2 docs.

Returns the new Client object.

**pomelo.poll**

Polls pending events in current thread.

**pomelo.disconnect()**

Disconnect form server.

Returns true on success.
Return nil and error information on error.


**pomelo.request(route, message[, timeout], callback)**

Send a request `message` to server at specified `route` with optional `timeout`.
When the server response received by client, the `callback` will be called  with `err` and `res`.
`err` is the err information if any or `nil` on success. `res` is the response string.

Returns true on success.
Return nil and error information on error.

**pomelo.notify(route, message[, timeout])**

Send a notify `message` to server at specified `route`.
And can optional specify `timeout`.

Returns true on success.
Return nil and error information on error.

**pomelo.on(event, listener)**

Adds the `listener` function to the end of the listeners array for the specified
`event`. No checks are made to see if the listener has already been added.
Multiple calls passing the same combination of event and listener will result
in the listener being added, and called, multiple times.

The events include:

* 'connect': with not callback args.
* 'disconnect': with not callback args.
* 'kick': with not callback args.
* 'error': with a reason callback arg.
* for user defined push, the route is the event, and message is the event callback arg.

```Lua
pomelo.on('server.module.method', function(message)
  print('got message form server.', message)
end)
```

Returns a reference to the `client` so calls can be chained.

**pomelo.addListener(event, listener)**

Alias for `pomelo.on(event, listener)`.

**pomelo.once(event, listener)**

adds a **one time** `listener` function for the `event`. This listener is invoked only the next time `event` is triggered, after which it is removed.

```Lua
pomelo.once('kicked', function()
  print('kicked by server')
end)
```
Returns a reference to the `client` so calls can be chained.

**pomelo.off(event, listener)**

Removes the specified `listener` from the listener array for the specified `event`.

```Lua
local function callback()
  print('Connected to server.')
end
pomelo.on('connected', callback)
-- ...
pomelo.off('connected', callback)
```

`pomelo.off` will remove, at most, one instance of a listener from the listener array.
If any single listener has been added multiple times to the listener array for
the specified event, then `pomelo.off` must be called multiple times to remove
each instance.

Because listeners are managed using an internal array, calling this will change
the position indices of any listener registered after the listener being removed.
This will not impact the order in which listeners are called, but it will means
that any copies of the listener array as returned by the `pomelo.listeners()`
method will need to be recreated.

Returns a reference to the `client` so calls can be chained.

**pomelo.removeListener(event, listener)**

Alias for `pomelo.off(event, listener)`.

**pomelo.listeners(event)**

Returns a copy of the array of listeners for the specified `event`.

**pomelo.state()**

return client state, one of 'NOT_INITED','INITED','CONNECTING','CONNECTED','DISCONNECTING','UNKNOWN'

**pomelo.conn_quality()**

**pomelo.connQuality()**

same as pomelo.conn_quality().


## Running Unit Tests

This project uses [busted](https://github.com/Olivine-Labs/busted) for unit test.
The the test server is running by pomelo. We can build using [luarocks](http://luarocks.org/).

**Requirements**:

1. Node.js installed on you machine.
2. Lua and [luarocks](https://github.com/keplerproject/luarocks/wiki/Download#installing) installed and available in you path.
3. Install pomelo by running `npm install pomelo -g`
4. Install busted via luarocks `luarocks install busted`
5. in `deps/libpomelo2/test/game-server` run `npm install`

**Running tests**:

1. in `deps/libpomelo2/test/game-server` run `pomelo start`
2. run `luarocks make` to build
3. run `busted`
