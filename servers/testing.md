---
title: Testing 
category: servers
permalink: /servers/testing.html
caption: Testing Server Applications 
priority: 1100
redirect_from:
  - /application/testing.html
---

Ktor is designed to allow the creation of applications that are easily testable. And of course,
Ktor infrastructure itself is well tested with unit, integration, and stress tests.
In this section, you will learn how to test your applications. 

**Table of contents:**

* TOC
{:toc}

## TestEngine

Ktor has a special kind engine `TestEngine`, that doesn't create a web server, doesn't bind to sockets and doesn't do
any real HTTP requests. Instead, it hooks directly into internal mechanisms and processes `ApplicationCall` directly. 
This allows for fast test execution at the expense of maybe missing some HTTP processing details. 
It's perfectly capable of testing application logic, but be sure to set up integration tests as well.

A quick walkthrough:  

* Add `ktor-server-test-host` dependency to the `test` scope 
* Create a JUnit test class and a test function
* Use `withTestApplication` function to setup a test environment for your Application
* Use the `handleRequest` function to send requests to your application and verify the results

See an [example](#example) on this page.

## Building post/put bodies

### `application/x-www-form-urlencoded`

When building the request, you have to add a `Content-Type` header:

```kotlin
addHeader(HttpHeaders.ContentType, ContentType.Application.FormUrlEncoded.toString())`
```

And then set the `bodyChannel`, for example, by calling the `setBody` method:

```kotlin
setBody("name1=value1&name2=value%202")
```

Ktor provides an extension method to build afrom url encoded `name1=value1&name2=value%202...`:
`fun List<Pair<String, String>>.formUrlEncode(): String`.

So a complete example to build a post request urlencoded could be:

```kotlin
val call = handleRequest(HttpMethod.Post, "/route") {
   addHeader(HttpHeaders.ContentType, ContentType.Application.FormUrlEncoded.toString())
   setBody(listOf("name1" to "value1", "name2" to "value2").formUrlEncode())
}
```

### `multipart/form-data`

When uploading big files, it is common to use the multipart encoding, which allows sending
complete files without preprocessing. Ktor's test host provides a `setBody` extension method
to build this kind of payload. For example:

```kotlin
val call = handleRequest(HttpMethod.Post, "/upload") {
    val boundary = "***bbb***"

    addHeader(HttpHeaders.ContentType, ContentType.MultiPart.FormData.withParameter("boundary", boundary).toString())
    setBody(boundary, listOf(
        PartData.FormItem("title123", { }, headersOf(
            HttpHeaders.ContentDisposition,
            ContentDisposition.Inline
                .withParameter(ContentDisposition.Parameters.Name, "title")
                .toString()
        )),
        PartData.FileItem({ byteArrayOf(1, 2, 3).inputStream() }, {}, headersOf(
            HttpHeaders.ContentDisposition,
            ContentDisposition.File
                .withParameter(ContentDisposition.Parameters.Name, "file")
                .withParameter(ContentDisposition.Parameters.FileName, "file.txt")
                .toString()
        ))
    ))
}
```
{: .compact}

## Defining configuration properties in tests

In tests, instead of using an `application.conf` to define configuration properties,
you can use the `MapApplicationConfig.put` method:

```kotlin
withTestApplication({
    (environment.config as MapApplicationConfig).apply {
        // Set here the properties
        put("youkube.session.cookie.key", "03e156f6058a13813816065")
        put("youkube.upload.dir", tempPath.absolutePath)
    }
    main() // Call here your application's module
})
```

## HttpsRedirect feature

The HttpsRedirect changes how testing is performed.
Check the [testing section of the HttpsRedirect feature](/servers/features/https-redirect.html#testing) for more information.

## Example

See full example of application testing in [ktor-samples-testable](https://github.com/ktorio/ktor-samples/tree/master/feature/testable).
Also, most [`ktor-samples`](https://github.com/ktorio/ktor-samples) modules provide
examples of how to test specific functionalities.

**build.gradle:**
```groovy
// ...
dependencies {
    // ...
    testCompile "io.ktor:ktor-server-test-host:$ktor_version"
}
```

**Module:**
```kotlin
fun Application.testableModule() {
    intercept(ApplicationCallPipeline.Call) { call ->
        if (call.request.uri == "/")
            call.respondText("Test String")
    }
}
```

**Test:**
```kotlin
class ApplicationTest {
    @Test fun testRequest() = withTestApplication(Application::testableModule) {
        with(handleRequest(HttpMethod.Get, "/")) {
            assertEquals(HttpStatusCode.OK, response.status())
            assertEquals("Test String", response.content)
        }
        with(handleRequest(HttpMethod.Get, "/index.html")) {
            assertFalse(requestHandled)
        }
    }
}
```
