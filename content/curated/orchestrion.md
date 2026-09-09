+++
title = "Observability made easy: Serverless and Datadog Orchestrion"
date = "2026-09-10"

[taxonomies]
tags = ["Golang", "Observability", "Datadog"]

[extra]
comment = true
+++

## init()
In this blog post, I'm going to shed some light on a pretty new Datadog product (released about a year ago. but still under active development(!)) called Orchestrion. The goal is to highlight some nice technics that I found recently in order to achieve good observability traces in a production environment with a minimum effort.
The language of reference in this blog post is Golang, but despite that you can get a general idea of how Orchestrion can be leveraged.
Some other languages that are currently supported are Rust, Java, Python, Node.js, Ruby and more.[1]

Let's say that you have a brand-new service in your preferred cloud provider( GCP, AWS, etc ) and you want to start getting traces to understand the bottlenecks of your application and be able to debug those if needed.

The "classic" way (in Golang) to do so is to decide which telemetry provider you want to choose(f.e Datadog, Otel), get your preferred programming language's client sdk, and start instrumenting your code. To do so you need to:
1. Start in your code the client sdk and of course there you need to spend some time to define a correct software design for the new component, like where it is gonna be located, how, and all your language's specific best practices.
2. Start the tracer Span that you want inside each component and function that you want to monitor.
3. Add attributes and do the correct error handling in the cases of errors.
4. Don't forget to close the Span at the end.

Bottom line, there is a lot of work included. What if you could pass all this work? Here is where Orchestrion comes into play.

Orchestrion is a tool that let you do "automatic" instrumentation for your code, meaning that you don't need to do _most_ of the things mentioned above, and the tool will do those for you. What it does in a few words is that it automatically creates all
the spans for you, just by adding a "key" function comment directive(f.e _`dd:span`_) where you want to get traces. The rest are done by the tool.

## Set up

Before we dive in lets first of all set up the tool locally and in our repository. For that we will need to run locally:

```bash
go install github.com/DataDog/orchestrion@latest
```
this should install the tool, so _`which orchestrion`_ should return the path of the tool.

Then the important step is to include the tool in our _`go.mod`_ file. For that we will need to run:
```bash
orchestrion pin
```
this should create a new file called _`orchestrion.tool.go`_ that includes the necessary imports for the tool to work.

Add and commit all these to your repo if you want to.

## Infrastructure

To start with we need to define our infrastructure. Since most of us use docker containers, we will use that as an example. So we have already a docker image that we use to deploy our app service in our cloud provider or locally via docker. What's next? Spin up a [Datadog Agent](https://docs.datadoghq.com/agent/?tab=Container+platforms) in order to collect our data and integrate Orchestrion in our docker image. So, to shortly explain it, the Orchestrion will do the generation of the traces in code and those data will be collected by the Datadog Agent, which will send them to our Datadog Site, in order to visualize them.

> Terminology: A metrics "Collector" is something pretty common in Observability Tools. For example two of the most common tools, Datadog and OTel, have a "Collector" component that is responsible for collecting the data and later on send them to the DD's "Backend". The "Backend" is the one that is responsible for storing the data and providing the UI for visualization.

## Architecture

### Sidecar or In-Process

Since we have our service as a Docker Container, we have two options for our observability services:

1. Sidecar: Deploys the Datadog Agent in a separate container alongside your app container. Have a separate service for the observability that will communicate with our app service.
2. In-Process: Wraps your application container with the Datadog Agent. That means that we can keep our current design and have both application and observability in one service, meaning one Docker Container.

The option that fits better my needs is the In-Process observability, since:

1. Lower cost overhead - there is one service/container running for both causes; no extra vCPU/memory to scale/bill, so less overhead.
2. Simpler setup - there is no need for extra configurations on an extra container, setting limits, etc.
3. Direct stdout/stderr access since we have one service, where in sidecar we need to utilize a shared volume in order to share the data from one service to the other.

A Sidecar would have been more fitting if we wanted to:

- Add multiple containers per app service, so that two app services can send observability data to one observability service.
- Require strict runtime isolation for the Datadog Agent.
- Want to tune Datadog Agent CPU/mem separately from the application service.

### Serverless Datadog Agent

And here is the next "trick" that we use to make our life easier and achieve a high quality outcome for our simple use case. We just need to add the _`serverless-init`_ Docker Image as an extra "[stage](https://docs.docker.com/get-started/docker-concepts/building-images/multi-stage-builds/)" in our application's Docker Image. Doing so we are able to spin up a Datadog Agent in the localhost of our app's service, that will be responsible for collecting our data and later on pipe them to our Datadog Site.

