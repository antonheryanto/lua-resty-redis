Name
====

lua-resty-redis - Lua redis client driver for the ngx_lua based on the cosocket API

Status
======

This library is considered production ready.

Description
===========

This Lua library is a Redis client driver for the ngx_lua nginx module:

http://wiki.nginx.org/HttpLuaModule

This Lua library takes advantage of ngx_lua's cosocket API, which ensures
100% nonblocking behavior.

Note that at least [ngx_lua 0.5.14](https://github.com/chaoslawful/lua-nginx-module/tags) or [ngx_openresty 1.2.1.14](http://openresty.org/#Download) is required.

Synopsis
========

    lua_package_path "/path/to/lua-resty-redis/lib/?.lua;;";

    server {
        location /test {
            content_by_lua '
                local redis = require "resty.redis"
                local red = redis:new()

                red:set_timeout(1000) -- 1 sec

                -- or connect to a unix domain socket file listened
                -- by a redis server:
                --     local ok, err = red:connect("unix:/path/to/redis.sock")

                local ok, err = red:connect("127.0.0.1", 6379)
                if not ok then
                    ngx.say("failed to connect: ", err)
                    return
                end

                ok, err = red:set("dog", "an animal")
                if not ok then
                    ngx.say("failed to set dog: ", err)
                    return
                end

                ngx.say("set result: ", ok)

                local res, err = red:get("dog")
                if not res then
                    ngx.say("failed to get dog: ", err)
                    return
                end

                if res == ngx.null then
                    ngx.say("dog not found.")
                    return
                end

                ngx.say("dog: ", res)

                red:init_pipeline()
                red:set("cat", "Marry")
                red:set("horse", "Bob")
                red:get("cat")
                red:get("horse")
                local results, err = red:commit_pipeline()
                if not results then
                    ngx.say("failed to commit the pipelined requests: ", err)
                    return
                end

                for i, res in ipairs(results) do
                    if type(res) == "table" then
                        if not res[1] then
                            ngx.say("failed to run command ", i, ": ", res[2])
                        else
                            -- process the table value
                        end
                    else
                        -- process the scalar value
                    end
                end

                -- put it into the connection pool of size 100,
                -- with 10 seconds max idle time
                local ok, err = red:set_keepalive(10000, 100)
                if not ok then
                    ngx.say("failed to set keepalive: ", err)
                    return
                end

                -- or just close the connection right away:
                -- local ok, err = red:close()
                -- if not ok then
                --     ngx.say("failed to close: ", err)
                --     return
                -- end
            ';
        }
    }

Methods
=======

All of the Redis commands have their own methods with the same name except all in lower case.

You can find the complete list of Redis commands here:

http://redis.io/commands

You need to check out this Redis command reference to see what Redis command accepts what arguments.

The Redis command arguments can be directly fed into the corresponding method call. For example, the "GET" redis command accepts a single key argument, then you can just call the "get" method like this:

    local res, err = red:get("key")

Similarly, the "LRANGE" redis command accepts threee arguments, then you should call the "lrange" method like this:

    local res, err = red:lrange("nokey", 0, 1)

For example, "SET", "GET", "LRANGE", and "BLPOP" commands correspond to the methods "set", "get", "lrange", and "blpop".

Here are some more examples:

    -- HMGET myhash field1 field2 nofield
    local res, err = red:hmget("myhash", "field1", "field2", "nofield")

    -- HMSET myhash field1 "Hello" field2 "World"
    local res, err = red:hmset("myhash", "field1", "Hello", "field2", "World")

All these command methods returns a single result in success and `nil` otherwise. In case of errors or failures, it will also return a second value which is a string describing the error.

A Redis "status reply" results in a string typed return value with the "+" prefix stripped.

A Redis "integer reply" results in a Lua number typed return value.

A Redis "error reply" results in a `false` value *and* a string describing the error.

A non-nil Redis "bulk reply" results in a Lua string as the return value. A nil bulk reply results in a `ngx.null` return value.

A non-nil Redis "multi-bulk reply" results in a Lua table holding all the composing values (if any). If any of the composing value is a valid redis error value, then it will be a two element table `{false, err}`.

A nil multi-bulk reply returns in a `ngx.null` value.

See http://redis.io/topics/protocol for details regarding various Redis reply types.

In addition to all those redis command methods, the following methods are also provided:

new
---
`syntax: red, err = redis:new()`

Creates a redis object. In case of failures, returns `nil` and a string describing the error.

connect
-------
`syntax: ok, err = red:connect(host, port, options_table?)`

`syntax: ok, err = red:connect("unix:/path/to/unix.sock", options_table?)`

Attempts to connect to the remote host and port that the redis server is listening to or a local unix domain socket file listened by the redis server.

Before actually resolving the host name and connecting to the remote backend, this method will always look up the connection pool for matched idle connections created by previous calls of this method.

An optional Lua table can be specified as the last argument to this method to specify various connect options:

* `pool`
: Specifies a custom name for the connection pool being used. If omitted, then the connection pool name will be generated from the string template `<host>:<port>` or `<unix-socket-path>`.

set_timeout
----------
`syntax: red:set_timeout(time)`

Sets the timeout (in ms) protection for subsequent operations, including the `connect` method.

set_keepalive
------------
`syntax: ok, err = red:set_keepalive(max_idle_timeout, pool_size)`

Puts the current Redis connection immediately into the ngx_lua cosocket connection pool.

You can specify the max idle timeout (in ms) when the connection is in the pool and the maximal size of the pool every nginx worker process.

In case of success, returns `1`. In case of errors, returns `nil` with a string describing the error.

Only call this method in the place you would have called the `close` method instead. Calling this method will immediately turn the current redis object into the `closed` state. Any subsequent operations other than `connect()` on the current objet will return the `closed` error.

get_reused_times
----------------
`syntax: times, err = red:get_reused_times()`

This method returns the (successfully) reused times for the current connection. In case of error, it returns `nil` and a string describing the error.

If the current connection does not come from the built-in connection pool, then this method always returns `0`, that is, the connection has never been reused (yet). If the connection comes from the connection pool, then the return value is always non-zero. So this method can also be used to determine if the current connection comes from the pool.

close
-----
`syntax: ok, err = red:close()`

Closes the current redis connection and returns the status.

In case of success, returns `1`. In case of errors, returns `nil` with a string describing the error.

init_pipeline
-------------
`syntax: red:init_pipeline()`

Enable the redis pipelining mode. All subsequent calls to Redis command methods will automatically get cached and will send to the server in one run when the `commit_pipeline` method is called or get cancelled by calling the `cancel_pipeline` method.

This method always succeeds.

If the redis object is already in the Redis pipelining mode, then calling this method will discard existing cached Redis queries.

commit_pipeline
---------------
`syntax: results, err = red:commit_pipeline()`

Quits the pipelining mode by committing all the cached Redis queries to the remote server in a single run. All the replies for these queries will be collected automatically and are returned as if a big multi-bulk reply at the highest level.

This method returns `nil` and a Lua string describing the error upon failures.

cancel_pipeline
---------------
`syntax: red:cancel_pipeline()`

Quits the pipelining mode by discarding all existing cached Redis commands since the last call to the `init_pipeline` method.

This method always succeeds.

If the redis object is not in the Redis pipelining mode, then this method is a no-op.

hmset
-----
`syntax: red:hmset(myhash, field1, value1, field2, value2, ...)`

`syntax: red:hmset(myhash, { field1 = value1, field2 = value2, ... })`

Special wrapper for the Redis "hmset" command.

When there are only three arguments (including the "red" object
itself), then the last argument must be a Lua table holding all the field/value pairs.

array_to_hash
-------------
`syntax: hash = red:array_to_hash(array)`

Auxiliary function that converts an array-like Lua table into a hash-like table.

This method was first introduced in the `v0.11` release.

read_reply
----------
`syntax: res, err = red:read_reply()`

Reading a reply from the redis server. This method is mostly useful for the [Redis Pub/Sub API](http://redis.io/topics/pubsub/), for example,

    local cjson = require "cjson"
    local redis = require "resty.redis"

    local red = redis:new()
    local red2 = redis:new()

    red:set_timeout(1000) -- 1 sec
    red2:set_timeout(1000) -- 1 sec

    local ok, err = red:connect("127.0.0.1", 6379)
    if not ok then
        ngx.say("1: failed to connect: ", err)
        return
    end

    ok, err = red2:connect("127.0.0.1", 6379)
    if not ok then
        ngx.say("2: failed to connect: ", err)
        return
    end

    local res, err = red:subscribe("dog")
    if not res then
        ngx.say("1: failed to subscribe: ", err)
        return
    end

    ngx.say("1: subscribe: ", cjson.encode(res))

    res, err = red2:publish("dog", "Hello")
    if not res then
        ngx.say("2: failed to publish: ", err)
        return
    end

    ngx.say("2: publish: ", cjson.encode(res))

    res, err = red:read_reply()
    if not res then
        ngx.say("1: failed to read reply: ", err)
        return
    end

    ngx.say("1: receive: ", cjson.encode(res))

    red:close()
    red2:close()

Running this example gives the output like this:

    1: subscribe: ["subscribe","dog",1]
    2: publish: 1
    1: receive: ["message","dog","Hello"]

The following class methods are provieded:

add_commands
------------
`syntax: hash = redis.add_commands(cmd_name1, cmd_name2, ...)`

Adds new redis commands to the `resty.redis` class. Here is an example:

    local redis = require "resty.redis"

    redis.add_commands("foo", "bar")

    local red = redis:new()

    red:set_timeout(1000) -- 1 sec

    local ok, err = red:connect("127.0.0.1", 6379)
    if not ok then
        ngx.say("failed to connect: ", err)
        return
    end

    local res, err = red:foo("a")
    if not res then
        ngx.say("failed to foo: ", err)
    end

    res, err = red:bar()
    if not res then
        ngx.say("failed to bar: ", err)
    end

Redis Authentication
====================

Redis uses the `AUTH` command to do authentication: http://redis.io/commands/auth

There is nothing special for this command as compared to other Redis
commands like `GET` and `SET`. So one can just invoke the `auth` method on your `resty.redis` instance. Here is an example:

    local redis = require "resty.redis"
    local red = redis:new()

    red:set_timeout(1000) -- 1 sec

    local ok, err = red:connect("127.0.0.1", 6379)
    if not ok then
        ngx.say("failed to connect: ", err)
        return
    end

    local res, err = red:auth("foobared")
    if not res then
        ngx.say("failed to authenticate: ", err)
        return
    end

where we assume that the Redis server is configured with the
password `foobared` in the `redis.conf` file:

    requirepass foobared

If the password specified is wrong, then the sample above will output the
following to the HTTP client:

    failed to authenticate: ERR invalid password

Redis Transactions
==================

This library supports the [Redis transactions](http://redis.io/topics/transactions/). Here is an example:

    local cjson = require "cjson"
    local redis = require "resty.redis"
    local red = redis:new()

    red:set_timeout(1000) -- 1 sec

    local ok, err = red:connect("127.0.0.1", 6379)
    if not ok then
        ngx.say("failed to connect: ", err)
        return
    end

    local ok, err = red:multi()
    if not ok then
        ngx.say("failed to run multi: ", err)
        return
    end
    ngx.say("multi ans: ", cjson.encode(ok))

    local ans, err = red:set("a", "abc")
    if not ans then
        ngx.say("failed to run sort: ", err)
        return
    end
    ngx.say("set ans: ", cjson.encode(ans))

    local ans, err = red:lpop("a")
    if not ans then
        ngx.say("failed to run sort: ", err)
        return
    end
    ngx.say("set ans: ", cjson.encode(ans))

    ans, err = red:exec()
    ngx.say("exec ans: ", cjson.encode(ans))

    red:close()

Then the output will be

    multi ans: "OK"
    set ans: "QUEUED"
    set ans: "QUEUED"
    exec ans: ["OK",[false,"ERR Operation against a key holding the wrong kind of value"]]

Load Balancing and Failover
===========================

You can trivially implement your own Redis load balancing logic yourself in Lua. Just keep a Lua table of all available Redis backend information (like host name and port numbers) and pick one server according to some rule (like round-robin or key-based hashing) from the Lua table at every request. You can keep track of the current rule state in your own Lua module's data, see http://wiki.nginx.org/HttpLuaModule#Data_Sharing_within_an_Nginx_Worker

Similarly, you can implement automatic failover logic in Lua at great flexibility.

Debugging
=========

It is usually convenient to use the [lua-cjson](http://www.kyne.com.au/~mark/software/lua-cjson.php) library to encode the return values of the redis command methods to JSON. For example,

    local cjson = require "cjson"
    ...
    local res, err = red:mget("h1234", "h5678")
    if res then
        print("res: ", cjson.encode(res))
    end

Automatic Error Logging
=======================

By default the underlying [ngx_lua](http://wiki.nginx.org/HttpLuaModule) module
does error logging when socket errors happen. If you are already doing proper error
handling in your own Lua code, then you are recommended to disable this automatic error logging by turning off [ngx_lua](http://wiki.nginx.org/HttpLuaModule)'s [lua_socket_log_errors](http://wiki.nginx.org/HttpLuaModule#lua_socket_log_errors) directive, that is,

    lua_socket_log_errors off;

Limitations
===========

* This library cannot be used in code contexts like init_by_lua*, set_by_lua*, log_by_lua*, and
header_filter_by_lua* where the ngx_lua cosocket API is not available.
* The `resty.redis` object instance cannot be stored in a Lua variable at the Lua module level,
because it will then be shared by all the concurrent requests handled by the same nginx
 worker process (see
http://wiki.nginx.org/HttpLuaModule#Data_Sharing_within_an_Nginx_Worker ) and
result in bad race conditions when concurrent requests are trying to use the same `resty.redis` instance.
You should always initiate `resty.redis` objects in function local
variables or in the `ngx.ctx` table. These places all have their own data copies for
each request.

Installation
============

If you are using the ngx_openresty bundle (http://openresty.org ), then
you don't need to do anything because it already includes and enables
lua-resty-redis by default. And you can just use it in your Lua code,
as in

    local redis = require "resty.redis"
    ...

If you're using your own nginx + ngx_lua build, then you need to configure
the lua_package_path directive to add the path of your lua-resty-redis source
tree to ngx_lua's LUA_PATH search path, as in

    # nginx.conf
    http {
        lua_package_path "/path/to/lua-resty-redis/lib/?.lua;;";
        ...
    }

TODO
====

Community
=========

English Mailing List
--------------------

The [openresty-en](https://groups.google.com/group/openresty-en) mailing list is for English speakers.

Chinese Mailing List
--------------------

The [openresty](https://groups.google.com/group/openresty) mailing list is for Chinese speakers.

Bugs and Patches
================

Please report bugs or submit patches by

1. creating a ticket on the [GitHub Issue Tracker](http://github.com/agentzh/lua-resty-redis/issues),
1. or posting to the [OpenResty community](#community).

Author
======

Yichun "agentzh" Zhang (章亦春) <agentzh@gmail.com>, CloudFlare Inc.

Copyright and License
=====================

This module is licensed under the BSD license.

Copyright (C) 2012-2013, by Yichun Zhang (agentzh) <agentzh@gmail.com>, CloudFlare Inc.

All rights reserved.

Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

See Also
========
* the ngx_lua module: http://wiki.nginx.org/HttpLuaModule
* the redis wired protocol specification: http://redis.io/topics/protocol
* the [lua-resty-memcached](https://github.com/agentzh/lua-resty-memcached) library
* the [lua-resty-mysql](https://github.com/agentzh/lua-resty-mysql) library

