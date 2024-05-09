---
title: 'Azure Functions With Spring Cloud - Introduce'
date: 2024-05-01
draft: true
url: 'azure-functions-with-spring-cloud-introduce'
description: Create, run locally and deploy very first Azure Functions using Spring Cloud.
tldr: |
  Sometimes, especially when you try to run your very first "Hello World!" program, you encounter a lot of issues.
  I've been through this for you. I'll try to guide you through you first Azure Functions deployment.
tags: [ 'azure functions', 'spring', 'spring cloud' ]
---

## Glossary

To be well-prepared to read this article, we need to define a couple of buzzwords that everyone who wants to deploy
[Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/) should know.

What the heck are those _functions_? The crucial thing to remember is — **serverless**. We don't need to worry about
infrastructure. We only create code that covers all our business logic. That's all!

We can think of it like running scripts in the Linux Shell. If we want to run a script, we provide only input data; a
Shell script has some logic, and as a result, there is some output. Our favorite Linux distribution takes care of
running it properly. We don't need to maintain our own 'infrastructure' to run it.

Now, somehow, we need to connect our business logic with Azure Functions. This can be achieved by
using [Spring Cloud Function (Azure Adapter)](https://docs.spring.io/spring-cloud-function/docs/current/reference/html/azure.html#_microsoft_azure_functions).
With this adapter, we have a very convenient way to build, test locally, and deploy our application. This library
behaves like a typical Spring application, which delights us.

The described flow looks:

{{< figure src="/images/2024/05/01/1-glossary.png" title="Figure 1. Azure Functions and Spring Cloud Function flow" >}}

## Hello World!

Let's try our first "Hello World!" application. The flow will be pretty straightforward. We put some text as an input,
and we return its uppercased transformation. Just as simple as that.

Let's look at the flow:

{{< figure src="/images/2024/05/01/2-hello-world-flow.png" title="Figure 2. \"Hello World!\" application flow" >}}

To begin, we only need to generate a very basic Spring application using [start.spring.io](https://start.spring.io):

{{< figure src="/images/2024/05/01/3-start-spring-io.png" title="Figure 3. Generate application" >}}

{{< callout >}}
It's important to create an application with Java 17 as the maximum version. Java 21 is in preview mode for the Linux OS
within the Azure Functions ecosystem.
{{< /callout >}}

Here is the information about
all [supported Java versions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference-java?tabs=bash%2Cconsumption#supported-versions).

As a first step we need to add the `spring-cloud-function-adapter-azure` dependency. This allows us to integrate our
codebase with the Azure Functions.

```groovy

implementation 'org.springframework.cloud:spring-cloud-function-adapter-azure:4.1.1'
```

Our codebase is divided into two simple pieces.

1. `UppercaseFunction` with the business logic:

<!--@formatter:off-->
```java
@Component
public class UppercaseFunction implements Function<String, String> {
    @Override
    public String apply(String message) {
        // Get input data to uppercase
        String payload = hasText(message) ? message : "default";
        // Transform using uppercase function
        return payload.toUpperCase();
    }
}
```
<!--@formatter:on-->

At this point, we can run the application locally (without integrating with Azure Functions) using
the `spring-cloud-starter-function-web` dependency. Spring Cloud Function automatically exposes all functions as HTTP
endpoints ([Standalone Web Applications](https://docs.spring.io/spring-cloud-function/docs/current/reference/html/spring-cloud-function.html#_standalone_web_applications)):

```groovy

implementation 'org.springframework.cloud:spring-cloud-starter-function-web:4.1.1'
```

We can check our business logic and output before we deploy or even test the application with Azure Functions by calling
exposed endpoints:

{{< figure src="/images/2024/05/01/4-post-uppercase.png" title="Figure 4. POST /uppercase" >}}

{{< figure src="/images/2024/05/01/5-get-uppercase.png" title="Figure 5. GET /uppercase" >}}

2. `AzureUppercaseHandler` describes the trigger when the business logic is called:

<!--@formatter:off-->
```java
@Component
public class AzureUppercaseHandler {
    private final UppercaseFunction uppercaseFunction;

    public AzureUppercaseHandler(UppercaseFunction uppercaseFunction) {
        this.uppercaseFunction = uppercaseFunction;
    }

    @FunctionName("uppercase")
    public String execute(
            @HttpTrigger(
                    name = "request",
                    // Run for GET and POST requests
                    methods = {GET, POST},
                    // Without authorization
                    authLevel = ANONYMOUS
            ) HttpRequestMessage<Optional<String>> request,
            ExecutionContext context
    ) {
        String body = request.getBody().orElse(null);

        // We can log something
        context.getLogger()
                .info("Trying to uppercase string [%s]...".formatted(body));

        return uppercaseFunction.apply(body);
    }
}
```
<!--@formatter:on-->

## Run locally

With the application at that stage, we can try to run it locally using
an [Azure Functions Plugin for Gradle](https://github.com/microsoft/azure-gradle-plugins/tree/master/azure-functions-gradle-plugin).

{{< callout >}}
It's very important (at least right now) to use the following plugin in version `1.14.0`. This is due
to [poorly handled](https://github.com/microsoft/azure-gradle-plugins/issues/177) `pricingTier` and `region` parameters.
{{< /callout >}}

Let's add the Gradle plugin and configure it, putting (for now) only an `appName` property:

```groovy
azurefunctions {
    appName = 'uppercase'
}
```

To run our application locally, we need to run a Gradle task which is provided by the _Azure Functions Plugin for
Gradle_.

```shell
./gradlew clean azureFunctionsRun
```

The following output should be produced:

{{< figure src="/images/2024/05/01/6-gradle-run-locally.png" title="Figure 6. Output of the local run" >}}

So let's try to call our `uppercase` function. The entry point is `http://localhost:7071/api/uppercase`.

💥 BOOOM! 💥

Unfortunately, our application isn't working. We received the following exception:

```
INFO 1095179 --- [uppercase] [pool-2-thread-1] o.s.boot.SpringApplication               : Started application in 1.595 seconds (process running for 17.217)
Executed 'Functions.uppercase' (Failed, Id=5555b7a4-5925-4a92-b1ee-6f2df865cb36, Duration=1990ms)
System.Private.CoreLib: Exception while executing function: Functions.uppercase. System.Private.CoreLib: Result: Failure
Exception: IllegalStateException: Failed to retrieve Bean instance for: class codes.piotrooo.azurefunctions.service.AzureUppercaseHandler. The class should be annotated with @Component to let the Spring framework initialize it!
Stack: java.lang.IllegalStateException: Failed to initialize
     at org.springframework.cloud.function.adapter.azure.AzureFunctionInstanceInjector.getInstance(AzureFunctionInstanceInjector.java:80)
     at com.microsoft.azure.functions.worker.binding.ExecutionContextDataSource.getFunctionInstance(ExecutionContextDataSource.java:103)
     at com.microsoft.azure.functions.worker.broker.JavaMethodInvokeInfo.invoke(JavaMethodInvokeInfo.java:20)
     at com.microsoft.azure.functions.worker.broker.EnhancedJavaMethodExecutorImpl.execute(EnhancedJavaMethodExecutorImpl.java:22)
     at com.microsoft.azure.functions.worker.chain.FunctionExecutionMiddleware.invoke(FunctionExecutionMiddleware.java:19)
     at com.microsoft.azure.functions.worker.chain.InvocationChain.doNext(InvocationChain.java:21)
     at com.microsoft.azure.functions.worker.broker.JavaFunctionBroker.invokeMethod(JavaFunctionBroker.java:125)
     at com.microsoft.azure.functions.worker.handler.InvocationRequestHandler.execute(InvocationRequestHandler.java:34)
     at com.microsoft.azure.functions.worker.handler.InvocationRequestHandler.execute(InvocationRequestHandler.java:10)
     at com.microsoft.azure.functions.worker.handler.MessageHandler.handle(MessageHandler.java:44)
     at com.microsoft.azure.functions.worker.JavaWorkerClient$StreamingMessagePeer.lambda$onNext$0(JavaWorkerClient.java:94)
     at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:572)
     at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:317)
     at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1144)
     at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:642)
     at java.base/java.lang.Thread.run(Thread.java:1583)
Caused by: java.lang.IllegalStateException: Failed to retrieve Bean instance for: class codes.piotrooo.azurefunctions.service.AzureUppercaseHandler. The class should be annotated with @Component to let the Spring framework initialize it!
     at org.springframework.cloud.function.adapter.azure.AzureFunctionInstanceInjector.getInstance(AzureFunctionInstanceInjector.java:70)
     ... 15 more
```

The most important (and interesting) part in this stack trace is:

```
Failed to retrieve Bean instance for: class codes.piotrooo.azurefunctions.service.AzureUppercaseHandler.
The class should be annotated with @Component to let the Spring framework initialize it!
```

But how?! We added a `@Component` stereotype to the `AzureUppercaseHandler` class, so what is the reason for
complaining?

If we take a closer look at the generated output, we can notice multiple lines with the following message:

```
Searching for start class in manifest:
```

and:

```
org.springframework.cloud.function.utils.FunctionClassUtils -- Loaded Start Class: class org.springframework.cloud.function.web.RestApplication
org.springframework.cloud.function.utils.FunctionClassUtils -- Main class: class org.springframework.cloud.function.web.RestApplication
```

The main class that is being picked up is definitely **NOT** our application's starting point. Upon further
investigation and digging deeper, we arrive at
the [FunctionClassUtils](https://github.com/spring-cloud/spring-cloud-function/blob/main/spring-cloud-function-context/src/main/java/org/springframework/cloud/function/utils/FunctionClassUtils.java)
class.

If we examine
the `FunctionClassUtils.getStartClass()` [method](https://github.com/spring-cloud/spring-cloud-function/blob/main/spring-cloud-function-context/src/main/java/org/springframework/cloud/function/utils/FunctionClassUtils.java#L53-L61)
— in the Javadoc — we find the following description:

```
/**
 * Discovers the start class in the currently running application.
 * The discover search order is 'MAIN_CLASS' environment property,
 * 'MAIN_CLASS' system property, META-INF/MANIFEST.MF:'Start-Class' attribute,
 * meta-inf/manifest.mf:'Start-Class' attribute.
 */
```

This gives us a hint that we don't have an entry point for our application to load all the needed classes. We need to
create a `MANIFEST.MF` file with the `Start-Class` header. The easiest way to create it in the generated `JAR` is by
putting it in the `build.gradle` file.

```groovy
jar {
    manifest {
        attributes(
                'Start-Class': 'codes.piotrooo.azurefunctions.UppercaseApplication'
        )
    }
}
```

Now, we can try to run the application once again. Let's try to uppercase something!

{{< callout >}}
If there is an error, check if the previous application releases the port. In the Gradle output, you can see a message:
`Port 7071 is unavailable. Close the process using that port, or specify another port using --port [-p].`
You may need to terminate a previously started process that uses port `7071`.

The `netstat -tulpn | grep LISTEN` command could be useful.
{{< /callout >}}

<!--@formatter:off-->
{{< figure src="/images/2024/05/01/7-locally-uppercased.png" title="Figure 7. Locally uppercased using Azure Functions" >}}
<!--@formatter:on-->

In the application logs we can also see our log statement:

```
Trying to uppercase string [running locally!]...
```

Perfect! 👌

At this point, it's also noteworthy that the `version` — specified in the `build.gradle` file or through command line
parameters (`-Pversion=X`) — **is required**. Without passing the `version` parameter, we may encounter the following
error:

```
Caused by: org.gradle.api.GradleException: Cannot package functions due to error: Azure Functions entry point not found, plugin will exit.
        at com.microsoft.azure.plugin.functions.gradle.task.PackageTask.build(PackageTask.java:56)
```

The `GradleProjectUtils` [class](https://github.com/microsoft/azure-gradle-plugins/blob/5902c7744ddbf821bb33875ab5bd422e2e07a312/azure-functions-gradle-plugin/src/main/java/com/microsoft/azure/plugin/functions/gradle/util/GradleProjectUtils.java#L56-L57)
in the _Azure Functions Plugin for Gradle_ uses a Gradle `version` property. Without this defined property, the plugin
cannot determine the build artifact file.

[//]: # (todo translations)

## Deploy to Azure

Before you try to deploy something, you must be sure that you are connected to the correct Azure subscription.
You can check it by:

{{< figure src="/images/2024/05/01/8-azure-account-show.png" title="Figure 8. Show current Azure account" >}}

{{< callout >}}
To log in to the Azure and set Azure subscription, you may need to run the following commands:

1. `az login`
2. `az account set --subscription <subscription-id>`

{{< /callout >}}

To deploy something, we need to make some adjustments to the _Azure Functions Plugin for Gradle_.

```groovy
azurefunctions {
    resourceGroup = 'PO'
    appName = 'uppercase'
    region = 'westeurope'
    runtime {
        os = 'linux'
        javaVersion = '17'
    }
    auth {
        type = 'azure_cli'
    }
    appSettings {
        FUNCTIONS_EXTENSION_VERSION = '~4'
    }
}
```

With this configuration, the plugin will create all necessary objects in the Azure Resource Group, which is called `PO`
in our sample.

The Gradle plugin provides us with a task called `azureFunctionsDeploy`. Let's try it by calling:

```shell
./gradlew clean azureFunctionsDeploy
```

{{< figure src="/images/2024/05/01/9-azure-deploy.png" title="Figure 9. Successfully deployed Azure Functions" >}}

Let’s take a closer look at what was created on Azure:

{{< figure src="/images/2024/05/01/10-azure-created-resources.png" title="Figure 10. Created resources" >}}

The _Azure Functions Plugin for Gradle_ handled everything for us. It created a `Function App` named _uppercase_ and
also set up the necessary `Storage account` for internal files. Additionally, it created `Application Insight`,
providing many features to enhance the performance, reliability, and quality of our application.

Now it's demo time! Let's try converting something to uppercase, but this time — **using Azure Functions**. In the
previously generated output, we can see an endpoint to make a request:

{{< figure src="/images/2024/05/01/11-running-on-azure.png" title="Figure 11. Uppercased on Azure" >}}

The generated log statements are also available in the Azure Portal:

{{< figure src="/images/2024/05/01/12-azure-logs.png" title="Figure 12. Azure Functions logs" >}}

We carried out our mission! The function is up and running, and what is more, we don't even create any virtual machine,
Kubernetes cluster or any of the 'infrastructure' stuff. It's awesome! That's the power of the Azure Functions and
general serverless programming.

---

The code which covers this article is available on
the [GitHub repository](https://github.com/piotrooo/piotrooo.codes-samples/tree/main/azure-functions/azure-functions-uppercase).
