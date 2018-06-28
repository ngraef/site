title: Actions
---

The actions are the callable/public methods of the service. The action calling represents a remote-procedude-call (RPC). It has request parameters & returns response, like a HTTP request.

If you have multiple instances of services, the broker will load balancing the request among instances. [Read more about balancing](balancing.html).

<div align="center">
![Action balancing diagram](assets/action-balancing.gif)
</div>

## Call services
To call a service, use the `broker.call` method. The broker looks for the service (and a node) which has the given action and call it. The function returns a `Promise`.

### Syntax
```js
const res = await broker.call(actionName, params, opts);
```
The `actionName` is a dot-separated string. The first part of it is the service name, while the second part of it represents the action name. So if you have a `posts` service with a `create` action, you can call it as `posts.create`.

The `params` is an object which is passed to the action as a part of the [Context](context.html). The service can access it via `ctx.params`. *It is optional.*

The `opts` is an object to set/override some request parameters, e.g.: `timeout`, `retryCount`. *It is optional.*

**Available calling options:**

| Name | Type | Default | Description |
| ------- | ----- | ------- | ------- |
| `timeout` | `Number` | `null` | Timeout of request in milliseconds. If the request is timed out and you don't define `fallbackResponse`, broker will throw a `RequestTimeout` error. To disable set `0`. If it's not defined, broker uses the `requestTimeout` value of broker options. [Read more](fault-tolerance.html#Timeout) |
| `retries` | `Number` | `null` | Count of retry of request. If the request is timed out, broker will try to call again. To disable set `0`. If it's not defined, broker uses the `retryPolicy.retries` value of broker options. [Read more](fault-tolerance.html#Retry) |
| `fallbackResponse` | `Any` | `null` | Returns it, if the request has failed. [Read more](fault-tolerance.html#Fallback) |
| `nodeID` | `String` | `null` | Target nodeID. If set, it will make a direct call to the given node. |
| `meta` | `Object` | `null` | Metadata of request. Access it via `ctx.meta` in actions handlers. It will be transferred & merged at nested calls as well. |
| `parentCtx` | `Context` | `null` | Parent `Context` instance.  |
| `requestID` | `String` | `null` | Request ID or correlation ID. _It appears in the metrics events._ |


### Usages
**Call without params**
```js
broker.call("user.list")
    .then(res => console.log("User list: ", res));
```

**Call with params**
```js
broker.call("user.get", { id: 3 })
    .then(res => console.log("User: ", res));
```

**Call with async/await**
```js
const res = await broker.call("user.get", { id: 3 });
console.log("User: ", res);
```

**Call with options**
```js
broker.call("user.recommendation", { limit: 5 }, {
    timeout: 500,
    retries: 3,
    fallbackResponse: defaultRecommendation
}).then(res => console.log("Result: ", res));
```

**Call with error handling**
```js
broker.call("posts.update", { id: 2, title: "Modified post title" })
    .then(res => console.log("Post updated!"))
    .catch(err => console.error("Unable to update Post!", err));    
```

**Direct call: get health info from the "node-21" node**
```js
broker.call("$node.health", {}, { nodeID: "node-21" })
    .then(res => console.log("Result: ", res));    
```

### Metadata
With `meta` you can send meta informations to services. Access it via `ctx.meta` in action handlers. Please note at nested calls the meta is merged.
```js
broker.createService({
    name: "test",
    actions: {
        first(ctx) {
            return ctx.call("test.second", null, { meta: {
                b: 5
            }});
        },
        second(ctx) {
            console.log(ctx.meta);
            // Prints: { a: "John", b: 5 }
        }
    }
});

broker.call("test.first", null, { meta: {
    a: "John"
}});
```

The `meta` is sent back to the caller service. You can use it to send extra meta information back to the caller. E.g.: send response headers back to API gateway or set resolved logged in user to metadata.

```js
broker.createService({
    name: "test",
    actions: {
        async first(ctx) {
            await ctx.call("test.second", null, { meta: {
                a: "John"
            }});

            console.log(ctx.meta);
            // Prints: { a: "John", b: 5 }
        },
        second(ctx) {
            // Modify meta
            ctx.meta.b = 5;
        }
    }
});
```

## Streaming
Moleculer supports Node.js streams as request `params` and as response. You can use it to transfer uploaded file from a gateway or encode/decode or compress/decompress streams.

### Examples

**Send a file to a service as a stream**
```js
const stream = fs.createReadStream(fileName);

broker.call("storage.save", stream, { meta: { filename: "avatar-123.jpg" }});
```

Please note, the `params` should be a stream, you cannot add any more variables to the `params`. Use the `meta` property to transfer additional data.

**Receiving a stream in a service**
```js
module.exports = {
    name: "storage",
    actions: {
        save(ctx) {
            const s = fs.createWriteStream(`/tmp/${ctx.meta.filename}`);
            ctx.params.pipe(s);
        }
    }
};
```

**Return a stream as response in a service**
```js
module.exports = {
    name: "storage",
    actions: {
        get: {
            params: {
                filename: "string"
            },
            handler(ctx) {
                return fs.createReadStream(`/tmp/${ctx.params.filename}`);
            }
        }
    }
};
```

**Process received stream on the caller side**
```js
const filename = "avatar-123.jpg";
broker.call("storage.get", { filename })
    .then(stream => {
        const s = fs.createWriteStream(`./${filename}`);
        stream.pipe(s);
        s.on("close", () => broker.logger.info("File has been received"));
    })
```

**AES encode/decode example service**
```js
const crypto = require("crypto");
const password = "moleculer";

module.exports = {
    name: "aes",
    actions: {
        encrypt(ctx) {
            const encrypt = crypto.createCipher("aes-256-ctr", password);
            return ctx.params.pipe(encrypt);
        },

        decrypt(ctx) {
            const decrypt = crypto.createDecipher("aes-256-ctr", password);
            return ctx.params.pipe(decrypt);
        }
    }
};
```

## Action visibility
The action has a `visibility` property to control the visibility & callability of service actions.

**Available values:**
- `published` or `null`: public action. It can be called locally, remotely and can be published via API Gateway
- `public`: public action, can be called locally & remotely but not published via API GW
- `protected`: can be called only locally (from local services)
- `private`: can be called only internally (via `this.actions.xy()` inside service)

**Change visibility**
```js
module.exports = {
    name: "posts",
    actions: {
        // It's published by default
        find(ctx) {},
        clean: {
            // Callable only via `this.actions.clean`
            visibility: "private",
            handler(ctx) {}
        }
    },
    methods: {
        cleanEntities() {
            // Call the action directly
            return this.actions.clean();
        }
    }
}
```

> The default values is `null` (means `published`) due to backward compatibility.

## Action hooks
You can define action hooks to wrap certain actions which comes from mixins.
There are `before`, `after` and `error` hooks. You can assign it to a specified action or all actions (`*`) in service.
The hook can be a `Function` or a `String`. In the latter case, it must be a local service method name, what you would like to call.

**Before hooks**

```js
const DbService = require("moleculer-db");

module.exports = {
    name: "posts",
    mixins: [DbService]
    hooks: {
        before: {
            // Define a global hook for all actions
            // The hook will call the `resolveLoggedUser` method.
            "*": "resolveLoggedUser",

            // Define multiple hooks
            remove: [
                function isAuthenticated(ctx) {
                    if (!ctx.user)
                        throw new Error("Forbidden");
                },
                function isOwner(ctx) {
                    if (!this.checkOwner(ctx.params.id, ctx.user.id))
                        throw new Error("Only owner can remove it.");
                }
            ]
        }
    },

    methods: {
        async resolveLoggedUser(ctx) {
            if (ctx.meta.user)
                ctx.user = await ctx.call("users.get", { id: ctx.meta.user.id });
        }
    }
}
```

**After & Error hooks**

```js
const DbService = require("moleculer-db");

module.exports = {
    name: "users",
    mixins: [DbService]
    hooks: {
        after: {
            // Define a global hook for all actions to remove sensitive data
            "*": function(ctx, res) {
                // Remove password
                delete res.password;

                // Please note, must return result (either the original or a new)
                return res;
            },
            get: [
                // Add a new virtual field to the entity
                async function (ctx, res) {
                    res.friends = await ctx.call("friends.count", { query: { follower: res._id }});

                    return res;
                },
                // Populate the `referrer` field
                async function (ctx, res) {
                    if (res.referrer)
                        res.referrer = await ctx.call("users.get", { id: res._id });

                    return res;
                }
            ]
        },
        error: {
            // Global error handler
            "*": function(ctx, err) {
                this.logger.error(`Error occurred when '${ctx.action.name}' action was called`, err);

                // Throw further the error
                throw err;
            }
        }
    }
};
```

The recommended use case is that you create mixins which fill up the service with methods and in `hooks` you just sets method names what you want to be called.

**Mixin**
```js
module.exports = {
    methods: {
        checkIsAuthenticated(ctx) {
            if (!ctx.meta.user)
                throw new Error("Unauthenticated");
        },
        checkUserRole(ctx) {
            if (ctx.action.role && ctx.meta.user.role != ctx.action.role)
                throw new Error("Forbidden");
        },
        checkOwner(ctx) {
            // Check the owner of entity
        }
    }
}
```

**Use mixin methods in hooks**
```js
const MyAuthMixin = require("./my.mixin");

module.exports = {
    name: "posts",
    mixins: [MyAuthMixin]
    hooks: {
        before: {
            "*": ["checkIsAuthenticated"],
            create: ["checkUserRole"],
            update: ["checkUserRole", "checkOwner"],
            remove: ["checkUserRole", "checkOwner"]
        }
    },

    actions: {
        find: {
            // No required role
            handler(ctx) {}
        },
        create: {
            role: "admin",
            handler(ctx) {}
        },
        update: {
            role: "user",
            handler(ctx) {}
        }
    }
};
```

## Contexts
TODO

## Context tracking
TODO

```js
const broker = new ServiceBroker({
    nodeID: "node-1",
    tracking: {
        enabled: true,
        shutdownTimeout: 5000
    }
});
```

**Disable tracking in calling option at calling**

```js
broker.call("posts.find", {}, { tracking: false });
```