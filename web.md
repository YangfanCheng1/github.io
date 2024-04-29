# Web

* [Security - CSRF](#security)
* [Security - CORS]
* [Security - XSS]
* [Security - WAF]

### Security - CSRF
CSRF is a common attack during which a victim user is exploited by attacker tricking user into using authenticated credentials (such as session cookie) to perform unintended actions, i.e. transfer fund from user's bank to that of attacker. 

The predominant solution to protect against CSRF attack is using a CSRF token. And this CSRF token is to be included in every stateful request. It works as follows:
1. Server includes a randomly generated CSRF token in client's frontend presentation, such as `<meta>` tag.
2. Server stores such CSRF token in HTTP session that is stored in session storage, i.e. Redis
3. Client sends a request along with CSRF token in the form of HTTP parameter or header.
4. Server receives the client request and verifies CSRF token with what's in the session storage.
5. Server rejects if token mismatches (Maclious request where attacker fails to forge the correct CSRF token).

Tech stack:
* Spring Security
* Spring Session
* Html + React/VueJs

Example HTML:
```html
<meta name="_csrf" content="${_csrf.token}"/>
```

Example http request using javascript:
```javascript

let csrf_token = document.querySelector('meta[name="_csrf"]').getAttribute('content');
let headers =
{
    headers : {
        "X-CSRF-TOKEN" : csrf_token,
        "Content-Type" : "application/json"
    }
};
axios
    .post("/api/transfer", data, headers)
    .then(response => response.data)
    .then(data => // ...
    })
    .catch(error => //...);
```
We can define in our Boot application:
```    
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.csrf()...
}
```
By default, CSRF protection is enabled by Spring Security so that above is redundant. Only required part is to provide the backing session storage if we opt for storing CSRF in [HTTP Session](https://github.com/spring-projects/spring-security/blob/main/web/src/main/java/org/springframework/security/web/csrf/HttpSessionCsrfTokenRepository.java), which is recommended over [Session Cookie](https://github.com/spring-projects/spring-security/blob/main/web/src/main/java/org/springframework/security/web/csrf/CookieCsrfTokenRepository.java). Like so:
```java
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 3600)
```
And that's it!