I am talking about this image: _`datadog/serverless-init:latest-alpine`_

And an example Docker image for Golang can be this one:

<details>
<summary>Example Dockerfile</summary>

``` dockerfile
# Build stage
FROM golang:1.24.8-alpine AS builder
RUN apk update && \
    apk upgrade --no-cache && \
    apk add --no-cache ca-certificates && \
    update-ca-certificates && \
    rm -rf /var/cache/apk/*

RUN go install github.com/DataDog/orchestrion@v1.6.1

WORKDIR /src
COPY . .

RUN orchestrion go build -o app cmd/main.go

# Final stage
FROM alpine:latest  
# For Security, Update packages to patch known vulnerabilities in the base image.
# Clean up apt cache to keep the image size down.
RUN apk update && \
    apk upgrade --no-cache && \
    apk add --no-cache ca-certificates && \
    update-ca-certificates

# Copy Go binary
COPY --from=builder /src/app /usr/local/bin/app

# Copy Datadog init
COPY --from=datadog/serverless-init:latest-alpine /datadog-init /usr/local/bin/datadog-init

ENV DD_SERVICE=app \ 
DD_ENV=development \
DD_TRACE_STARTUP_LOGS=true \
DD_TRACE_DEBUG=true \
DD_TRACE_SAMPLING_RULES='[{"service": "app", "sample_rate": 0.1}]'

# Set entrypoint and default command
ENTRYPOINT ["/usr/local/bin/datadog-init"]
CMD ["/usr/local/bin/app"]

EXPOSE 8080
```
</details>

What do we see in this Dockerfile happening? 

The magic lines regrading the usage of the Orchestrion are:
```dockerfile
RUN go install github.com/DataDog/orchestrion@v1.6.1

WORKDIR /src
COPY . .

RUN orchestrion go build -o app cmd/main.go
```
where we download and install as executable the Datadog Orchestrion tool and then we use it to build our application. The latter means that the final binary will include "automagically" all the tracing instrumentation needed in our codebase which is autogenerated by the Orchestrion tool.  

And the magic lines regarding the spin up of the in-process Datadog Agent are: 
```dockerfile
# Copy Datadog init
COPY --from=datadog/serverless-init:latest-alpine /datadog-init /usr/local/bin/datadog-init

# Set entrypoint and default command
ENTRYPOINT ["/usr/local/bin/datadog-init"]
```
where we just copy from the datadog image the _`datadog-init`_ binary and we set it as the entrypoint of our Docker Image. That's it! The _`entrypoint_` will always run before our defined _`cmd`_ command.

