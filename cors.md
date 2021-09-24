# CORS (Cross-Origin Resource Sharing)

## What is cross-origin

* When you want to access the resource in `origin-B` from `origin-A`; this is a cross-origin request.
* Same-origin policy
  * origin: scheme + host + port
  * `https://example.com` vs `https://example.com/api`: same origin
  * `https://example.com` vs `https://api.example.com`: different origin; because of different host
  * `https://data.example.com` vs `https://api.example.com`: different origin; because of different host
  * `https://example.com` vs `http://example.com`: different origin; because of different scheme
  * `https://example.com` vs `https://example.com:8080`: different origin; because of different port

## Why does browser block different origin request?

Of course, it is for security and here are some scenarios:

* intra-net: if no CORS, a fishing website could make requests to an internal website to access confidential information.
* localhost: a fishing website could also request to `localhost` to get secret information.

## How the cross-origin request is blocked?

* Only browser would block cross-origin requests; other tools, such as `curl`, `wget`, don't have this limitation.
  * The cross-origin request is explicitly invoked with these tools instead of implicitly made by the fishing website.
* The browser **would still make the cross-origin requests** and get the result successfully, but just blocked the result back to the caller.
  * This is the concept of blocking cross-origin requests but there is difference between `simple request` and `preflight request`.

## Simple request vs Preflight request

* `Simple request`: if the request meets the following criterias, it is a `simple request`.
  * It uses one of the following methods: `GET`, `POST` or `HEAD`
  * It only uses `CORS-safe` headers: `accept`, `accept-language`, `content-language` and `content-type`.
  * And `content-type` needs to be one of the followings: `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`.
* `Preflight request`: if the request is not a `simple request` then it is a `preflight request`. For examples:
  * It uses `PUT`, `DELETE` or `PATCH` method.
  * It has self-defined header, like `X-USER-ID`.
  * Its `content-type` is `application/json`.
* If the request is a `simple request`, browser would make the request directly and check the response header for CORS. 
If the request is a `preflight request`, browser would make a `OPTIONS` request with the following 2 headers.
  * `Access-Control-Request-Headers`: which HTTP headers the client might send when the actual request is made.
  * `Access-Control-Request-Method`: which HTTP method will be used when the actual request is made.
* Then, the backend must responses the `OPTIONS` request with CORS header properly then browser would make the actual request.
  
## Set `Access-Control-*` headers

* `Access-Control-Allow-Origin`: The backend adds this header in the response. It tells the browser to allow cross-origin
request if current origin matches this value. This value could either be `*` or a single origin which means you can't set multiple
origins with this header.
  * If we don't want to use wildcard but need to allow multiple origins, the backend need to dynamically set this header based
  on the request origin header (compared with internal whitelist).
  * If the cross-origin request uses `credentials: 'include'` to include cookie in the header, `Access-Control-Allow-Origin` 
  can't be `*` and needs to set the correct origin to pass CORS.
* `Access-Control-Allow-Credentials`: If the cross-origin request uses `credentials: 'include'` to include cookie, this header 
must be `true` to pass CORS.
* `Access-Control-Allow-Headers`: The backend needs to explicitly set the allowed headers otherwise preflight request won't pass CORS.
* `Access-Control-Allow-Methods`: If the frontend uses methods except for `GET`, `POST` or `HEAD`, the backend needs to
  explicitly set the allowed methods with this header, such as `PUT`, `PATCH` or `DELETE`.
* `Access-Control-Expose-Headers`: If using any custom response headers, it needs to be exposed with this header explicitly.
* `Access-Control-Max-Age`: Indicates the number of seconds (5 by default) the information provided by the 
`Access-Control-Allow-Methods` and `Access-Control-Allow-Headers` headers can be cached.

## A pitfall for CORS header and cache behavior

* There is a pitfall for CORS header and browser cache behavior
  * Consider a server would only return `Access-Control-Allow-Origin` if the request is a CORS request.
  * First, the browser issues a non-CORS request. So, it gets a response without CORS header, and it caches that response.
  * Then, the browser subsequently makes a CORS request, it will use that cached response from the previous non-CORS request.
  * As a result, the subsequent CORS request would be blocked.
* How to solve?
  * Use `Vary: Origin` header so the browser would still issue the later CORS request if the origin is different from the cached one.

## Reference

* https://blog.huli.tw/2021/02/19/cors-guide-1/
* https://blog.huli.tw/2021/02/19/cors-guide-3/
* https://blog.huli.tw/2021/02/19/cors-guide-4/