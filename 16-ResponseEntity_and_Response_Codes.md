# ResponseEntity and Response Codes

### Response Contains 3 Parts:
- **Status Code**: HTTP return code like 200 OK, 500 INTERNAL_ERROR
- **Header**: Additional information 
- **Body**: Data need to be sent in response

We can use `ResponseEntity<T>` to create Response and in this "T" represents the type of the Body.

#### Example

```java
import java.net.http.HttpHeaders;

@GetMapping(path = "/get-user")
public ResponseEntity<String> getUser() {
    HttpHeaders headers = new HttpHeaders();
    headers.add("My-Header-1", "SomeValue1");
    headers.add("My-Header-2", "SomeValue2");
    
    return ResponseEntity.status(HttpStatus.OK)
            .headers(headers)
            .body("My response body object can go here");
}
```

`body` should be last, actually it's kind of using Builder design pattern, so `status`, `headers` all are returning Builder object and `body` method call returns the ResponseEntity object.

### What to do when we don't want to add any body in the response
We should use `build` method

### By default, 200 OK is the status code set
```java
@RestController
@RequestMapping(value = "/api")
public class UserController {
    @GetMapping(path = "/get-user")
    public User getUser() {
        User responseObj = new User("XYZ", 28);
        return responseObj;
    }
}
```

## @ResponseBody
- When we return plain string or POJO directly from the class, then @ResponseBody annotation is required
- Because, it tells to considered value as Response Body not as the view.

In above example, we did not use @ResponseBody, because @RestController automatically puts @ResponseBody to all the methods

## Response Codes
- [1xx (Informational)](#1x-informational)
- [2xx (Success)](#2xx-success)
- [3xx (Redirection)](#3xx-redirection)
- [4xx (Validation Error)](#4xx-validation-error)
- [5xx (Server Error)](#5xx-server-error)

### 2xx (Success)
Request received from client is received and process successfully

| Status Code | Reason | Mostly Used In | More Details |
|-------------|--------|----------------|--------------|
| 200 | OK | GET, POST (Idempotent Calls) | Request is successful and we are returning the response body |
| 201 | CREATED | POST | Request is successfully accepted but processing is not yet completed. Batch processing like Export, Import, etc. |
| 204 | NO CONFLICT | DELETE | Request is successful and we are NOT returning any data in the response body |
| 206 | PARTIAL CONTENT | POST | Request is partial successful, say suring bulk addition of 100 users, 95 requests passed and 5 failed, so this response code can be used. |

### 3xx (Redirection)
Client must take additional action to complete the request

| Status Code | Reason             | Mostly Used In                                                                                                                                                                                                                                                                                             | More Details                                                                                                                                                              |
|-------------|--------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 301 | MOVED PERMANENTLY  | When we migrate from legacy API to new API (old status code, new one is 308)                                                                                                                                                                                                                               | All request should be redirected to the new API                                                                                                                           |
| 308 | PERMANENT REDIRECT | When we migrate from legacy PAI to new API                                                                                                                                                                                                                                                                 | Same as 301, but it does not allow HTTP Method to change while redirect (for ex: if old API call is POST, then new API should also be POST, whereas it's flexible in 301) |
| 304 | NOT MODIFIED       | GET, <br/> PATH ❌ (try to avoid using it with PATCH, because: you are trying to update a name of the user, but lets say name is already same in the db, so no update required. In that case, we should not throw 304 {NOT_MODIFIED} error code, instead 204 {NO_CONFLICT} or 200 {OK} is more appropriate. | 1. Client makes a GET call, sever returned it with Last-Modified time in header. <br/> 2. Client cache the response. <br/> 3. Client makes a GET call, pass this last modified time in 'If-Modified-Since' header. <br/> 4. Server checks the particular resource last update time with what client provided, if resource is not updated, server simply returns 304 {NOT)MODIFIED}. <br/> 4. If modified, server process the request as usual and returns the new values. |

### 4xx (Validation Error)
Client need to pass correct request to server

| Status Code | Reason                | Mostly Used In                                                                                                                                                                                                                                                                           | More Details                                                                                                                                                              |
|------------|-----------------------|--------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 400 | BAD REQUEST           | GET, POST, PATCH, DELETE | Client is not passing the required details to process the request |
| 401 | UNAUTHORIZED          | GET, POST, PATCH, DELETE | Any API, which requred Authentication (like Bearer token, Basic authentication etc.) and client try to access it without providing authentication details |
| 403 | FORBIDDEN             | GET, POST, PATCH, DELETE | Lets say, only ADMIN can perform certain operation. <br/> But, if API gets invoked apart from ADMIN. <br/> We should throw 401 status code as clients (apart from ADMIN) do not have permission to access the resource. |
| 404 | NOT FOUND             | GET, PATCH, DELETE | The requested resource which client passes, is not found in DB by the server. For ex: GET the user detais with ID: 123, but in DB there is no such ID present. |
| 405 | METHOD NOT ALLOWED    | GET, POST, PATCH, DELETE | Ex: Hitting GET API, but with POST HTTP method. In Spring Boot, dispatcher servlet might throw this error, as control not reached to controller. |
| 409 | CONFLICT              | PATCH, POST, DELETE | _Add description_ |
| 422 | UN-PROCESSABLE ENTITY | GET, POST, PATCH, DELETE | Your application business validation: <br/> Like Frnce users should not be allowed to open an account (as country is not supported yet). |
| 429 | TOO MANY REQUESTS     | GET, POST, PATCH, DELTE | Lets say: <br/> Our rule is: 1 user can max make 10 calls in a minute. If user: 12345 makes the 11th call in a minute then this 11th call should get failed and we can throw 429 error code |

### 5xx (Server Error)
Request failed at server, even though client passed the valid request, means something is wrong at server.

| Status Code | Reason                | Mostly Used In                                                                                                                                                                                                                                                                           | More Details                                                                                                                                                                                                                                                                                                               |
|------------|-----------------------|--------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 500 | INTERNAL SERVER ERROR | GET, POST, PATCH, DELETE | Generic error code when no more specific error code is suitable                                                                                                                                                                                                                                                            |
| 501 | NOT IMPLEMENTED | GET, POST, PATCH, DELETE | API lacks the ability to fulfill the request. Or lets say, API is in development and in future it will be available                                                                                                                                                                                                        |
| 502 | BAD GATEWAY | GET, POST, PATCH, DELETE | Server acting as a proxy and while calling upstream got invalid response. <br/> Example: <br/> My application is deployed behind reverse proxy (niginx). If nginx is not able to communicate with my application (because of misconfiguration of port number or something), then its eligible to through 502 - BAD GATEWAY |

### 1x (Informational)
Interim response to communicate request progress or its status before processing the final request

| Status Code | Reason | Mostly Used In | More Details                                                                                                                                                                                                                                                                                                                                                                                                                                                            | 
|-------------|--------|----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 100 | CONTINUE | POST | Before sending the request, client check with server, if it can handle the request and ready: <br/> 1. Client add few things in the header first, like: <br/> a. content length: 1048576 <br/> b. content type: multipart/form-data <br/> c. expect: 100-continue. <br/> 2. Server, checks that in header, 'expect:100-continue' is present , means, client is just checking. So server validate everything (authentication, authorization, content type, length, etc.) |




