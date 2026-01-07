---
title: "Deep Dive into ObjectMapper: From Internal Mechanics to Exception Handling"
seoTitle: "ObjectMapper"
seoDescription: "ObjectMapper"
datePublished: Wed Jan 07 2026 01:57:19 GMT+0000 (Coordinated Universal Time)
cuid: cmk3dbrc9000202jm0qwj0ajn
slug: deep-dive-into-objectmapper-from-internal-mechanics-to-exception-handling
tags: objectmapper

---

```java
try {
    objectMapper.readValue(request, A::class.java)
} catch (e: JsonProcessingException) {
    throw CoreException(InMemoryExceptionCode.FAILED_PARSE_JSON)
} catch (e: JsonMappingException) {
    throw CoreException(InMemoryExceptionCode.FAILED_MAP_TO_SCHEMA)
}
```

As a web backend developer, you quickly realize how frequently `ObjectMapper` is used. In a typical Controller, the Jackson library's `ObjectMapper` handles the serialization of JSON requests into our desired object types. It is also common practice for developers to inject `ObjectMapper` to handle data when interacting with Redis.

To be honest, I didn't find the code above strange at first. In fact, based on my experience, I thought writing it this way was the only way to perfectly control the frequent exceptions thrown by `ObjectMapper`. This assumption likely stemmed from not looking deeply enough into the internal libraries that Spring relies on.

---

## How ObjectMapper Operates in a Controller

1. **Identify Content-Type**: When an HTTP request arrives, the server checks the `Content-Type`.
    
2. **Trigger Jackson**: If it is `application/json`, Jackson’s `ObjectMapper` is invoked.
    
3. **Deserialization**: `ObjectMapper` converts the JSON string into an instance of the specified class.
    

> **Serialization** &gt; - The process of converting an object in memory into a format that can be stored or transmitted.
> 
> * Converting Java/Kotlin objects into JSON strings or byte streams.
>     
> 
> **Deserialization** &gt; - The process of converting stored or transmitted data back into an object that can be used in memory.
> 
> * Converting JSON strings back into Java/Kotlin objects.
>     

A quick question: Does `ObjectMapper` also work when communicating with a Database?

* **No.** `ObjectMapper` is primarily responsible for converting JSON objects during HTTP communication.
    
* When communicating with a DB, **JPA** maps objects (Entities) based on table metadata, while **MyBatis** maps SQL results to objects.
    

So far, we know `ObjectMapper` deserializes JSON strings into objects. However, data actually arrives at the server as a **byte stream**, not a raw JSON string. Let’s look at where that transformation happens.

> 1. **Client sends JSON**: `{"name": "Kim", "age": 25}`
>     
> 2. **Network Transmission**:
>     
>     * The HTTP request is converted into a byte stream.
>         
>     * `Content-Type: application/json` is included in the header.
>         
> 3. **Spring Server**:
>     
>     * Byte stream → Converted to JSON string.
>         
>     * `ObjectMapper` converts the JSON string → Kotlin/Java object.
>         

---

## Who Converts Byte Streams to JSON?

In Spring MVC, the `HttpMessageConverter` is responsible for converting the HTTP request byte stream into a JSON string.

1. **Arrival**: The HTTP request byte stream arrives.
    
2. **Read**: The byte stream is read via `ServletInputStream`.
    
3. **Process**: `MappingJackson2HttpMessageConverter` takes over.
    
    * It uses `ObjectMapper` internally.
        
    * It converts the byte stream into a string using `InputStreamReader`.
        
4. **Convert**: `ObjectMapper` converts the JSON string into an object.
    

Looking at Spring's default configuration, we can see where converters are added in the WebMvc configuration classes.

```java
// Inside WebMvcConfigurationSupport
if (jackson2XmlPresent) {
    Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.xml();
    if (this.applicationContext != null) {
        builder.applicationContext(this.applicationContext);
    }
    messageConverters.add(new MappingJackson2XmlHttpMessageConverter(builder.build()));
    // ...
}
```

If you examine `MappingJackson2HttpMessageConverter`, it inherits from `AbstractJackson2HttpMessageConverter`. Inside that class, you can see the part where it receives the byte stream through an `InputStream` object.

