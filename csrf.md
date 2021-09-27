# Cross Site Request Forgery (CSRF)

The malicious website submitted unauthorized commands from a user that the web application trusts. The malicious
website could use, such as specially-crafted image tags, hidden forms, and JavaScript XMLHttpRequests, to submit such
unauthorized commands with correct user session id from the cookie. This kind of attack could work without the user's
interaction so it is known as 'one-click attack'.

## How to CSRF

For example, the malicious website submits unauthorized HTTP requests to the victim web application with the following 
form:

```
<iframe style="display:none" name="csrf-frame"></iframe>
<form method='POST' action='https://victim.example.com/delete' target="csrf-frame" id="csrf-form">
  <input type='hidden' name='id' value='3'>
  <input type='submit' value='submit'>
</form>
<script>document.getElementById("csrf-form").submit()</script>
```

Whenever user opens this malicious website, the above form would be submitted silently in a hidden form. Because of how 
cookie works, the victim web application would get a correct cookie and it would think this is a real request from the 
user. So, that `delete` request could be executed successfully.

## How to prevent CSRF

The difference between a malicious CSRF request and a normal request is `they have different origins`. So, we just need
to block malicious request from different origins.

1. CSRF with `GET` request is easy. The web application MUST NOT do any state change from `GET` method.
2. Use `POST` + `application/json` request and MUST CHECK the `content-type` header. A CSRF form request could mimic a 
[json payload](http://blog.opensecurityresearch.com/2012/02/json-csrf-with-parameter-padding.html) but it can't use 
`application/json` content type.
3. `Double Submit Cookie`: The site puts a random string in its cookie and inserts it in each HTML form with the same key. 
The server checks the equality between cookie value and form field to known if it's a CSRF request. Because of same-origin 
policy, the malicious site can't read and set the cookie on the target domain so it would never know the random string.
4. `SameSite` cookie attribute: if the cookie has been set with the `SameSite` attribute, the browser would only send 
the cookie with request from the same origin domain. So, this makes CSRF attach be ineffective.

## Reference

* https://blog.techbridge.cc/2017/02/25/csrf-introduction/
* https://en.wikipedia.org/wiki/Cross-site_request_forgery