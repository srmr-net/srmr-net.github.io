---
layout: post
title: A single file API with JBang and Spring Boot
tags: jbang spring
---

The other day I came across a comment along the lines of "even a single endpoint requires at least 3 different files with Spring Boot". To be fair, the person writing made it clear that they weren't a Java/Spring dev and if you follow most (if not all) of the tutorials out there then you will get the impression that there isn't a lightweight way to create an endpoint. So I thought a post about how you can create an API with just a single .java file would be a good way to start blogging again.

What you'll need:

- Modern Java (25+) 
- JBang

So, without much further ado, here it is:

{% highlight java %}
///usr/bin/env jbang "$0" "$@" ; exit $?
//DEPS org.springframework.boot:spring-boot-starter-web:4.0.2

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.*;
import org.springframework.web.servlet.function.*;
import org.springframework.web.servlet.config.annotation.*;

import static org.springframework.web.servlet.function.RouterFunctions.*;
import static org.springframework.web.servlet.function.RequestPredicates.*;
import static org.springframework.web.servlet.function.ServerResponse.*;

@SpringBootApplication
@ComponentScan("disable")
class App {

        @Bean
        RouterFunction<ServerResponse> routes() {
                return route(GET("/"), (req) -> ok().body("Hello, World"));
        }

        void main(String[] args) {
                SpringApplication.run(this.getClass(), args);
        }
}
{% endhighlight %}

That's it. Just `chmod +x App.java` and then `./App.java` and you have a working endpoint on port 8080. Still more boilerplate than I'd like, but a lot better than a pom.xml, an application class and an a controller class. With that said, there are some things beyond it being a single file that are worth noting.

### "Disabling" Component Scanning

You'll notice `@ComponentScan("disable")` annotation on the class. This doesn't actually disable component scanning. What it actually does is cause Spring to try and scan a package called `disable`. As there aren't any classes in that package Spring carries on its normal startup.

Why do we have to do this? You'll notice that we don't declare what package the class belongs to. Which means the class is in the "unnamed package". By default Spring Boot will try and component scan the package the application class is in. But Spring goes nuts when that happens - it tries to scan the entire world and will run into missing dependencies. 

(It's not surprising that Spring doesn't handle the unnamed package very well. Until recently with the work on making Java friendlier for beginners, placing a class in the unnamed package was generally considered to be very poor form in almost all circumstances)

### Router Functions

You'll notice it's not a traditional Controller class. Instead we define our endpoint with a RouterFunction

{% highlight java %}
        @Bean
        RouterFunction<ServerResponse> routes() {
                return route(GET("/"), (req) -> ok().body("Hello, World"));
        }
{% highlight %}

A RouterFunction is Function that routes a request to HandlerFunction, which is a Function that takes a request and returns a response. So in the above `(req) -> ok().body("Hello, World") is our HandlerFunction. `GET("/")` is a call to a utility function that creates a RouterPredicate. Here, it's returning a predicate that will return true if the request is a HTTP GET request for the root page. Another uility function, `route`, is what builds the actual RouterFunction, from the RequestPredicate and the RequestHandler.