```java
private Object readJavaType(JavaType javaType, HttpInputMessage inputMessage) throws IOException {
    // ...
    try {
        InputStream inputStream = StreamUtils.nonClosing(inputMessage.getBody());
        // ...
```

### End-to-End Flow Summary

* **Initial HTTP Processing**: Request arrives → Tomcat Connector assigns a thread → Request parsed into `HttpServletRequest`.
    
* **FilterChain**: Passes through `DelegateFilter` → `SecurityFilter` → etc.
    
* **DispatcherServlet**: `doDispatch()` is called → Find Handler via `HandlerMapping` → Execute via `HandlerAdapter`.
    
* **@RequestBody Processing**: Read body via `ServletInputStream` → `HttpMessageConverter` (Byte Stream → JSON → Object).
    
* **@ResponseBody Processing**: Return value processed by `HttpMessageConverter` (Object → JSON → Byte Stream) → Response written via `ServletOutputStream`.
    

---

## Moving Beyond Try-Catch: ObjectMapper Configurations

Let's analyze the exceptions handled in the original code:

1. `JsonProcessingException`: The root exception for Jackson. It covers all issues during JSON parsing or generation (syntax errors, incomplete strings). Since these are often "human errors" from the client side, we might still need some level of control here.
    
2. `JsonMappingException`: A subclass of `JsonProcessingException` for specific mapping issues (type mismatch, missing fields).
    
    * *Realization*: My original code caught `JsonProcessingException` first. Since it's the parent, `JsonMappingException` would never be caught in its own block. I should have analyzed the library hierarchy more carefully.
        

---

## Recommended ObjectMapper Configurations

Instead of messy try-catches, we can configure `ObjectMapper` to handle many common issues gracefully.

### 1\. Basic & Deserialization Settings

```kotlin
objectMapper.apply {
    setSerializationInclusion(JsonInclude.Include.NON_NULL) // Exclude nulls
    configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false) // Ignore unknown fields
    configure(DeserializationFeature.READ_UNKNOWN_ENUM_VALUES_AS_NULL, true) // Unknown Enum as null
    registerModule(JavaTimeModule()) // Support Java 8 Date/Time
}
```

### 2\. Custom Wrapper for Clean Code

Since we cannot completely eliminate `JsonProcessingException`, I recommend using a Wrapper class:

```kotlin
@Component
@Slf4j
class JsonConverter(private val objectMapper: ObjectMapper) {
    fun <T> fromJson(json: String, type: Class<T>): Optional<T> {
        return try {
            Optional.ofNullable(objectMapper.readValue(json, type))
        } catch (e: JsonProcessingException) {
            log.error("JSON Conversion Failed: {}", e.getMessage())
            Optional.empty()
        }
    }
}
```

---

## Supplemental: Issues with MappingJackson2HttpMessageConverter

I’d like to share an issue discussed in my dev community regarding `MappingJackson2HttpMessageConverter`.

**The Problem**: When communicating with an external partner API using `WebClient` or `RestClient`, an error occurred stating the request body was empty. Interestingly, it worked fine with `OpenFeign` or when sending data as a raw `String`.

**The Cause: Chunked Transfer Encoding**. When you pass an `Object` directly to the request body, `MappingJackson2HttpMessageConverter` triggers. In Spring 6.1+, to optimize memory, `RestTemplate` and `RestClient` no longer buffer the request body by default. Consequently, the `Content-Length` header is not set, and the data is sent using `Transfer-Encoding: chunked`.

If the external partner's server does not support chunked encoding, it fails.

**Solutions**:

1. Override `getContentLength` in `MappingJackson2HttpMessageConverter`.
    
2. Wrap the `ClientHttpRequestFactory` with `BufferingClientHttpRequestFactory` to force buffering (and thus set `Content-Length`).
    
3. Send the data as a `String` (which uses `StringHttpMessageConverter` that provides a `Content-Length`).
    

This change was documented in the [Spring Framework 6.1 Release Notes](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-6.1-Release-Notes) to optimize memory usage. It’s a crucial reminder that keeping up with release notes is just as important as writing clean code!