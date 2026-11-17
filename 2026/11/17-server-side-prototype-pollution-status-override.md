# Detecting Server-Side Prototype Pollution with a Status Code Override

*Web application vulnerabilities & exploitation · 2026-11-17*

## What it is

In Node.js apps that deep-merge user JSON into objects, a `__proto__` key can add properties to `Object.prototype`, and every object in the process then inherits them. On the server, polluting a property that changes no output looks the same as pollution failing, and a careless test can break the app for every user. A safer black-box check is to pollute a property that only changes behaviour you can observe. Node's `http-errors` builds error responses from an error object's `status`/`statusCode`, so a polluted `status` changes the HTTP code of later error responses.

## Example

```http
# 1. Baseline: malformed JSON triggers a body-parser error -> 400
POST /api/profile    Content-Type: application/json
{"name": "x",}

# 2. Candidate pollution via a merged field
POST /api/profile    Content-Type: application/json
{"name": "x", "__proto__": {"status": 555}}

# 3. Repeat request 1. A 555 instead of 400 confirms pollution
```

- `{"name": "x",}`: the trailing comma makes body-parser throw, so the app goes down a predictable error path.
- `"status": 555`: use an unusual code between 400 and 599. Values outside that range fall back to 500, so you can't tell whether pollution worked.
- Step 3: the error object has no `status` of its own, so it inherits the polluted one.
- Clean up afterwards with `{"__proto__": {"status": 400}}`. Pollution lasts until the process restarts.

## When to use it / why it matters

- **Offensive (authorized testing):** this confirms a pollution sink without depending on any gadget being reflected back. Similar non-destructive checks include `"json spaces": 10`, which changes the indentation of JSON responses (patched in Express 4.17.4, so view the Raw response), and a `content-type` charset override in body-parser. Once you've confirmed a sink, you can look for gadgets such as `child_process` `shell`/`NODE_OPTIONS` that lead to RCE.
- **Defensive:** use `Object.create(null)` or `Map` for merged data, validate request bodies against a schema with additional properties rejected, and use merge libraries that skip `__proto__`, `constructor` and `prototype`. `Object.freeze(Object.prototype)` at startup blocks the whole class.
- **Gotchas:** blocking only the `__proto__` key can be bypassed with `{"constructor": {"prototype": {...}}}`. `node --disable-proto=delete` has the same gap. Pollution is shared across the whole process, so test against staging when you can.
- **Detection:** alert on request bodies containing `__proto__` or `constructor.prototype`, and watch for unusual 4xx/5xx codes such as 555 in access logs.

## Sources

- [PortSwigger Web Security Academy: Server-side prototype pollution](https://portswigger.net/web-security/prototype-pollution/server-side)
- [Gareth Heyes, "Server-side prototype pollution: Black-box detection without the DoS"](https://portswigger.net/research/server-side-prototype-pollution)
- [`http-errors` README](https://github.com/jshttp/http-errors)
