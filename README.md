# redis-client
[![Coverage Status](https://coveralls.io/repos/github/Oozlum/redis-client/badge.svg)](https://coveralls.io/github/Oozlum/redis-client)
[![Build Status](https://travis-ci.com/Oozlum/redis-client.svg?branch=master)](https://travis-ci.com/github/Oozlum/redis-client)

A Redis client library for Lua

## Compatibility
Lua >= 5.1 and LuaJIT.

## Features
- Written in pure Lua.
- Available through luarocks.
- One external dependency.
- Synchronous by default, optionally asynchronous using continuation queues (cqueues).
- A single connection can be shared by multiple co-routines (cqueues).
- Manages transactions.
- Supports redis pub/sub mechanism.
- Easily extensible.
- Allows both low- and high-level protocol use.

## Runtime Dependencies
- [cqueues](https://github.com/wahern/cqueues) >= 20150907

## Installation
redis-client is available via luarocks:
```
$ luarocks install redis-client
```
Alternatively clone the repo and copy the library files directly, for example:
```
$ git clone https://github.com/Oozlum/redis-client
$ cp -r redis-client/redis-client /usr/local/share/lua/5.3
```

## Building and Development
redis-client is written in pure Lua, so there are no build steps required.  However, you may want to use the following tools to check and test any changes:
- [luacheck](https://github.com/mpeterv/luacheck)
- [busted](http://olivinelabs.com/busted/)
- [luacov](https://keplerproject.github.io/luacov/)

```
$ luacheck .
$ busted -c && luacov && less luacov.report.out
```

## Usage
redis-client is split into four modules which provide functionality at increasingly higher levels:
- [redis-client.protocol](#redis-clientprotocol) handles the encoding/decoding of messages between redis server and client.
- [redis-client.response](#redis-clientresponse) deals with formatting of the data returned by the redis server.
- [redis-client.redis](#redis-clientredis) manages the network communications and synchronisation between multiple users (co-routines) of a single client connection.
- [redis-client](#redis-client) performs high-level transaction handling, validation of commands and command-dependent response manipulation.

### Error Handling
There are two types of errors within this library: soft- and hard errors.  A soft error is one that is reported by the redis server itself -- there was no error within the library or the protocol, but the server didn't like what it received; soft errors are not managed by the library but are reported as valid data responses, as described below.  A hard error is something that prevented the library from completing an operation and is reported directly to the caller.

All library functions report hard errors by returning ```nil, err_type, err_msg``` by default.  (This error handling can be overridden by providing a custom error handler at the module-, client- or call level, which is described later.)

```err_type``` and ```err_msg``` are strings.  ```err_type``` is one of the following values:
- 'SOCKET' -- an error occurred at the network level.
- 'PROTOCOL' -- the library didn't understand what the server said.
- 'USAGE' -- there was a problem with how you called a function, perhaps invalid arguments.

'SOCKET' and 'PROTOCOL' errors are generally unrecoverable and require you to close and re-establish the connection to the server.

### Response Handling
All redis commands return a value that is of one of the following types:
- STATUS
- ERROR
- INT
- STRING
- ARRAY

STRING and ARRAY types may be nil, empty or appropriate content.  Arrays are integer keyed tables containing elements of the above types and may contain other arrays.  Data responses are by default formatted as a table (This can be overridden by providing a custom response renderer, described later):
```lua
{
  type = redis_client.response.STATUS, -- .ERROR, .INT, .STRING or .ARRAY
  data = data,
}
```
Some example responses:
```lua
-- PING =
{
  type = redis_client.response.STATUS,
  data = 'PONG',
}

-- GET string =
{
  type = redis_client.response.STRING,
  data = 'string value',
}

-- LRANGE list 0 -1 =
{
  type = redis_client.response.ARRAY,
  data = {
    {
      type = redis_client.response.STRING,
      data = 'list entry 1',
    },
    {
      type = redis_client.response.STRING,
      data = 'list entry 2',
    },
  },
}
```

## redis-client.protocol

This module is responsible for encoding/decoding messages on the wire.

### redis-client.protocol.send\_command
```lua
ok, err_type, err_msg = redis_client.protocol.send_command(file, args)
```
```file```: an object that behaves like a Lua ```file``` object, providing ```read```, ```write``` and ```flush``` functions.
```args```: a array of strings/integers containing the redis command and arguments.
returns ```true``` or ```nil, err_type, err_msg``` on failure.

Example:
```lua
ok, err_type, err_msg = redis_client.protocol.send_command(file, { 'get', 'string' })
```

### redis-client.protocol.read\_response
```lua
ok, err_type, err_msg = redis_client.protocol.read_response(file)
```
```file```: an object that behaves like a Lua ```file``` object, providing ```read```, ```write``` and ```flush``` functions.
returns a table containing the response data (see [Response Handling](#response-handling) or ```nil, err_type, err_msg``` on failure.

## redis-client.response

This module provides the default response formatting for higher-level modules.  The default response rendering can be changed by replacing the ```redis-client.response.new``` function, for example:

```lua
redis_client.response.new = function(cmd, options, args, type, data)
  return {
    type = type,
    data = data
  }
end
```
```cmd```, ```options``` and ```args``` are the (uppercased) command, options and arguments that were given as part of the redis command.  ```type``` and ```data``` are as described above in [Response Handling](#response-handling).
