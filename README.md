# Don't be careless with Groovy (un)checked exceptions

Groovy language introduced many interesting features from Java programmer's point of view. One of them is the lack of differentiation between checked and unchecked exceptions - a nice quality for programmers who struggled with checked exceptions requirements. It turns out, however, that using this neat Groovy feature with existing frameworks can sometimes be tricky which I've experienced lately.

I worked with relatively straightforward microservice written in Groovy. It was built on top of Spring Boot 1.3. Among others it used `@RestController` to define REST endpoints and `@ControllerAdvice` to handle exceptions thrown by controller or its internals.
Very simplified example code looks like this (complete source code can be downloaded from https://github.com/yu55/groovy-undeclared-throwable-demo ):

```groovy
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
class ExampleRestController {

    @RequestMapping(value = '/get')
    String get() {
        /*
         Lets imagine this exception is thrown somewhere from deepest layers of our service code
         and we don't have to be immediately aware of this.
          */
        throw new ExampleRestControllerException()
    }
}
```
```groovy
import org.springframework.web.bind.annotation.ControllerAdvice
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.ResponseBody

@ControllerAdvice
class ExampleRestControllerAdvice {

    @ExceptionHandler(value = ExampleRestControllerException.class)
    @ResponseBody
    String onExampleRestControllerException() {
        return 'ExampleRestControllerException handled in ControllerAdvice'
    }

    @ExceptionHandler(value = Exception.class)
    @ResponseBody
    String onException() {
        return 'Exception handled in ControllerAdvice'
    }
}
```
```groovy
class ExampleRestControllerException extends Exception {
}
```

When `ExampleRestControllerException` was thrown, it was handled by `onExampleRestControllerException()` method in `ExampleRestControllerAdvice` class. And this worked perfectly fine.
But I had to add a simple aspect after returning of `get()` method in `ExampleRestController` to do some additional stuff.
```groovy
@Aspect
@Component
class RestControllerAspect {

    @AfterReturning('execution(* org.yu55.ued.controller.ExampleRestController.get())')
    public void logServiceAccess(JoinPoint joinPoint) {
        // empty implementation for clarity reasons
    }
```

With this simple aspect introduced, the microservice began to behave strange. `ExampleRestControllerAdvice` started calling `onException()` method instead of `onExampleRestControllerException()` when `ExampleRestControllerException` was thrown. I didn't expect that. What happend?
When aspect for `ExampleRestController.get()` method is defined, the Spring generates a proxy for `ExampleRestController`. Since `ExampleRestController` doesn't implement any interface, Spring uses CGLIB library instead of `java.lang.reflect.Proxy` for a dynamic proxy generation [[1](http://docs.spring.io/spring/docs/4.2.4.RELEASE/spring-framework-reference/html/aop.html#aop-proxying)]. Generated proxy class name is similar to `ExampleRestController$$EnhancerBySpringCGLIB$$8d751c8@4481` and it's an `ExampleRestController` subclass which intercepts all methods calls. There is also another dynamically generated class created: `ExampleRestController$$FastClassBySpringCGLIB$$b01891ea` which is a `ExampleRestController` class wrapper. This wrapper class promises faster methods invocations than the Java reflection API [[2](https://dzone.com/articles/cglib-missing-manual)].
Spring is invoking `get()` method (via `sun.reflect` reflection classes) on enhancer class which then invokes `get()` method (via CGLIB `MethodProxy`) on fast class instance. When controllers `get()` method throws `ExampleRestControllerException` it's rethrown by fast class to an enhancer class and then the enhancer class `get()` method throws `java.lang.reflect.UndeclaredThrowableException` (which contains `ExampleRestControllerException` inside). This results in `ExampleRestControllerAdvice` matching `UndeclaredThrowableException` with `Exception` class and firing handler method different than expected.
![Sequence diagram](/../master/gutd-diagram.png?raw=true "Sequence diagram")
But why `UndeclaredThrowableException` is thrown? Because the `get()` method written in Groovy in fact didn't declare any checked exceptions that it could potentially throw. Proxy doesn't know anything about that controller which is written in Groovy and no throws declared means that no checked exception may occur. This behaviour also corresponds to `Proxy` documentation [[3](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Proxy.html)].
The issue can be fixed easily by adding checked exceptions to `get()` method definition or defining `ExampleRestControllerException` as runtime exception. Former solution technically works but isn't 'groovy' though and the latter is an approach that Spring prefers and is currently considered to be best practice for exception handling [[4](http://www.javacodegeeks.com/2012/03/why-should-you-use-unchecked-exceptions.html)].

At the end, I would not blame Groovy for this situation. Instead, I would rather point that despite the fact that Groovy simply don't care about checked exceptions, it doesn't mean that other libraries also don't care. This is not what most programmers think of first when using Groovy. Similar situation may happen with any other JVM language that 'ignores' checked exceptions.
Best thing we can do to protect ourselves from situations like this is to prepare tests carefully. Imagine no controller tests for `ExampleRestControllerException` case - whole application builds and runs perfectly on production until this special case occurs and controller simply returns a wrong answer to the client. Bug like this may not be so obvious and fast to track or fix.
