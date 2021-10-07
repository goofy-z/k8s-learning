## kubernetes构建方式

k8s的代码拥有大约360多万行，GO语言占300多万

k8s构建方式可以分为3种，分别是本地环境构建、容器环境构建、Bazel环境

### 1 本地构建

make或make all命令会编译k8受的所有组件，输出的二进制文件的相对路径是_output/bin/。单独编译某个组件如kubectl，`make WHAT=cmd/kubectl`

#### makefile

Go语言开发这习惯手动执行go build和go test(单元测试)命令，但在大型项目或复杂的项目，最好去使用Makefile，它还适用于大多数编程语言。k8s的源码种有两个Makefile相关文件。

- Makefile：顶层Makefile文件，描述了整个项目所有代码文件的编译顺序，编译规则和编译后的二进制输出。
- Makefile.generated_files: 描述了代码生成的逻辑。

### 2 容器环境构建

### 3 Bazel环境构建

### 4 代码生成器

在generated_files中定义了5个代码生成器

