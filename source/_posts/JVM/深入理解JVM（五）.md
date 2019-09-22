---
title: 深入理解JVM（五） --- 早期编译
date: 2019-09-22 20:47:12
tags: [JAVA,JVM]
categories: JAVA


---

<!-- more -->

Java的编译包括前端编译、后端编译，前端编译器把`*.java`编译为`*.class`，而后端编译有运行期编译（JIT）、提前编译（AOT）等。在本篇中主要将前端编译（早期编译）。

#### Javac编译过程

javac的编译过程，主要分为3个过程：

- 解析与填充符号表
- 插入式注解处理
- 语义分析与生成字节码

![image-20190922212854084](/Users/qudian/Library/Application Support/typora-user-images/image-20190922212854084.png)

在openjdk中，java的编译入口为`com.sun.tools.javac.main.JavaCompiler`类中的compile与compile2方法，代码如下：

compile:

```java
public void compile(List<JavaFileObject> sourceFileObjects,
                        List<String> classnames,
                        Iterable<? extends Processor> processors)
        throws IOException // TODO: temp, from JavacProcessingEnvironment
    {
        if (processors != null && processors.iterator().hasNext())
            explicitAnnotationProcessingRequested = true;
        // as a JavaCompiler can only be used once, throw an exception if
        // it has been used before.
        if (hasBeenUsed)
            throw new AssertionError("attempt to reuse JavaCompiler");
        hasBeenUsed = true;

        start_msec = now();
        try {
            initProcessAnnotations(processors);//初始化注解处理器

            //必须以链式调用以避免内存溢出
            delegateCompiler =
                processAnnotations(//2.执行注解处理
                    enterTrees(//1.2 输入到符号表
                      stopIfError(CompileState.PARSE, 
                                  	parseFiles(sourceFileObjects)) // 1.1 词法、语法分析
                    							),classnames);
						delegateCompiler.compile2();//3.语义分析与生成字节码
            delegateCompiler.close();
            elapsed_msec = delegateCompiler.elapsed_msec;
        } catch (Abort ex) {
            if (devVerbose)
                ex.printStackTrace();
        } finally {
            if (procEnvImpl != null)
                procEnvImpl.close();
        }
    }
```

compile2:

```java
private void compile2() {
        try {
            label44:
            switch(this.compilePolicy) {
            case ATTR_ONLY:
                this.attribute((Queue)this.todo);
                break;
            case CHECK_ONLY:
                this.flow(this.attribute((Queue)this.todo));
                break;
            case SIMPLE:
                this.generate(this.desugar(this.flow(this.attribute((Queue)this.todo))));
                break;
            case BY_FILE:
                Queue var1 = this.todo.groupByFile();

                while(true) {
                    if (var1.isEmpty() || this.shouldStop(CompileState.ATTR)) {
                        break label44;
                    }

                    this.generate(this.desugar(this.flow(this.attribute((Queue)var1.remove()))));
                }
            case BY_TODO:
                while(true) {
                    if (this.todo.isEmpty()) {
                        break label44;
                    }
										//
                    this.generate(//生成字节码
                      this.desugar(//反语法糖
                        this.flow(//数据流分析
                          this.attribute((Env)this.todo.remove())//3.1标注
                        )
                      )
                    );
                }
            default:
                Assert.error("unknown compile policy");
            }
        } catch (Abort var2) {
            if (this.devVerbose) {
                var2.printStackTrace(System.err);
            }
        }

        if (this.verbose) {
            this.elapsed_msec = elapsed(this.start_msec);
            this.log.printVerbose("total", new Object[]{Long.toString(this.elapsed_msec)});
        }

        this.reportDeferredDiagnostics();
        if (!this.log.hasDiagnosticListener()) {
            this.printCount("error", this.errorCount());
            this.printCount("warn", this.warningCount());
        }

    }
```



#### 1.解析与填充符号表

1. 词法分析、语法分析

词法分析是将代码的字符流转换为Token集合，例如：int a = b + c，则int 、a、b、c都为一个token。

语法分析则是根据Token集合构造抽象语法树（Abstract Syntax Tree, AST）的过程，语法树中的每一个节点都代表程序中的一个语法结构,最终产生一个JCTree对象，而经过词法、语法分析后，javac编译器就不会再对源码文件进行操作了。

2. 填充符号表

符号表是一组符号地址与符号信息构成的表格，类似与HashMap。填充符号表则是将每一个编译单元的抽象语法树根节点装入一个ToDoList列表中。在语义分析时，符号表会被用做语义检查与生成中间代码；在目标代码生成阶段进行符号名分配地址时，符号表将是地址分配的依据。

#### 2. 注解处理器

注解处理器首先在initProcessAnnotations方法中进行，执行过程在processAnnotations方法完成。

#### 3.语义分析与字节码生成

词法、语法分析后，编译器已经获得了代码的AST。而语义分析主要任务是对AST结构上进行上下文类的审查，语义分析过程主要分为标注检查、数据控制流分析以及解语法糖三个步骤。

1. 标注检查

标注检查内容主要为检查变量使用前是否已被声明、变量类型是否正确、变量是否赋值等，此外还有一个重要的动作为`常量折叠`。比如一行代码：`int a = 1 + 2`，经过标注检查的常量折叠后，a的值会被折叠为3.

2. 数据及控制流分析

数据与控制流分析是对程序上下文逻辑的进一步验证，它可以检查出如局部变量是否赋值、方法的每个路径是否有返回值、受检异常是否有被处理等

3. 解语法糖

解语法糖主要将java中的语法糖语法如泛型、自动装箱/拆箱、可变参数等语法进行转换为基础java代码的过程

字节码生成：

字节码生成是javac编译过程等最后一个阶段，主要将前面各个步骤生成的信息转化成字节码到磁盘中，并且还有少量代码添加、转换工作。例如实例构造器<init>()方法和类构造器方法<cinit>()就是在这个阶段加入到语法树中的。在完成了语法树的遍历与调整后，就会把所有所需信息的符号表交给`ClassWriter`类，生成最终的Class文件。