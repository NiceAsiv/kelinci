# README #
[English](https://github.com/NiceAsiv/kelinci)
用于在Java程序上运行AFL的界面。

Kelinci在印度尼西亚语（爪哇岛上的语言）中意为兔子。

此README假设您已经安装了AFL。有关如何安装和使用AFL的信息，请参见 <http://lcamtuf.coredump.cx/afl/>。Kelinci已成功测试了AFL版本2.44及更新版本。README解释了如何使用该工具。有关技术背景，请参阅'docs'目录中的CCS'17论文。

在'examples'目录中提供了几个示例，每个示例都附有一个README，指定了重复实验的确切步骤。

Kelinci已经发现了与JPEG解析相关的错误，在OpenJDK（版本6-9）和Apache Commons Imaging 1.0 RC7中。这些示例都包含在提供的示例中。这是bug报告：
- http://bugs.java.com/bugdatabase/view_bug.do?bug_id=JDK-8188756
- https://issues.apache.org/jira/browse/IMAGING-203

### 安装 ###

该应用程序有两个组件。首先，有一个C应用程序，作为AFL的目标应用程序。
它的行为与使用afl-gcc / afl-g++构建的应用程序相同；AFL无法区分差异。
这个C应用程序位于子目录'fuzzerside'中。它将AFL生成的输入文件通过TCP连接发送给JAVA侧。
然后接收结果并以AFL期望的格式转发给AFL。要构建，请在'fuzzerside'子目录中运行`make`。

第二个组件位于JAVA侧。它位于'instrumentor'子目录中。
这个组件为目标应用程序提供AFL风格的管理，并加入了与C侧通信的组件。
当稍后执行被插桩的程序时，这会设置一个TCP服务器，并为每个传入请求在单独的线程中运行目标应用程序。
它会发送回一个退出代码（成功、超时、崩溃或队列已满），以及收集的路径信息。任何从main逃逸的异常都被视为崩溃。
要构建，请在'instrumentor'子目录中运行`gradle build`。

### 使用 ###

这描述了如何在目标应用程序上运行Kelinci。它假设已经构建了AFL和两个Kelinci组件。

**1. 可选：构建驱动程序**
AFL/Kelinci期望一个程序，该程序接受一个指定文件位置的参数。它随机变异文件以对这个程序进行模糊测试。如果目标应用程序的工作方式不是这样，那么需要构建一个驱动程序，该驱动程序解析输入文件，并根据该文件模拟正常交互。构建驱动程序时，请记住输入文件将被随机变异。文件中程序期望的结构和凝聚力越少，模糊测试就越有效。即使程序确实接受输入文件，构建一个接受不同格式的驱动程序也可能是有意义的，以最大化有效版本无效输入文件的比率。

构建驱动程序时，还需考虑的一点是目标程序将在VM中运行，其中main方法将被一次又一次地调用。所有运行都必须是独立和确定的。例如，如果程序将输入的信息存储在数据库或静态内存位置，请确保重置它，以便不影响未来的运行。

**2. 插桩**
我们假设目标应用程序和驱动程序已经构建，输出目录为'bin'。我们的下一步是为使用Kelinci而对类进行插桩。该工具提供了edu.cmu.sv.kelinci.instrumentor.Instrumentor类来实现这一点。它在`-i`标志后面接受一个输入目录（这里是'bin'），在`-o`标志后面接受一个输出目录（这里是'bin-instrumented'）。我们需要确保kelinci JAR在类路径上，以及目标应用程序的所有依赖项。假设目标应用程序依赖的JAR位于/path/to/libs/，则插桩的命令如下所示：

```java -cp /path/to/kelinci/instrumentor/build/libs/kelinci.jar:/path/to/libs/* edu.cmu.sv.kelinci.instrumentor.Instrumentor -i bin -o bin-instrumented```

请注意，如果项目依赖的库版本与Kelinci Instrumentor依赖的版本不同，可能会遇到问题。目前，这些是args4j版本2.32，ASM 5.2和Apache Commons IO 2.4。在大多数情况下，可以通过将Kelinci构建的'classes'目录放在类路径上而不是Fat JAR上，然后将一个版本的库JAR添加到类路径上，以便Kelinci和目标都可以工作。

**3. 创建示例输入**
我们现在想测试被插桩的Java应用程序是否正常工作。为此，请创建一个示例输入文件的目录：
```mkdir in_dir```

AFL稍后将使用此目录来获取它将变异的输入文件。因此，将有代表性的输入文件放在那里是非常重要的。在那里复制有代表性的文件，或者创建它们。

**4. 可选：测试Java应用程序**
查看被插桩的Java应用程序是否能够使用提供/创建的输入文件正常工作：
```java -cp bin-instrumented:/path/to/libs/* <driver-classname> in_dir/<filename>```

**5. 启动Kelinci服务器**
我们现在可以启动Kelinci服务器了。我们将简单地编辑我们执行的最后一个命令，它运行了Java应用程序。Kelinci期望目标应用程序的主类作为第一个参数，因此我们现在只需在那个之前添加Kelinci的主类。我们还需要将具体的文件名替换为`@@`，Kelinci将用它写入的输入文件的实际路径替换它。其他参数可以保持不变，将在运行中固定。

```java -cp bin-instrumented:/path/to/libs/* edu.cmu.sv.kelinci.Kelinci <driver-classname> @@```

可选地，我们可以指定一个端口号（默认是7007）：
```java -cp bin-instrumented:/path/to/libs/* edu.cmu.sv.kelinci.Kelinci -port 6666 <driver-classname> @@```

**6. 可选：测试接口**
在我们开始使用模糊器之前，让我们确保与Java侧的连接按预期工作。`interface.c`程序也有一个在AFL之外运行的模式，因此我们可以按如下方式测试它：
```/path/to/kelinci/fuzzerside/interface in_dir/<filename>```

如果我们在第6步中创建了服务器列表，我们可以如下添加：
```/path/to/kelinci/fuzzerside/interface -s servers.txt in_dir/<filename>```

可选地，我们可以使用-s标志指定一个服务器（例如-s 192.168.1.1或"sv.cmu.edu"，默认是"localhost"）和-p标志指定端口号（默认是7007）。

**7. 启动fuzzing！**
如果一切都按预期工作，现在我们可以启动AFL！与Kelinci服务器端类似，AFL期望一个以输入文件为参数的二进制文件，由`@@`指定。在我们的情况下，这始终是`interface`二进制文件。它还期望一个包含要开始模糊测试的输入文件的目录，以及一个输出目录。

```/path/to/afl/afl-fuzz -i in_dir -o out_dir /path/to/kelinci/fuzzerside/interface [-s servers.txt] @@```

如果一切正常，AFL接口将在短时间内启动，您将注意到正在发现新的路径。为了进行额外的监视，请查看输出目录。'queue'子目录中的输入文件触发了不同的程序行为。还有'crashes'和'hangs'子目录，其中包含导致崩溃或超时的输入。请参阅AFL网站以获取更多详细信息：http://lcamtuf.coredump.cx/afl/

### 关于并行化的注意事项 ###

Java端天然支持并行化。只需为要在其上执行Java运行的每个核心启动一个实例。这可以在同一台机器上（但端口不同！）或在多台机器上进行。

有关如何并行运行AFL的详细信息，请参阅附带的`parallel_fuzzing.txt`文档。您将希望有尽可能多的afl-fuzz进程运行，就像有Java端Kelinci组件一样，其中每个afl-fuzz进程连接到不同的Kelinci服务器。要连接到的Kelinci服务器可以使用`-s <server>`和`-p <port>`标志为`interface.c`指定。

### 开发者 ###

Rody Kersten（rodykersten@gmail.com）