We defined some env vars in the Dockerfile, like the _`DD_TRACE_STARTUP_LOGS=true`_ and _`DD_TRACE_DEBUG=true`_ that will help us see what is happening with the tracer and [here](https://docs.datadoghq.com/tracing/trace_collection/library_config/go/#traces) you can find the complete list of available env vars. So if we check the logs of the container running our application we should see something like:
_`2025/09/24 08:42:19 Datadog Tracer v2.2.3 INFO: DATADOG TRACER CONFIGURATION`_
and the configuration of the Agent,
```json
{
  "date": "2025-09-24T08:42:19Z",
  "os_name": "Alpine Linux",
  "os_version": "unknown",
  "version": "v2.2.3",
  "lang": "Go",
  "lang_version": "go1.24.7",
  "env": "development",
  "service": "app",
  "agent_url": "http://localhost:8126/v0.4/traces",
  "debug": true
  .....
```
this _`"agent_url": "http://localhost:8126/v0.4/traces"`_, showcases that the Agent is running in the localhost.

And like that we have a Datadog Agent and tracing instrumentation in our application. Easy peasy.

## Orchestrion

Orchestrion give the ability by adding comments into our app's code, to instruct the tool to add tracing spans, define the span name, custom tags and ignore functions from tracing.
A simple Golang example is something like:
```go
func main() {
	trace()
	doNotTrace()
}

//dd:span custom_tag:tag_value span.name:trace
func trace() {
	fmt.Println("Hello, World!")
}

//orchestrion:ignore
func doNotTrace() {
	fmt.Println("Ignore me, World!")
}
```

and in order to see the magic of what the Orchestrion tool does to our code, we could use this command with the _`-work_` flag:
```bash
orchestrion go build -work main.go
```
you should see an output like:

```bash
WORK=/var/folders/b9/fppvks895kl9fq05bpfr382w0000gp/T/go-build4071752359
```

and then by feeding this to _`orchestrion diff_`:
```bash
orchestrion diff /var/folders/b9/fppvks895kl9fq05bpfr382w0000gp/T/go-build4071752359
```
We get the diff of what the tool has changed to our code.

A small sample but valuable from the output of the above mentioned commands is:
```markdown
//dd:span custom_tag:tag_value span.name:operationName
 func trace() {
+//line <generated>:1
+	{
+		ctx := __orchestrion_context.TODO()
+		var span *__orchestrion_tracer.Span
+		span, ctx = __orchestrion_tracer.StartSpanFromContext(ctx, "operationName",
+			__orchestrion_tracer.Tag("function-name", "trace"),
+			__orchestrion_tracer.Tag("custom_tag", "tag_value"),
+		)
+
+		defer span.Finish()
+	}
+//line /Users/georgiou/go/src/github.com/giou-k/playground/main.go:12
 	fmt.Println("Hello, World!")
 }
 
 //orchestrion:ignore
 func doNotTrace() {
 	fmt.Println("Ignore me, World!")
+}
+
```
As you can see the tool added some code to start and finish a Span, as we would have done manually, following the comment directive that we added in our code. Note that if we had an error block in _`trace()`_ the tool would have handled the tracing there too.

It is interesting to understand vaguely how the orchestration tool works.

Regrading the _`//orchestrion:ignore`_, we can read in the docs[1] "You can use the //orchestrion:ignore directive to prevent orchestrion from performing any modification on the annotated code."
Did some tests on that, and even without annotating this directive, only from the fact the we don't add the _`//dd:span`_ directive, there are not annotations added by the tool. On the other hand the docs explain that some times the instrumentation code might be added to a dependency itself, so in order to avoid that we can use this directive. But either way this is more of a detail. When you see an instrumentation happening and you want to ignore, you know what to use.

Most importantly, we can see that when we add the _`//dd:span`_ directive, we are getting a span initialized and deferred.

### Under the hood

To understand how Orchestrion works its magic, we have to look at how a compiler "sees" our code.

The process starts by breaking our source code into tokens (yes, "tokens" terminology existed before the AI tokens era), which are the smallest possible units of a program, such as keywords, numbers, or function names.
The compiler then uses a parser to organize these tokens into an Abstract Syntax Tree (AST), a data structure where every token becomes a node that defines the program's logic. Because the standard Go AST does not handle comments well, Orchestrion utilizes a Decorated Syntax Tree (DST), which allows it to treat "magic comments" (like //dd:span) as structural labels. By manipulating this tree, Orchestrion inserts new nodes—representing the instrumentation—directly into the code's structure.

To ensure these changes end up in the compiled binary without permanently altering your source files, the tool uses the _`-toolexec`_ build flag to act as a pre-processor. It intercepts the compiler, writes the modified version of your code to a temporary directory, and redirects the Go compiler to build the binary from those temporary files instead of your originals. 

So it edits, adds code in our original code, pass this to the compiler and then builds the binary.✨✨

## Caveats

Everything in life has pros and cons, right? Lets talk about the cons of this architecture.

**Regarding the Orchestrion tool:**
1. The tool is in version 1.X but is is still in active development.
2. One important thing to mention is that as of the time of writing, we cannot input variable values to Orchestrion custom tags. If you are interested in this feature, keep an eye on [this](https://github.com/DataDog/orchestrion/discussions/454) discussion. 
3. It seems that custom attributes are not supported yet, but there are some [default attributes](https://docs.datadoghq.com/standard-attributes/?product=apm).

**Regarding Serverless:**

The caveat with the before mentioned architecture is on metrics. Unfortunately, the _`serverless-init`_ image doesn't support all metric types. It supports only the _`Distribution`_ metric type, which might not be enough for your use case. So in order to support all metric types, you will need to go with each own Datadog Agent(not serverless, **Or** use the [datadog API]https://docs.datadoghq.com/api/latest/metrics/ to send the metrics immediately to Datadog Site, bypassing the Datadog Agent "Collector".


# Sources
[1] https://docs.datadoghq.com/tracing/trace_collection/automatic_instrumentation/dd_libraries/go/?tab=compiletimeinstrumentation

[2] https://www.datadoghq.com/blog/go-instrumentation-orchestrion/
