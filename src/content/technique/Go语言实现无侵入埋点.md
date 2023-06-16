---
title: Go语言实现无侵入埋点
publishDate: 2023-04-13 09:41:00
img: /assets/technique/Go语言实现无侵入埋点/封面.jpg
img_alt: 
description: |
  Java语言有javaagent这样的外部动态插件实现探针技术，可以做到实现可观测性数据采集功能而不修改业务代码，被称作无侵入埋点方案。Golang语言则通过手动埋点方式接入可观测性，本文旨在探究Golang实现无侵入埋点的可能性。
tags:
  - 可观测性
  - 探针
---

　　原文为全英文论文，我先自己阅读一遍，再根据自己的理解翻译解读。https://docs.google.com/document/d/1jMzCrr1vIEWvv-PJG9XiqJ6BGIB1wq4tvwyP4XL9lOI/mobilebasic

### Background

　　In the current Golang version of the agent, we can integrate our projects with go2sky and implement distributed tracing by manually embedding points into third-party frameworks. However, there are a few drawbacks to this approach:

　　　　1.Projects must have manual instrumentation. If no points are embedded in the code(such as ctx.Context), it is impossible to enable the monitoring and propagation of the tracing context between services.

　　　　2.It cannot be intercepted if the framework does not actively provide an API (e.g. a middleware or an inspector). Sometimes wrapping is complicated if no interface is provided or even with an interface you can’t access internal details.

　　Considering the two issues mentioned above, I have been pondering whether there is a better way to implement an automatic instrumentation agent similar to Java Agent for Golang.

　　<font color=bule>在当前的Golang版本，要实现业务系统接入可观测性，我们可以通过go2sky技术（Golang用skywalking实现全链路追踪）并且手动埋点来实现。但是这种方式有下面两个缺点：

　　　　1.项目中必须提供上下文传播工具context.Context()，如果不传播上下文就做不了应用监控。

　　　　2.如果业务系统使用的框架不对外暴露特定的API，就不能被拦截计算出监控信息，或者需要复杂的额外包装才能实现。

　　考虑到这两个问题，我在思考Golang有没有办法像Java的javaagent一样不用手动埋点也能实现应用监测效果。
　　</font>

### Technology Select

　　Through my technical research, I have found that there are currently two technologies that can implement a similar functionality: eBPF or hybrid compilation.

　　<font color=bule>通过我的技术预研，我发现目前有两种技术可以实现期望的功能：eBPF或混合编译。
　　</font>

#### eBPF（extended Berkeley Packet Filter）
![eBPF](/assets/technique/Go语言实现无侵入埋点/eBPF.png)

　　eBPF is a popular technology in recent years. In Apache SkyWalking, we have our own eBPF Agent -- Apache SkyWalking Rover, which allows non-intrusive monitoring of applications. In simple terms, with eBPF, the program can be monitored without any modifications; all that is needed is to deploy the eBPF Agent on the machine where the service is running.

　　For Golang automatic instrumentation agents, eBPF can monitor the execution process of each method in a Golang program. All that is required is to intercept, acquire, and expand the data during method execution.

　　Ideally, eBPF seems like a great solution, but upon further research, I found that it has a few issues, also, I found a similar answer in an issue on datadog's GitHub repository:
https://github.com/DataDog/dd-trace-go/issues/1273#issuecomment-1119580249

　　　　1. Performance problems caused by context switching: When eBPF executes a user-mode program, it first needs to switch the current user mode to kernel mode execution, and then switch back to user mode after kernel mode execution is completed.

　　　　2. Golang version compatibility and data manipulation: Different Golang versions may have completely different internal data structures, making it difficult to discern. Furthermore, when reading or writing data, using eBPF to operate Golang's memory space may be extremely challenging. Even if we can create a separate Tracing Context within the current service, the inability to pass the Tracing Context prevents it from combining with other services, which goes against the original purpose of Tracing Context.

　　Due to the two key issues mentioned above, we have decided to abandon this approach.

　　<font color=bule>eBPF（extended Berkeley Packet Filter）是一种开源技术，它提供了一种在操作系统内核中执行自定义代码的机制。eBPF最初是为网络数据包过滤而设计的，但现在已经扩展到其他领域，如安全监控、性能分析和系统跟踪等。

　　BPF的核心思想是在内核中嵌入一种虚拟机，它可以执行安全且高效的用户定义代码片段，这些代码片段通常被称为eBPF程序。eBPF程序可以在数据包到达网络协议栈的各个阶段执行，从而实现实时的数据包过滤和处理。与传统的基于规则的数据包过滤器相比，eBPF提供了更高的灵活性和可编程性。

