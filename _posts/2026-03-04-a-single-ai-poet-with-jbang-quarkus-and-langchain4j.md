---
layout: post
title: A single file AI Poet with JBang, Quarkus and LangChain4j
tags: jbang quarkus langchain4j ai
---

I've been spending time playing about with LLM based AI[^fn1] lately (who hasn't?) and I've been wanting to get under the hood of how exactly things like agents work. As part of this, 
I dusted off my 25-ish year old copy of "[Constructing Intelligent Agents Using Java](https://www.amazon.co.uk/Constructing-Intelligent-Agents-Using-Java)" to remind myself how Agents used to be built (It's more similar than you might
think and is probably worth a blog post on its own at some point) and from that I decided to look into how to construct an intelligent agent using Java in 2026. I also wanted to see how
lightweight I could make in the second of what will probably be several "what can you do in a single file with JBang" posts.

### A brief glance at the Java AI landscape

The first thing I needed to do was look at what platforms I could leverage. It turns out that there are two, very familiar, standouts - Spring AI and Quarkus with LangChain4J (in what feels like
a dynastic sports rivalry that crosses generations). As someone who went over to the Spring side back in the version 1.2 time frame, you'd think I'd be kicking Spring AI's tires 
(and one day I probably will), but this time I thought I'd give Quarkus a try (mostly because the [A2A Protocol SDK](https://github.com/a2aproject/a2a-java) is tilted towards Quarkus 
and I plan on looking into that later, but also because no-one is paying me to do this so I might as well learn something extra while I'm at it). 

So with that all out of the way, here's the Poet AI example from [Quarkiverse](https://docs.quarkiverse.io/quarkus-langchain4j/dev/quickstart.html) but as a JBang Script

{% highlight java %}
///usr/bin/env jbang "$0" "$@" ; exit $?
//RUNTIME_OPTIONS --add-opens java.base/java.lang=ALL-UNNAMED
//RUNTIME_OPTIONS -Dquarkus.langchain4j.openai.base-url=https://openrouter.ai/api/v1
//RUNTIME_OPTIONS -Dquarkus.langchain4j.openai.chat-model.model-name=arcee-ai/trinity-large-preview:free
//RUNTIME_OPTIONS -Dquarkus.langchain4j.openai.api-key=YOUR_KEY_HERE
//JAVA 25+
//DEPS io.quarkus:quarkus-bom:3.32.1@pom
//DEPS io.quarkus.platform:quarkus-langchain4j-bom:@pom
//DEPS io.quarkiverse.langchain4j:quarkus-langchain4j-openai:1.7.4
//DEPS io.quarkus:quarkus-picocli

import jakarta.enterprise.context.ApplicationScoped;
import dev.langchain4j.service.SystemMessage;
import dev.langchain4j.service.UserMessage;
import io.quarkiverse.langchain4j.RegisterAiService;
import jakarta.enterprise.context.control.ActivateRequestContext;
import jakarta.inject.Inject;
import picocli.CommandLine;
import picocli.CommandLine.Command;
import picocli.CommandLine.Parameters;


@Command(name = "poem", mixinStandardHelpOptions = true)
class Poem implements Runnable {

        @RegisterAiService
        @SystemMessage("You are a professional poet")
        @ApplicationScoped
        public interface MyAiService {
                @UserMessage("""Write a poem about {topic}. The poem should be {lines} lines long.""")
                String writeAPoem(String topic, int lines);
        }


    @Parameters(paramLabel = "<topic>", defaultValue = "quarkus",description = "The topic.")
    String topic;

    @CommandLine.Option(names = "--lines", defaultValue = "4",description = "The number of lines in the poem.")
    int lines;

    @Inject
    MyAiService myAiService;

    @Override
    public void run() {
        IO.println(myAiService.writeAPoem(topic, lines));
    }

} 
{% endhighlight %}

So a lot more code than last time. But still not a whole lot. Let's go through it

{% highlight java %}
///usr/bin/env jbang "$0" "$@" ; exit $?
//RUNTIME_OPTIONS --add-opens java.base/java.lang=ALL-UNNAMED
//RUNTIME_OPTIONS -Dquarkus.langchain4j.openai.base-url=https://openrouter.ai/api/v1
//RUNTIME_OPTIONS -Dquarkus.langchain4j.openai.chat-model.model-name=arcee-ai/trinity-large-preview:free
//RUNTIME_OPTIONS -Dquarkus.langchain4j.openai.api-key=YOUR_KEY_HERE 
{% endhighlight %}

After the usual JBang start line, we get to work by setting some runtime options. The first one (`--add-opens java.base/java.lang=ALL-UNNAMED`) is kind of interesting. If we don't do this we get

{% highlight bash %}
Exception in thread "vert.x-internal-blocking-1" java.lang.IllegalAccessError: module java.base does not open java.lang to unnamed module @e98770d; to use the thread-local-reset capability on Java 24 or later, use this JVM option: --add-opens java.base/java.lang=ALL-UNNAMED
{% endhighlight %}
when the app terminates. Not sure why. Don't really care. I just want it to work.[^fn2] That runtime option makes it work.

