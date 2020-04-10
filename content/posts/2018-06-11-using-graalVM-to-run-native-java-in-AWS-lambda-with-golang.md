---
title: Using GraalVM to run Native Java in AWS Lambda with Golang
date: '2018-06-11T13:32:49.631Z'
---

> Originally appeared on [Opsgenie Engineering Blog](https://engineering.opsgenie.com/run-native-java-using-graalvm-in-aws-lambda-with-golang-ba86e27930bf)

If you are deploying serverless applications in AWS Lambda and using Java, you are well aware of [cold start problems](https://hackernoon.com/cold-starts-in-aws-lambda-f9e3432adbf0). Cold start happens because of the way the Java Virtual Machine works, it kicks in JIT (Just-in-time), and it needs to “warm-up” like a car from the 80s. AWS Lambda caches containers for you, so when idle, it is paused and brought back online immediately as soon as a request arrives. There is no guarantee that there will be enough containers available. Old ones might be killed (even when they are frequently invoked), so from time to time, your function might be initialized again. Depending on the number of classes loaded and the actual functions used, this process can be frustrating, and can lead to 5–10 seconds of delays, which are called cold starts. There are [many ways to reduce the number of cold starts](https://medium.com/thundra/dealing-with-cold-starts-in-aws-lambda-a5e3aa8f532), such as manual warm up by triggering it with a void input. But in this blog, we will introduce you a probably not practical but a cool use case using GraalVM.

[GraalVM](https://www.graalvm.org/) is an universal virtual machine by Oracle. It allows programs written in Java, JavaScript, R, Python and many more languages to work in a shared runtime, removing barriers and allow zero overhead interoperability. Therefore, you can extend your programs and select the best language for a task. Graal supports both JIT (just-in-time) and AOT (ahead-of-time) compiling. The AOT compiling allows producing native binaries, with the tool native-image. The main advantage is that it eliminates the need for warming up because the code is already optimized. There is an [excellent blog post](https://medium.com/graalvm/instant-netty-startup-using-graalvm-native-image-generation-ed6f14ff7692) about making Netty start under 10 milliseconds and use much less memory. Graal is a unique solution to avoid cold-start problems, but it also has its limitations; it does not support dynamic class loading, and reflection support is also limited. There are other limitations documented which you might need to consider before making the change. There is [another blog post](https://medium.com/graalvm/graalvm-ten-things-12d9111f307d) by GraalVM team, which demonstrates some interesting things you can do with GraalVM with examples.

As you might be aware, AWS Lambda announced Go support in January. Unlike other languages, it does not require a specific function signature to be exported; instead, it communicates with your Go function using net/rpc package. The main advantage is, your Go code is compiled into a binary in your local computer (with GOOS=linux GOARCH=amd64 target), and they can start very fast as they also do not need any dynamic loading and JIT. So, in theory, as long as you implement the RPC format, you can write it in any language. The downside is that net/rpc uses Gob format for exchanging messages, which is not popular in other languages and therefore not implemented. So, we decided to combine Go for RPC, and Java for actual computation and decieded to glue them with Graal.

### The code

Go programs can be compiled as shared libraries, and they can be efficiently loaded with JNI in Java. Luckily, Graal also supports JNI to some extent, enough for our use case. There are no code changes required in Java part, except the Go code cannot call Java with non-primitive types such as string or byte arrays. So, our entry point is from Java, and it calls Go code to set up the RPC handler and communicates with Go channels! The code is simple to understand, but making it work was an enlightening experience which made us more familiar with JNI, Graal, and Cgo.

![Illustration of how the compilation, linking and communication with AWS Lambda works](/img/0__EO090B3qfK__U34J2.jpg)

The native calls we are making from Java to Go is as follows:

*   **void start():** Signals the Go to start the RPC communication with AWS. Each invocation request first hits Go handler, and it notifies the Java using the below functions.
*   **String readRequest():** Asks the Go to read any available input, the Go counterpart blocks on a channel. The Lambda handler writes to a channel, and this function then returns a String as output.
*   **void writeResponse(String input):** Notifies the Go as a response to the previous invocation. Since the Lambda executes only one function at a time per container, the concurrency will not be a problem. The Go routine of the RPC call waits on this response from the channel and returns a response to AWS when Java calls this function.

As you might notice, this pattern is similar to sockets & pipes. At first, we tried using socket communications, although it is less ideal with I/O overhead, it seemed more natural to understand for a blog post. However, AWS shut us down, as you can’t bind to any socket for creating servers. Then, we tried creating Unix pipes, but Java can’t create them (but use them as they are regular files), and the Go function when calls syscall.Mkfifo returns file exists error, although it does not. So, we came up with the below solution, where everything happens in the memory of the single binary and works very fast.

The Go code for handling communication with AWS and Java is as follows:

```golang
func communicateJava(input interface{}) (interface{}, error) {
  inBytes, err := json.Marshal(input)
  inputStr := string(inBytes)
  goRequest <- inputStr
  respStr := <-javaResponse
  var resp interface{}
  err = json.Unmarshal([]byte(respStr), &resp)
}

//export Java_Test_start
func Java_Test_start(env *C.JNIEnv, clazz C.jclass) {
  log.SetPrefix("GO - ")

  go func() {
     lambda.Start(communicateJava)
  }()
}

//export Java_Test_writeResponse
func Java_Test_writeResponse(env *C.JNIEnv, clazz C.jclass, input C.jstring) {
  a := C.convert_to_cstring(env, input)
  b := C.GoString(a)
  javaResponse <- b
}

//export Java_Test_readRequest
func Java_Test_readRequest(env *C.JNIEnv, clazz C.jclass) C.jstring {
  input := <-goRequest
  cstr := C.CString(input)
  cjstring := C.convert_to_jstring(env, cstr)
  return cjstring
}
```

The Java code which is started at the creation of Lambda container loads and calls the Go counterparts using JNI is as follows:


```java
System.loadLibrary("Hello");
start();
while (true) {
  String request = readRequest();
  KMeansResult response = kMeans.calculate(request);
  String response = response.toString();
  writeResponse(response);
  }
}
```

Java counter part of communicating with Golang shared library and doing the actual calculation

To link them both, under Linux, we compile the Go as a shared library. Then, in any machine, we compile Java with regular Java compiler. In our case, we used Maven, to include external dependencies such as Apache Commons Math and org.json to make the task harder, but more meaningful as using plain Java is not interesting at all. We package it as a fat-jar and supply it to native-image tool included with the GraalVM distribution. Example build output is as follows:

```sh
root@03da97ad05ed:/code# go build \
 -buildmode=c-shared \
 -o libHello.so \
 src/main/go/lambda.go

root@03da97ad05ed:/code# $GRAALVM_HOME/bin/native-image \
 MyEntrypointJava \
 -cp "target/graalvm-java-go-awslambda-1.0-SNAPSHOT.jar" \
 -H:+JNI -H:+ReflectionEnabled \
 -H:+ReportUnsupportedElementsAtRuntime \
 -Djava.library.path=$(pwd) - no-server
 
classlist: 4,593.14 ms
(cap): 1,545.73 ms
setup: 3,208.61 ms
(typeflow): 7,488.44 ms
(objects): 3,405.12 ms
(features): 76.77 ms
analysis: 11,184.50 ms
universe: 471.31 ms
(parse): 1,677.39 ms
(inline): 1,150.32 ms
(compile): 9,294.53 ms
compile: 12,667.04 ms
image: 1,297.93 ms
write: 354.55 ms
[total]: 33,868.05 ms

root@03da97ad05ed:/code# zip lambda.zip libHello.so myentrypointjava
adding: libHello.so (deflated 68%)
adding: myentrypointjava (deflated 67%)
```

We have used a custom Docker image with OpenJDK 8, Go 1.10.2 and GraalVM 1.0 RC1. You also need zlib development files and GCC. As you can see, the lambda.go file is compiled as c-shared with Go compiler. After that, the bundled Fat JAR file is passed to native-image, which takes a considerable amount of time (33.8 seconds) to determine the used classes, analyze the flow and optimize & inline as a regular compiler does. In the end, it generates a single executable, only depending on system libraries such as libc, libz, libpthread and libcrypt. You still need to supply the shared library produced by Go, as it is loaded dynamically (with System.loadLibrary() call), not linked statically.

After uploading the package to AWS Lambda with even a relatively small memory configuration, it generally works under 10ms and sometimes even sub-milliseconds. You can see there are no more cold starts and the function is available immediately upon container initialization.

An example invocation log from Cloudwatch is as follows:

![Yes, A sub-millisecond cold start (first invocation) with only consuming 38 MB with external dependencies, even in the slowest Lambda settings.](/img/0__O7NTtO8lqlYkcZs9.jpg)

The code in the example is just generating small random vectors and applying k-means clustering algorithm, to make sure we are doing the somewhat meaningful computation in Java part (with the help of Apache Commons Math) and we can adequately benchmark it and compare it with regular Lambda. [The code is uploaded to Github,](https://gist.github.com/mustafaakin/71e4494508311f7764e59b16e9dd3b8e) if you are interested and want to try on your own.

### Testing & Benchmarks

What good of AOT-compiled native binaries versus JIT Java classes if you do not indent to compare the performance? We also have the regular Java version of the above function as a Java Lambda. The function is parameterized, so that we can generate enough computational power need to amortize the external costs such as I/O, similar to one benchmark in one of our [old blog posts](https://engineering.opsgenie.com/how-does-proportional-cpu-allocation-work-with-aws-lambda-41cd44da3cac) trying to explain AWS CPU performance for various memory settings. Always remember that, for nice benchmarks, you need repetitions and enough samples so that you can remove outliers and see the actual patterns. Also, if you are comparing raw CPU performance or multi-threads, you need to make sure that the overhead of external factors such as initialization and I/O should be minimal and your functions should have a consistent runtime for same input sizes. Otherwise, you will be benchmarking external factors which will benefit no one and confuse many people.

In this benchmark, we have used Apache Commons Math to generate a uniformly distributed N number of d-dimensional points and apply k-means clustering algorithm, which makes no sense but it is a CPU and Memory burden and does not depend on any I/O, thus a good candidate for benchmarking in our case. The wiring of the Go code and Java is established as above, and the inputs are passed as strings. As GraalVM does not support reflection, the parsing of the input in Java is done via org.json package; we construct the POJOs by hand, which would not be desirable for a massive project where you have much input and outputs already defined and uses Jackson. We have compared both Java and Go+Graal version with various memory settings with a large enough input to make overheads negligible. Although the Graal version has no cold starts, it turns out it is slower and consumes more memory.

![Benchmark results for K-Means calculation with Plain Java and GraalVM+Golang in AWS Lambda with various memory settings](/img/1__g9SNrYT2GSlC8ba__WroaVA.png)

Graal + Go version was always slower, but the runtime duration was more consistent. It suffered no cold start, but it also consumed more memory. Plain Java version used 50–60 MB of memory but the Graal version used additional 110–290 MB of memory.

Naturally, we blamed Go binding and AWS Lambda, and run the tests locally with plain Java and native-image with no Go code. It turns out it is also slower and used more memory in our local machines as well. AoT compiler of GraalVM might not be suitable for some use cases. Therefore, we might need to look at the generated assembly code to understand what it has compiled to and why it is slower. But since the library we used is complicated, and understanding machine code and comparing it with JVM bytecode is no easy task, so we tried to benchmark a simple job, which calculates sin(x) continuously. To make sure the compiler did not optimize out sin(x), we updated x continually with the result of sin(x) and saw the results are same for both Java and Graal. In this simple task, native-image created by Graal was 2 times faster, for 10 million sin(x) calculations.

As most people use Lambda to communicate with external systems, such as uploading a file, processing an authentication flow, or doing administrative tasks, they might utilize the CPU differently and therefore have different performance output depending on the number of IO performed. We intended to do a test consisting of I/O, such as HTTP requests to AWS Services, but GraalVM [does not support SSL yet](https://github.com/oracle/graal/issues/390). If you are making an SSL connection for the first time, it takes considerable time because it loads many classes and JIT is not kicked in yet. So it would be good to see how the SSL can be improved with GraalVM.

As with all fancy benchmarks, you should question their validity and applicability carefully. Maybe your function will be three times faster, or it will not compile at all with Graal. Critical thinking and avoiding hype train, or boarding that train with proper caution is never a bad idea. Although the GraalVM has different performance for various workloads and supports a subset of JVM features, it eliminates the cold start problem, and the native binaries start- immediately. This fast start time enables your Java programs to be used as CLI programs as well. We have some lambda functions that include some Hadoop libraries for ETL tasks and they have around 5–10 seconds cold start time in low memory settings, and we sure can make use of Graal for them in the future.