　　eBPF程序使用一种基于RISC（精简指令集计算机）的指令集来定义其逻辑。这些指令被编译成一种中间表示（IR），然后由内核的JIT（即时编译）编译器转换成本机机器码，以便在内核中执行。

　　eBPF是近年来流行的一种技术。在Apache SkyWalking中，我们有自己的eBPF代理——Apache SkyWalking Rover，它允许对应用程序进行非侵入式监视。

　　eBPF可以监视Golang程序中每个方法的执行过程。它做的就是在方法执行期间拦截方法，获取数据。

　　简单来说，使用eBPF，无需任何修改即可对程序进行监控，要做的就是在运行服务的机器上部署eBPF代理，这点和javaagent很像了。

　　eBPF看起来是一个很优秀的解决方案，但经过我的进一步研究，发现了它存在一些关键性问题。https://github.com/DataDog/dd-trace-go/issues/1273#issuecomment-1119580249

　　　　1. 上下文切换导致的性能问题：当eBPF执行用户模式的程序时，需要先把用户模式切换到内核模式运行，然后在内核模式运行结束后再切换回用户模式。

　　　　2. Golang版本兼容性问题：不同的Golang版本编写的服务可能具有完全不同的数据结构底层实现。在读取或写入数据时，使用eBPF操作Golang的内存空间可能极具挑战性。即使我们在当前服务中创建单独的链路上下文，也做不到跨服务传播，这违背了链路上下文的设立初衷。

　　由于这2个关键性问题，我决定放弃使用eBPF技术。
　　</font>

#### Hybrid compilation

![Hybrid_compilation](/assets/technique/Go语言实现无侵入埋点/Hybrid_compilation.png)

　　Hybrid compilation takes advantage of Golang's “-toolexec” parameter during program building, dynamically executing compilation operations through the provided program, and ultimately generating the final binary file. With this feature, we can intercept any required Go file and make modifications during method execution, thus implementing dynamic interception. I found an article that provides a more detailed explanation of the implementation principles of dynamic interception: https://blog.sqreen.com/dynamic-instrumentation-go/

　　Using hybrid compilation, it typically offers more efficient performance, as it is mixed with the program and compiled into a binary file. Moreover, it can also intercept any method.

　　The hybrid compilation also has a few drawbacks:

　　　　1. Unable to import libraries dynamically: Since it is based on intercepting Golang compilation to achieve enhancement, its extensibility is limited to existing libraries. This can lead to the inability to actively import some foundational libraries, such as go2sky or gRPC framework, which need to be actively introduced within the application to trigger compilation.

　　　　2. toolexec only supports a single program: It is not possible to import multiple toolexec programs.

　　Based on the two approaches mentioned above, we found that hybrid compilation is more suitable for our needs. It can embed custom programs into any code within the service while offering better performance.

　　To verify the feasibility of hybrid compilation, I have specifically written a demo program to validate its basic functionalities, including method interception, cross-goroutine data transmission, enhanced entities, and more. You can access the project and try it out: https://github.com/mrproliu/go-agent-instrumentation

　　<font color=bule>合编译利用Golang程序构建期间的“-toolexec”参数（我之前没玩过这个参数，有谁用过吗），通过所提供的程序动态执行编译操作，并最终生成最终的二进制文件。有了这个特性，我们可以拦截任何所需的Go文件，并在方法执行期间进行修改，从而实现动态拦截。我找到了一篇文章，它对动态拦截的实现原理提供了更详细的解释:https://blog.sqreen.com/dynamic-instrumentation-go/

　　使用混合编译，它通常提供更高效的性能，因为它与程序混合并编译成二进制文件。同时混合编译也存在一些缺点：

　　　　1. 无法动态导入依赖库：由于它基于拦截Golang编译过程来实现增强，导致无法动态导入一些基本库，比如go2sky或gRPC框架。这些库需要在应用程序的gomod中主动引入以触发编译。(为什么这是缺点？不理解)

　　　　2. 程序编译时只支持单个toolexec：不能导入多个toolexec程序。

　　为了验证混合编译的可行性，我专门编写了一个演示程序来验证其基本功能，包括方法拦截、跨协程传输数据、增强的实体等。https://github.com/mrproliu/go-agent-instrumentation
　　</font>

### Design Goal

　　Based on the two approaches mentioned above, we found that hybrid compilation is more suitable for our needs. It can embed custom programs into any code within the service while offering better performance.Based on the hybrid compilation implementation approach, we hope to achieve the following results.

　　<font color=bule>基于上述两种方法的比较，我发现混合编译方法更能满足我们的需要。它可以将自定义程序方法嵌入到服务中的任何代码中，同时提供更好的性能。基于混合编译方法，我们希望实现以下结果。
　　</font>

#### For User

　　Users only need to import the agent module in their program.