Following that, we set up some options for LangChain4J. As I wanted this keep this self contained in a single file, I needed to set those options as runtime
options instead of in an application.properties file.

I'm using OpenRouter to access the LLM model and with LangChain4J that means using the OpenAI integration but changing the base url 
(I am wondering how this is supposed to work if I want to have both OpenRouter on OpenAI configured in the same app and swap between them - a challenge for another day).

I'm using [arcee-ai/trinity-large-preview:free](https://openrouter.ai/arcee-ai/trinity-large-preview:free) as the model, partly because "It excels in creative writing, storytelling, role-play, chat scenarios...", but 
mostly because it's free to use. (I am wondering what happens if I want to swap models on the fly - again, a challenge for another day)

Finally, it's time to set the API key. You'll need your own. Please note, "Key Management Best Practices" is something beyond the scope of this post (though think before you commit to git).

Skipping over the dependencies and their imports, we come to our application's class

{% highlight java %}
@Command(name = "poem", mixinStandardHelpOptions = true)
class Poem implements Runnable {
{% endhighlight %}

The @Command annotation is because we're using Picocli to handle parsing the command line and implementing Runnable so that Quarkus runs our app. 
Normally the command class would be it's own thing but here we're reusing the main application class to simplify things a bit.

Next up, create an interface that will be used by Quarkus to generate an "AI Service"

{% highlight java %}
@RegisterAiService
@SystemMessage("You are a professional poet")
@ApplicationScoped
public interface MyAiService {
        @UserMessage("""Write a poem about {topic}. The poem should be {lines} lines long.""")
        String writeAPoem(String topic, int lines);
}
{% endhighlight %}

If you've ever used a Spring Data repository then this pattern should look eerily familiar. But here we're connecting to a LLM model instead of a database. It's all nice and simple.

{% highlight java %}
@Parameters(paramLabel = "<topic>", defaultValue = "quarkus",description = "The topic.")
String topic;

@CommandLine.Option(names = "--lines", defaultValue = "4",description = "The number of lines in the poem.")
int lines;

@Inject
MyAiService myAiService;
{% endhighlight %}

Here, we're getting what (if anything) was passed to us at the command line and an instance of that MyAiService interface that Quarkus built for us, all configured to talk to OpenRouter.

Finally

{% highlight java %}
@Override
public void run() {
    IO.println(myAiService.writeAPoem(topic, lines));
} 
{% endhighlight %}

The bit Quarkus will use to run our application. So our app will take the topic and number of lines (or their defaults), generate a prompt from the @ServiceMessage and
@UserMessage, send it off to OpenRouter for inference and the print the results. So `chmod +x Poem.java` and then `./Poem.java --lines=8 "Perpetual motion"` and you should get something similar to

{% highlight bash %}
[jbang] Building jar for Poem.java...
[jbang] Post build with io.quarkus.launcher.JBangIntegration
[jbang] Quarkus augmentation completed in 2117ms
__  ____  __  _____   ___  __ ____  ______
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/
2026-03-04 13:50:45,553 INFO  [io.quarkus] (main) quarkus 999-SNAPSHOT on JVM (powered by Quarkus 3.32.1) started in 0.549s.
2026-03-04 13:50:45,559 INFO  [io.quarkus] (main) Profile prod activated.
2026-03-04 13:50:45,560 INFO  [io.quarkus] (main) Installed features: [cdi, langchain4j, langchain4j-openai, picocli, qute, rest-client, rest-client-jackson, smallrye-context-propagation, vertx]
In timeless dance, the orbs revolve
Eternal spin, a mystery solved
Science dreams of endless flow
Perpetual, to and fro
A vision of perpetual grace
A law of physics to embrace
Forever turning, a wondrous sight
The fuel of dreams, a boundless light
2026-03-04 13:50:50,227 INFO  [io.quarkus] (main) quarkus stopped in 0.021s
{% endhighlight %}

### Wait, how does this run?

You might have noticed something. We don't have a main method (in any of the now accepted styles). We only have a `Runnable::run`, so there is the question of how exactly does the above code 
run by itself.

It turns out JBang has helped us out here. JBang has an experimental feature called ["Build Integration"](https://www.jbang.dev/documentation/jbang/latest/integration.html#build-integration-experimental) which
is aware of Quarkus and is able get our app running without us having to define an explicit main method.

[^fn1]: AI has long been a hobby of mine. I used to spend time playing with older AI tech like Eliza/Alice Bots, Semantic Web/Knowledge Graphs, rules engines/expert systems and even genetic algorithms. Oddly, I've got a sneaking suspicion that at least some of the old tech is going to make a quiet comeback at some point, partly because people like determinism in their software and partly because LLMs should make building with that tech a whole lot easier.

[^fn2]: I could say that about a lot of things when it comes to almost anything related to JPMS or the clamping down in the name of security that has been taking place since 11. 
