## 面向配置编程

配置，对于刚刚入行的程序员来说，很容易忽视它。甚至很多时候，直接将配置写死在程序里。
随着工作经历的增加，不管是因为编程规范的强制调教，还是因为真的发现配置提取出来更加方便。
大家还是都学会了尽量将配置从程序中提取出来的做法，而不是再像一开始时写死在程序里。
但是，不管是写死在代码里，还是提取出来放到文件里或git里。
对于很多人来说，配置都仅仅是程序的一个辅助功能而已。

如果配置的内容仅仅是描述系统中某些属性的值，那么以这种方式看待配置是合适的。
比如，在配置中指定数据库的连接方式，使用的线程数，内存使用的大小等等。
这类配置都是在设置程序中某些属性的值，因此含义和使用都比较简单。

但是，如果配置是与业务系统紧密结合的话，那么这种将配置视为辅助功能的方式就不合适了。
通常，涉及业务系统的配置，都带有很强的业务场景含义和对应的组织结构。
比如在风控场景中，提取哪些特征、计算哪些规则、使用哪种模型、做出哪种决策，
这些都是与风控场景的业务逻辑强相关的。

因此在业务场景下，从某种意义上来讲，配置才是程序的核心所在。就像风控场景里，提取哪些特征、计算哪些规则、使用哪种模型、做出哪种决策，
当我们在配置文件中，配置好这些因素后，程序无非就是读出这些配置，然后按照这些配置的逻辑，解析和执行即可。

我们不妨思考两个问题。
脚本是配置吗？当然是，如果将脚本解释器视为一个普通程序，脚本解释器读取脚本，按照脚本定义的过程和规则执行。从这个角度上将，脚本就是解释器的配置文件。
配置是脚本吗？当然是，如果我们将程序视为一个解释器，配置就是这个解释器的输入。从这个角度上将，配置就是程序的脚本。

因此，在业务逻辑比较复杂的情况下，我们不妨以配置为核心来指导程序的开发，这就是面向配置编程。
当按照面向配置编程的思路来设计程序时，就好像我们在开发一个脚本解释器。
而众所周知，相比普通专用程序而言，解释器的灵活性更好，因为当解释器开发好后，
开发业务逻辑只需要编写脚本即可，而不再需要修改解释器本身。

与脚本解释器的结构非常类似。面向配置编程，包含两个部分：
* 配置
正如前文所说，当涉及业务逻辑时，配置才是描述系统执行逻辑的核心所在。
因此，针对具体业务场景，设计合适的配置项目和配置组织结构，是配置设计的核心所在。
* 引擎
引擎的开发应该围绕着配置来进行。当配置在设计好后，按照配置表达的业务逻辑，开发对应的执行引擎。


## 面向配置编程的好处
* 灵活和轻便
当面向配置编程的引擎在开发完成后，只要整体逻辑不变，调整业务只需要编写或修改对应的配置文件就可以了。
而不需要再修改程序，重新构建测试和部署上线。极大地缩短了业务上线周期。

* 简洁和透明
在使用面向配置编程时，编写配置文件相比程序开发会简单和透明很多，因为配置编写的过程直接就是实现业务逻辑的过程。

* 抽象和泛化
面向配置编程开发的的引擎，相当于一个脚本解释器，它是对业务执行流程的终极抽象。
而配置的灵活性，使得只要是在这个业务执行流程的框架下，我们可以任意的设置业务流程的各种指标或参数，可以说是对业务执行流程的终极泛化。

### 更高级的配置：领域特定语言
面向配置编程的一种更高级形式，就是领域特定语言（Domain Specified Languag，后文简称DSL）了。

所谓DSL，是指针对特定领域而设计和开发的一种专用计算机程序语言。
相比通用语言而言，DSL具有有限的表达能力，或者说DSL不是图灵完备的。

DSL特别适合用于实现业务逻辑。那DSL相比前面说到的一般配置，又"高级"了哪些内容呢？
其实，如果我们用JSON来表示配置，那JSON就是一种DSL了。
通常的配置可以表达配置项清单以及各配置项间的层次和依赖关系，但是不能表达符号和计算逻辑，比如变量、操作符和函数等等。
DSL能够表达的内容会更丰富些。既然DSL号称为语言，那就或多或少具有计算机语言的某些特性，
比如它能够支持变量、操作符和函数等等。
但是相比通用编程语言，DSL又不是图灵完备的。换言之，你不能用DSL实现C语言，但是能够用C语言实现DSL。

虽然相比普通配置，DSL功能会更加强大，但是这是有代价的。功能越强的DSL，开发的难度和复杂度也越高。
相较而言，用JSON表示的配置是一种最简单的DSL，在语言层面基本上无需开发，就没什么开发难度了。