```
import _ "github.com/apache/skywalking-go"
```

　　And add it to the compilation parameters using “-toolexec” when building the project. The configuration of the Agent can be passed in via configuration files.

```
$ go install github.com/xxx/go-agent-instrument
$ export GO_AGENT_CONFIG=/path/to/config.yaml
$ go build -toolexec /path/to/go-agent-instrument ./
```

　　During the execution of the user's program, there is no need to worry about whether the context is passed or not. The system will automatically associate the tracing context with the current goroutine or any goroutine generated by the current goroutine.

　　<font color=bule>对于用户来说，只需要在项目中添加依赖，并且在编译的时候添加参数 -toolexec 就能实现自动埋点。用户不需要在代码中传递上下文，用这种方式编译出的程序会自动跟踪程序当前运行的goroutine并自动生成、传播链路上下文。
　　</font>

#### For Framework Plugin Developer

　　For developers who want to create plugins, we hope the process is as simple as possible, similar to the current Java Agent. You need to follow these steps.

　　<font color=bule>对于插件开发者来说，我希望开发插件的过程尽可能简单，就像开发java-agent一样。你需要遵循一下6个步骤。
　　</font>

##### Import agent core

　　When developing an agent, import the core library.

　　<font color=bule>在开发go-agent插件的时候，需要引入核心库。这里的核心库应该是指Golang的内置库built-in library。[参考网页1](https://www.golang-book.com/books/intro/13)　　[参考网页2](https://tip.golang.org/doc/go1.20#library)
　　</font>

##### Plugin module import

　　When writing plugins, only the agent core library and the required enhancing libraries for the plugin can be imported. The only thing to be cautious about is not importing other libraries, as hybrid compilation cannot import additional libraries, and doing so may cause errors during compilation.

　　If the module you currently depend on contains the necessary sub-modules you need to use, you can use them directly.

　　But If you want to introduce a new library, it can only be imported and used through the Agent Core in a unified manner. This approach ensures consistency and compatibility across the various components of the agent.

　　<font color=bule>在开发go-agent插件时，只能导入内部库，不能导入外部库。因为导入外部库会让混合编译在编译期间失败。如果导入依赖是为了使用它的子模块，那么应该直接导入那些子模块而不是导入一个大包。在添加依赖时，要用agent core来统一管理以确保各组件之间的一致性、兼容性。
　　</font>

##### Define Instrument Info
　　A plugin can define Instrumentation information, which includes the following key information:

　　　　1. Package Name: The package to be enhanced, for example, "github.com/gin-gonic/gin".

　　　　2. Version Checker: Indicates which versions of the current framework are supported.

　　　　3. File System: The directory export where the current framework is located, is used for copying and modifying the interceptor files when the compilation is executed. You can directly use “go:embed *”.

　　　　4. Interception Points: Specify which methods can be intercepted, which classes can be enhanced, etc. More details will be introduced in the next section.

　　<font color=bule>插件中包含的信息：

　　　　1. 软件包名：插件是哪个包。

　　　　2. 版本检查：说明兼容的Golang版本范围。

　　　　3. 文件系统：说明用于复制和修改编译文件的拦截器使用的文件夹。

　　　　4. 拦截点：说明哪些函数哪些类可以被插入链路追踪。

　　</font>


##### Define Instrument Points

　　An Instrumentation can define multiple interception points, each containing the following information:

　　　　1. Relative package path: In the Instrumentation, we define the package path, but here we have the relative path within the current package. For example, if the path of the framework files you intercept is "github.com/gin-gonic/gin/render", then its path in the Instrumentation is "github.com/gin-gonic/gin", and the relative path for the current interception point is "render". Since the “go build” compiles at each package level during compilation, we still need to comply with this rule.

　　　　2. File name: Specifies in which file the current interception point needs to be processed. Using this parameter can reduce the traversal of all files, speeding up the program's compilation speed.

　　　　3. Interception configuration: Supports method interception and enhancement of specified struct structures.

　　<font color=bule>一个插件可以有多个函数拦截点，并且每个函数拦截点包含以下信息：

　　　　1. 相对路径：例如我们要混合编译的代码是github.com/gin-gonic/gin/render，那么它在插件中的路径是github.com/gin-gonic/gin，函数拦截点的相对路径是render。

　　　　2. 文件名：指定要拦截的函数拦截点在哪个文件中，这样可以避免遍历整个目录下的全部文件，加快编译速度。

　　　　3. 函数拦截器的可选择配置项：允许用户选择一些插入规则。

　　</font>

##### Intercept method

　　Provide interception for static methods (e.g., “func NewXXX()”) or class methods (e.g., “func (i *Instance) InstanceMethod()”). At the same time, it supports the validation of parameter information, and the method can be intercepted only when the validation rules are met.

　　The following two fields of information are required:

　　　　1. Validation: Since method interception is based on AST syntax validation, I will provide a dedicated API to help users complete the validation work more easily, reducing the cost of use. This is as simple as using the ElementMatcher in the ByteBuddy framework in JavaAgent.

　　　　2. Interceptor name: You also need to specify which interceptor to use to handle the method execution flow when the specified validation rules are met.

　　Interceptors need to implement the following two methods. Please refer to the code:

　　<font color=bule>提供静态方法和类方法的拦截方式，同时支持校验配置，只有满足校验规则的函数才会被拦截。有两个东西是必须的。

　　　　1. 检验逻辑：函数的拦截是基于AST（abstract syntax tree [抽象语法树](https://juejin.cn/post/7030457038840791053)）语法验证做的，文章作者会提供一个API来帮插件开发者做检验逻辑，就像java-agent中的bytebuddy框架（字节码修改技术）使用element-matcher（元素匹配器）一样简单。

　　　　2. 拦截器名称：可以设置多个拦截器，每个拦截器有自己的名称和插入逻辑，对满足不同校验逻辑的函数选择对应名称的拦截器来处理插入逻辑。

　　</font>

```
// Invocation define the method invoke context
type Invocation struct {
        CallerInstance interface{}   // Caller from, nil if static method
        Args           []interface{} // Function arguments
        Continue       bool          // Is continue to invoke method
        Return         []interface{} // The method return values when continue to invoke
        Context        interface{}   // Propagating context between BeforeInvoke or AfterInvoke
}

type Interceptor interface {
        // BeforeInvoke method before invoking
        BeforeInvoke(invocation *Invocation) error
        // AfterInvoke after invoke finished
        AfterInvoke(invocation *Invocation, result ...interface{}) error
}
```

##### Enhance Instance

　　The purpose of enhancing instance is to embed custom fields in an object to assist during plugin usage. For example, in Java Agent, there is an “EnhancedInstance”.

　　Similar to method interception, you still need to define a validation configuration method. This informs which type needs to be enhanced.

　　We will define this interface in the plugin core module. Please refer to the following code:

　　<font color=bule>改造实例的目的是在目标类中嵌入自定义字段，与函数拦截器类似，改造实例也需要定义校验逻辑告知需要改造的类。在插件的核心模块中定义了这个接口，请遵守。
　　</font>

```
type EnhancedInstance interface {
  GetSkyWalkingDynamicField() interface{}
  SetSkyWalkingDynamicField(interface{})
}
// cast to the enhanced instance
instance := invocation.CallerInstance.(core.EnhancedInstance)
// setting the customized value
instance.SetSkyWalkingDynamicField("test")
// get the customized value from instance
val := instance.GetSkyWalkingDynamicField()
```

#### Thread Local Usage

　　In Golang, we can also implement a function similar to "Context#getRuntimeContext" in Java, and it will be passed between Goroutines, as shown in the following code:

　　<font color=bule>在Golang里面我们也可以做一个运行时上下文传递的函数在协程中传递，就像Java一样。
　　</font>

```
core.GetRuntimeContext().Put("test", "value1")
go func() {
        core.GetRuntimeContext().Get("test") // the result is "value1"
}()
```

#### Tracing Span Operation API

　　In the original go2sky, to operate a span, you need to know the tracer and context objects. In the automatic enhancement plugin, you don't need to pass in this information.

　　Therefore, it's necessary to introduce new APIs in the Agent Core module to operate the Span, making it the same as Java's ContextManager. For example, when creating an Entry, you only need to specify the operation name (Endpoint Name) and Injector (Carrier) by default.

　　<font color=bule>在已有的go2sky，从Golang服务发送链路数据到skywalking的工具库中，用户需要熟悉手动埋点的流程。但是在无侵入埋点方式中，用户不需要了解这些细节，只需要在程序初始化的时候填写一些配置项例如上报地址endpoint、注入器carrier即可。
　　</font>

#### Import Plugin

　　Once we have completed the plugin development, we need to actively introduce the plugin into the code for use. The reason is that there is no dynamic loading mechanism like Java's SPI in Golang. You only need to import it in one Golang file.

　　<font color=bule>插件开发完成之后，需要在待代码的go mod和something.go中添加依赖，因为Golang没有Java SPI动态加载机制。（Service Provider Interface [JDK内置的一种服务提供发现机制](https://zhuanlan.zhihu.com/p/84337883)）
　　</font>

#### Plugin workflow

　　The following sequence diagram explains the execution process of how the enhanced program integrates the plugin with the original file. If you want to know more about the detailed implementation principles, please refer to the next section.

　　<font color=bule>下面这个时序图解释了go-agent无侵入埋点插件的工作原理
　　</font>

![OTA_时序图](/assets/technique/Go语言实现无侵入埋点/OTA_时序图.png)

### Implementation mechanism

![Implementation_mechanism](/assets/technique/Go语言实现无侵入埋点/Implementation_mechanism.png)

　　When the toolexec program is specified during the "go build", the Go program will collect the files from each package-level directory (including Go's own package directories, such as runtime, text, etc.) and send the "compile" or "asm" command to the toolexec program. The toolexec program actually executes the method and generates the target file. You can think of the toolexec program as a proxy. It is important to note here that "go build" will only send Go files of the same level; if there are different level directories, they will be called multiple times.

　　So when the Go agent instrumentation program executes, it can obtain the file paths in each package, allowing it to access all the Go files required for the target program's compilation. At this point, the enhancement program can use AST technology to parse and modify the file content. After the modifications or additions are completed, the parameters required for compilation can be modified, and the compilation command can be executed eventually. This is the principle of hybrid compilation.

　　<font color=bule>当在go build编译时在命令行使用参数toolexec指定go-agent插件时，Golang会从所有的package级别收集文件。遇到compile或者asm命令的时候把package内全部文件发给toolexec指定的go-agent插件生成目标编译文件。因此可以把toolexec指定的go-agent插件看做是一个编译代理。所以说，混合编译的原理就是go-agent运行时访问package中的所有待编译文件，增强程序使用AST技术解析文件内容并且添加埋点方法。
　　</font>


#### Method interception

　　When the enhancement program detects that the go file of the current package contains the specified method, the program will make the following changes.

　　<font color=bule>当增强程序检测到当前package包含符合条件的函数时，程序就会做出一下更改。
　　</font>

##### Insert code before method execution

　　This is used to intercept the execution flow before and after the method is executed.

```
func (x *Receiver) TestMethod(arg1 int) (result error) {
        if skip_result1, _sw_invocation, _sw_keep := _skywalking_enhance_receiver_TestMethod(&x, &arg1); !_sw_keep {
                return skip_result1
        } else {
                defer _skywalking_enhance_receiver_TestMethod_ret(_sw_invocation, &result)
        }

        // real method invoke
}
```

　　In the code above, we can see that we want to intercept the TestMethod of the Receiver, and the aforementioned code is inserted before the actual code execution of the method.

　　　　1. _skywalking_enhance_receiver_TestMethod: This function is executed before the method interception, and the Receiver and the method's parameters are passed into it. The function returns the current invocation object and whether to continue execution. If further execution is not needed, a custom result is returned. Otherwise, the subsequent code is executed.

　　　　2. _skywalking_enhance_receiver_TestMethod_ret: This function is executed after the intercepted method has been completed, and the return value of the current method is passed into it. It is worth noting that if the method to be intercepted does not have parameter names, the enhancement program will automatically set names for them.

　　<font color=bule>这段代码展示了在方法执行之前和方法执行之后的拦截流程。通过读这段代码，我们可以看出，想要拦截Receiver类的TestMethod方法。如果还没有被拦截过，那么返回改造过的方法，如果已经拦截过，defer在最后执行一些事项。

　　　　1. _skywalking_enhance_receiver_TestMethod:此函数在方法拦截之前执行，并将Receiver和方法的参数传递给它。函数返回当前调用对象以及是否继续执行。如果不需要进一步执行，则返回自定义结果。否则，执行后续代码。

　　　　2. _skywalking_enhance_receiver_TestMethod_ret:该函数在拦截方法完成后执行，并将当前方法的返回值传递给它。值得注意的是，如果要拦截的方法没有参数名，增强程序会自动为其设置参数名。
　　</font>

##### Copy interceptor code to the package

　　The enhancement program copies the files from the plugin into the framework's code package, allowing the interceptor to be combined with method execution.

　　It's important to note that the copying process strictly follows the hierarchy of the method interception. If it's in the root directory, only files in the root directory will be copied. If it's in a certain level under a package, the files in that level will be copied. The reason for this approach is due to the principle that only files of the same package will be compiled during mixed compilation.

　　<font color=bule>增强程序把插件中的文件复制到项目的package中，从而使得拦截器与方法相结合。值得注意的是，复制过程严格遵循方法拦截的层次结构。如果它在根目录中，则只复制根目录中的文件。如果它位于package下的某个级别，则该级别中的文件将被复制。采用这种方法的原因是混合编译期间只编译同一package的文件这一原则。
　　</font>

##### Create bridge methods

　　The bridge method is created to integrate the logic before and after method execution with the code in the interceptor.

```
// the interceptor only created once
var testMethodIntercepterInstance = &ServerHTTPInterceptor{}
func _skywalking_enhance_receiver_TestMethod(recv_0 **Receiver, param_0 *int) (result1 int, inv *Invocation, keep bool) {
        // fallback for the plugin execute failure
        defer func() {
                if r := recover(); r != nil {
                        // log the error
                }
        }

        // create a new invocation instance
        invocation := &Invocation{}
        // for caller if exist
        invocation.CallerInstance = *recv_0        
        // for parameters
        invocation.Args = make([]interface{}, 1)
        invocation.Args[0] = *param_0

        // before method invoke
        if err := testMethodIntercepterInstance.BeforeInvoke(invocation); err != nil {
                // if invoke have return error, then keep the real method running
                return 0, invocation, true
        }
        // is skip method invoke, then return the customized result
        if invocation.Continue {
                return invocation.result[0], invocation, false
        }
        // otherwide, keep the method running
        return 0, invocation, true
}

func _skywalking_enhance_receiver_TestMethod_ret(invocation *Invocation, result_1 *error) {
        // fallback for the plugin execute failure
        defer func() {
                if r := recover(); r != nil {
                        // log the error
                }
        }
        testMethodIntercepterInstance.AfterInvoke(invocation, *result_1)
}
```

　　This part of the code serves to combine the code from the first and second sections together.

　　　　1. Interceptor instance: The instance will be constructed within the current package, rather than being created each time it is executed, in order to reduce memory overhead.

　　　　2. The first method: In the method interceptor, the parameters and Receiver information will be constructed into the Invocation object, and the "BeforeInvoke" method of the interceptor will be executed. If it doesn't need to continue, it will return a custom return value, otherwise, it will continue running. If any problems occur during code execution (recover or error return value from BeforeInvoke), the program will continue executing as well.

　　　　3. The second method: This will be executed after the intercepted method is completed, with the return value of the intercepted method also being passed in. Similarly, there is a "recover" method to prevent issues caused by the plugin code.

　　<font color=bule>创建桥接方法是为了将方法执行前后的逻辑与拦截器中的代码集成在一起。

　　　　1. 拦截器实例:实例将在当前package中构造，而不是在每次执行时创建，以减少内存开销。

　　　　2. 第一个方法:在方法拦截器中，将参数和Receiver信息构造到调用对象中，并执行拦截器的BeforeInvoke方法。如果它不需要继续，它将返回一个自定义返回值，否则，它将继续运行。如果在代码执行期间出现任何问题(从BeforeInvoke中恢复或返回错误值)，程序也将recover继续执行。

　　　　3. 第二个方法:它将在被截获的方法完成后执行，同时也传入被截获的方法的返回值。类似地，有一个“恢复”方法来防止由插件代码引起的问题。

　　</font>

##### Result

　　Upon completion of these three steps, the following modifications can be made to the files in the intercepted package:

　　　　1. Modify the file containing the enhanced method: Create code before the intercepted method is executed.

　　　　2. Add interceptor file: Copy the file where the interceptor is located.

　　　　3. Add bridge file: Write a separate file for the bridging part.

　　<font color=bule>完成这三个步骤后，可以对拦截的package中的文件进行以下修改:

　　　　1. 修改含有增强方法的文件:在执行被截获的方法之前创建代码。

　　　　2. 添加拦截器文件:复制拦截器所在的文件。

　　　　3. 添加桥接文件:为桥接部分编写一个单独的文件。

　　</font>

##### Code Location Change Issue

　　Based on the first step, if we add code to the intercepted method, it may cause the code line position to be shifted when a problem occurs in the framework. The solution I came up with is to create a new method and name it as the real method, and rename the intercepted method to a temporary name. For example, the following code:

```
// before
func (x *Receiver) TestMethod(arg1 int) (result error) {
        // real method invoke
}

// after
func (x *Receiver) skywalking_enhanced_TestMethod(arg1 int) (result error) {
        // real method invoke
}
func (x *Receiver) TestMethod(arg1 int) (result error) {
        if skip_result1, _sw_invocation, _sw_keep := _skywalking_enhance_receiver_TestMethod(&x, &arg1); !_sw_keep {
                return skip_result1
        } else {
                defer _skywalking_enhance_receiver_TestMethod_ret(_sw_invocation, &result)
        }
        return x.skywalking_enhanced_TestMethod(arg1)
}
```

　　Using this approach, the only downside I can think of is that when a problem occurs, the method name in the stack trace might not be correct, but the code line would be completely accurate.

　　<font color=bule>如果我们将代码添加到被截获的方法中，当出现问题时，可能会导致代码行位置发生移位，导致报错代码定位不准。我想到的解决方案是创建一个新方法并将其命名为真正的方法，并将被截获的方法重命名为临时名称。示例代码如上。使用这种方法，我能想到的唯一缺点是，当出现问题时，堆栈跟踪中的方法名称可能不正确，但代码行定位将完全准确。
　　</font>

#### Instance Enhance

　　For instance enhancement, the process is relatively straightforward. We just need to embed a new field of any type (interface{}) into the current struct. Additionally, in the Bridge's Go file, we need to add the implementation for the EnhancedInstance methods (mentioned in the "For framework plugin developer" section) specific to that struct. The following code demonstrates this:

```
type Test struct {
        // existing fields
        skywalking_enhance_field interface{}        // adding the field into the structure
}

// adding these two methods into bridge go file
func (receiver *Test) SetSkyWalkingField(val interface{}) {
        receiver.skywalking_enhance_field = val
}

func (receiver *Test) GetSkyWalkingField() interface{} {
        return receiver.skywalking_enhance_field
}
```

#### Goroutine with Tracing Context

　　In Golang, there is no functionality similar to Java's ThreadLocal. However, we can achieve this by adding properties to the "struct g". The "struct g" represents a goroutine object in Golang, and in the "runtime" package, you can obtain the currently running goroutine object through the "getg() *g" function.

##### Adding fields in the struct g

```
type struct g {
        // real fields
        // thread-local fields
        skywalking_enhance_obj interface{}
}

// create a new file in "runtime" package for export the thread-local value
import (
 _ "unsafe"
)

//go:linkname _skywalking_tls_get _skywalking_tls_get
var _skywalking_tls_get = _skywalking_tls_get_impl

//go:linkname _skywalking_tls_set _skywalking_tls_set
var _skywalking_tls_set = _skywalking_tls_set_impl

//go:nosplit
func _skywalking_tls_get_impl() interface{} {
 return getg().m.curg.skywalking_enhance_obj
}

//go:nosplit
func _skywalking_tls_set_impl(v interface{}) {
 getg().m.curg.skywalking_enhance_obj = v
}
```

　　In the code snippet above, we added a new attribute "skywalking_enhance_obj" to the g structure, and in the runtime package, we added a new file to export this field. The reason is that the "getg()" method is not exposed externally.

　　Next, we need to import the exported methods in the plugin core and go2sky (which will be introduced in more detail in the next section).

```
import _ "unsafe"

var (
        GetGLS = func() interface{} { return nil }
        SetGLS = func(interface{}) {}
)

//go:linkname _skywalking_tls_get _skywalking_tls_get
var _skywalking_tls_get func() interface{}

//go:linkname _skywalking_tls_set _skywalking_tls_set
var _skywalking_tls_set func(interface{})

func init() {
        if _skywalking_tls_get != nil && _skywalking_tls_set != nil {
                GetGLS = _skywalking_tls_get
                SetGLS = _skywalking_tls_set
        }
}
```

　　Now, in the plugin, you can obtain the data information of the current goroutine without a context object.

##### Obtain data in different goroutine

　　In Golang, we can start new goroutines to accomplish tasks at any time. This leads to a problem: there can be many goroutines, but we only set custom data in the first one. When an RPC request is executed in a newly created goroutine, we are unable to access the data from the first goroutine, as demonstrated in the following code:

```
// goroutine A

core.SetGLS("key", "test data")
go func() {

      // goroutine B
        core.GetGLS("key")        // return nil
}()
```

　　After learning the Golang Runtime code, we can see that when creating a new goroutine, the "runtime.newproc1" method is actually called. By analyzing the parameters of the "runtime.newproc1" method, we can see that the second parameter, callergp, indicates who created the goroutine. The method signature is shown below:

```
// Create a new g in state _Grunnable, starting at fn. callerpc is the
// address of the go statement that created this. The caller is responsible
// for adding the new g to the scheduler.
func newproc1(fn *funcval, callergp *g, callerpc uintptr) *g {}
```

　　In the example code above, when executed, the second parameter represents "goroutine A", and the returned instance is "goroutine B".

　　Based on the fact that we have already added custom attributes to struct G in the previous section, we can modify the "runtime.newproc1" method. When it returns, we can set the custom attributes in the returned instance to complete the task.

```
func newproc1(fn *funcval, callergp *g, callerpc uintptr) (result *g) {
        defer func() {


                result.skywalking_enhance_obj = &tracingContext{span: snapshot(span), }
        }()

      // code
}
```

##### Span Operation with GLS(Goroutine Thread Local)

　　When using methods like "core.CreateLocalSpan(opName string)", we need to obtain the tracing context and global tracer information from GLS to create the current span information.

　　Based on the current implementation in "go2sky", we can put the "segmentSpan" into GLS. Whenever a new span is needed, simply treat the Span from GLS as the parent span and store the current span back into GLS. As shown in the following logical code example:

```
// create new local span
func CreateLocalSpan(name string) span {
        parentSpan := GetGLS()
        var result span
        if parentSpan == nil {
                result = newRootSpan(GetGlobalTracer())
        } else {
                result = newSpan(opName, parentSpan)
        }
        SetGLS(result)
        return result
}
// found current active span
func ActiveSpan() span {
        return GetGLS()
}
```

##### Runtime Context with GLS

　　Currently, Java Agent has the functionality of "Context.getRuntimeContext().put("test", "value")", which can also be used in the Golang Agent. The current idea is to store the segment span and runtime context in a single instance within GLS. For example, create the following structure and store it in GLS:

```
type tracingContext struct {
        activeSpan span
        runtimeContext map[string]interface{}
}
```

#### Auto-instrument Agent with Dependencies

　　Since the enhanced agent is based on the compilation process, it lacks the ability to import other modules into the application. To achieve communication between the agent and the OAP server, it is necessary to introduce key modules, such as the Tracing data model and the gRPC communication protocol.

![Auto-instrument](/assets/technique/Go语言实现无侵入埋点/Auto-instrument.png)

##### go2sky mechanism

　　Therefore, we need to use "go2sky" as a base module to integrate into the target application, making the integration of the automatic Agent and go2sky crucial.

　　First, we need to understand how go2sky works when building spans in the tracing context.

![go2sky](/assets/technique/Go语言实现无侵入埋点/go2sky.png)

　　As shown in the illustration above, it relies on the "ctx.Context" instance in "Golang". Since it cannot be some method Thread Local, data sharing within and across goroutines can only be accomplished through Context. Additionally, different frameworks need context for invoking, and by using the Context Propagation mechanism, information from the current and parent Contexts can be traced.

　　In go2sky, there is no concept of a parent Span. Therefore, if an inherited context is used at the same level, such as after completing a Local Span in the Context and using the returned Context from the Local Span, the parent of the Local Span would be the Exit Span instead of the upper-level Span.

##### Customized Agent

　　Referring to the "go2sky" project, when we want to create a base library, we will copy the key code from "go2sky" and make the following modifications.

　　For the modules introduced in the user's project, the following functionalities will be provided:

　　　　1. Introduce the gRPC and goapi modules.

　　　　2. Provide the code related to gRPC and OAP communication.

　　　　3. Provide the basic API originally offered by go2sky, such as Correlation, TraceID, and other functionalities.

　　Enhancing the code during the hybrid compilation phase, including the following parts:

　　　　1. Introduce Agent Core-related features(Copy files into agent).

　　　　2. Communicate with the customized Agent's protocol and send Segment data.

　　　　3. Export the global tracer in the main package.

　　　　4. Enhance the implementation of the provided external APIs.

##### Tracing Context with GLS

　　In go2sky, span information is passed through the Context object. We need to replace this with the GLS mechanism, as mentioned in "GoRoutine with Tracing Context.". Also, when saving Span information, we need to save a reference to the parent Span together, otherwise, we may encounter issues with not finding the parent Span and binding the "parentSpanID" incorrectly. the code like below:

```
func (s *span) End() {
        core.SetGLS(span.parentSpan())
}
```

#### Project Structure

　　The new agent project is should organize all project modules within a single repository to make it easier to manage dependencies and maintain consistency across modules. We can use Go modules to manage your dependencies and import the necessary modules within user code. Here's a suggested directory structure for the agent project:

![Project_Structure](/assets/technique/Go语言实现无侵入埋点/Project_Structure.png)

　　　　1. docs: This directory manages all documentation information for the current project. It can be synchronized with the Apache SkyWalking Website to keep the documentation up-to-date and consistent.

　　　　2. plugins: This directory stores the base code for all plugins and the implementation of each individual plugin. The core subdirectory contains the base code, and other subdirectories contain plugin implementations. Plugins need to import the core module.

　　　　3. reporter: Responsible for reporting the Tracing data collected from plugins to the OAP backend.

　　　　4. test/e2e: Contains end-to-end tests for each different plugin to ensure their proper functioning. We can still use skywalking-infra-e2e for the testing of plugins. Using the SkyWalking Agent Test Tool as the server-side component and validator will help ensure compatibility and consistency with the Apache SkyWalking ecosystem.

　　　　5. toolkit: In order to provide users with a set of plugins for common use cases, such as integrating tracing context into logging systems.

　　　　6. tools/go-agent-enhance: Used for enhancing user programs. When users need to use this during their project build, they need to download the current package and specify it during the build process. For example: "go download github.com/apache/skywalking-go/tools/go-agent-enhance && go build -toolexec=$(/path/to/go-agent-enhance)"

　　　　7. api.go: Users can use this API to obtain current Tracing data content, such as the current TraceID, Correlation information, etc.

