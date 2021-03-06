# javaagent

## javaagent介绍

`java.lang.Instrument`包是在JDK5引入的，开发者可以通过修改方法的字节码实现动态修改类代码。javaagent的主要功能如下：

* 可以在加载class文件之前做拦截，对字节码做修改\(`premain`\)，可以实现`AOP`、修改源代码等功能。
* 可以在运行期对已加载类的字节码做变更\(`agentmain`\)，可以实现**热部署**等功能。
* 还有其他一些小众的功能
  * 获取所有已经加载过的类
  * 获取所有已经初始化过的类（执行过clinit方法，是上面的一个子集）
  * 获取某个对象的大小
  * 将某个jar加入到bootstrap classpath里作为高优先级被bootstrapClassloader加载
  * 将某个jar加入到classpath里供AppClassloard去加载
  * 设置某些native方法的前缀，主要在查找native方法的时候做规则匹配

## premain用法示例

假如在项目`agent-demo`中引入一个第三方jar包`dog-1.0.0.jar`，其中有一个类`com.maxwell.learning.javaagent.dog`，代码如下：

```java
public class Dog{
    public toString(){
        return "i am a cute dog";
    }
}
```

现在，想修改这个类的`toString()`方法，修改为

```java
public class Dog{
    public toString(){
        return "i am a cruel dog";
    }
}
```

使用`java agent`来修改Dog类的源码，需要写一个`agent`和`ClassFileTransformer`：

`DogAgent`

```java
public class DogAgent {
    private static final Logger logger = Logger.getLogger(DogAgent.class.getName());
    private DogAgent() {
        throw new InstantiationError("Must not instantiate this class");
    }
    public static void premain(String agentArgs, Instrumentation inst) {
        ClassFileTransformer transformer = new DogTransformer();
        inst.addTransformer(transformer, true);
    }
}
```

`ClassFileTransformer`

```java
public class DogTransformer implements ClassFileTransformer {
    private static final Logger logger = Logger.getLogger(DogTransformer.class.getName());

    private static final String DOG_CLASS_NAME="com.maxwell.learning.javaagent.dog.Dog";
    @Override
    public byte[] transform(ClassLoader loader, String classFile, Class<?> classBeingRedefined,
                            ProtectionDomain protectionDomain, byte[] classFileBuffer) {
        try {
            final String className = toClassName(classFile);
            if (DOG_CLASS_NAME.equals(className)) {
                logger.info("Transforming class " + className);
                CtClass clazz = getCtClass(classFileBuffer, loader);
                //get doString() method
                CtMethod method = clazz.getDeclaredMethod("toString");
                //modify toString() method
                method.setBody("return \"i am a cruel dog\";");
                return clazz.toBytecode();
            }
        } catch (Throwable t) {
            logger.log(Level.SEVERE, "Transforming class error", t);
        }
        return null;
    }
    private static String toClassName(String classFile) {
        return classFile.replace('/', '.');
    }
    private static CtClass getCtClass(byte[] classFileBuffer, ClassLoader classLoader) throws IOException {
        ClassPool classPool = new ClassPool(true);
        if (classLoader == null) {
            classPool.appendClassPath(new LoaderClassPath(ClassLoader.getSystemClassLoader()));
        } else {
            classPool.appendClassPath(new LoaderClassPath(classLoader));
        }
        CtClass clazz = classPool.makeClass(new ByteArrayInputStream(classFileBuffer), false);
        clazz.defrost();
        return clazz;
    }
}
```

 将`DogAgent`和`DogTransformer`打成`jar`包，打包时，添加`MANIFEST.MF`信息，可配置`pom.xml`如下：

```markup
<dependencies>
    <!-- 修改字节码时，使用javassist -->
    <dependency>
        <groupId>org.javassist</groupId>
        <artifactId>javassist</artifactId>
        <version>3.22.0-GA</version>
    </dependency>
</dependencies>
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <version>2.4</version>
            <configuration>
                <archive>
                    <!--打包时，设置MANIFEST.MF的信息-->
                    <manifestEntries>
                        <Premain-Class>com.maxwell.learning.javaagent.dogagent.DogAgent</Premain-Class>
                        <Can-Redefine-Classes>true</Can-Redefine-Classes>
                        <Can-Retransform-Classes>true</Can-Retransform-Classes>
                    </manifestEntries>
                </archive>
            </configuration>
        </plugin>
        <plugin>
            <!-- 防止引用该jar的项目也使用javassist，与jar中javassist冲突-->
            <artifactId>maven-shade-plugin</artifactId>
            <version>3.1.0</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                    <configuration>
                        <createSourcesJar>true</createSourcesJar>
                        <relocations>
                            <relocation>
                                <pattern>javassist</pattern>
                                <shadedPattern>com.maxwell.learning.javaagent.dogagent.javassist</shadedPattern>
                            </relocation>
                        </relocations>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

 在agent-demo中，使用Dog类，

```java
public class Client {
    public static void main(String[] args) {
        Dog dog = new Dog();
        System.out.println(dog.toString());
    }
}
```

运行`Client`，输出：`i am a cute dog`

添加`vm`参数：`-javaagent:/path/to/dog-agent-1.0.0.jar`，再次运行`Client`输出如下：

`i am a cruel dog`

## agentmain用法示例

延续上面的例子，我们想在项目运行后，将Dog的toString\(\)方法修改为：

```java
public class Dog{
    public toString(){
        return "i am a gentle dog";
    }
}
```

与premain类似，编写`agent`和`ClassFileTransformer`：

DogAgent：与premain不同，这是agentmain方法，并且加了一句`inst.retransformClasses(Dog.class)`（）

```java
public class DogAgent {
    
    private DogAgent() {
        throw new InstantiationError("Must not instantiate this class");
    }

    public static void agentmain(String agentArgs, Instrumentation inst) throws UnmodifiableClassException {
        ClassFileTransformer transformer = new DogTransformer();
        inst.addTransformer(transformer, true);
        //告诉JVM，要重新转换Dog这个类，注意：premain的时候，不必加这句代码
        inst.retransformClasses(Dog.class);
    }
}
```

 `ClassFileTransformer`与之前逻辑一致，只是对toString\(\)修改不同：

```java
public class DogTransformer implements ClassFileTransformer {
    private static final String DOG_CLASS_NAME="com.maxwell.learning.javaagent.dog.Dog";
    @Override
    public byte[] transform(ClassLoader loader, String classFile, Class<?> classBeingRedefined,
                            ProtectionDomain protectionDomain, byte[] classFileBuffer) {
        try {
            final String className = toClassName(classFile);
            if (DOG_CLASS_NAME.equals(className)) {
                System.out.println("Transforming class " + className);
                CtClass clazz = getCtClass(classFileBuffer, loader);
                //modify toString() methdo
                CtMethod method = clazz.getDeclaredMethod("toString");
                method.setBody("return \"i am a gentle dog\";");
                return clazz.toBytecode();
            }
        } catch (Throwable t) {
            t.printStackTrace();
        }
        return null;
    }

    private static String toClassName(String classFile) {
        return classFile.replace('/', '.');
    }
    private static CtClass getCtClass(byte[] classFileBuffer, ClassLoader classLoader) throws IOException {

        ClassPool classPool = new ClassPool(true);
        if (classLoader == null) {
            classPool.appendClassPath(new LoaderClassPath(ClassLoader.getSystemClassLoader()));
        } else {
            classPool.appendClassPath(new LoaderClassPath(classLoader));
        }

        CtClass clazz = classPool.makeClass(new ByteArrayInputStream(classFileBuffer), false);
        clazz.defrost();
        return clazz;
    }
}
```

将`DogAgent`和`DogTransformer`打成`jar`包，打包时，添加`MANIFEST.MF`信息，与premain类似，但是指定的是Agent-Class：

```markup
<dependencies>
        <!-- 修改字节码时，使用javassist -->
        <dependency>
            <groupId>org.javassist</groupId>
            <artifactId>javassist</artifactId>
            <version>3.22.0-GA</version>
        </dependency>
        <dependency>
            <groupId>com.maxwell</groupId>
            <artifactId>dog</artifactId>
            <version>1.0.0</version>
            <scope>compile</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>2.4</version>
                <configuration>
                    <archive>
                        <!--打包时，设置MANIFEST.MF的信息-->
                        <manifestEntries>
                            <Agent-Class>com.maxwell.learning.javaagent.dogagentmain.DogAgent</Agent-Class>
                            <Can-Redefine-Classes>true</Can-Redefine-Classes>
                            <Can-Retransform-Classes>true</Can-Retransform-Classes>
                        </manifestEntries>
                    </archive>
                </configuration>
            </plugin>
            <plugin>
                <!-- 防止引用该jar的项目也使用javassist，与jar中javassist冲突-->
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.1.0</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <createSourcesJar>true</createSourcesJar>
                            <relocations>
                                <relocation>
                                    <pattern>javassist</pattern>
                                    <shadedPattern>com.maxwell.learning.javaagent.dogagentmain.javassist</shadedPattern>
                                </relocation>
                            </relocations>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

 修改Cleint代码如下，使其持续打印：

```java
public class DogClient {
    public static void main(String[] args) {
        Dog dog = new Dog();
        while (true){
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(dog.toString());
        }
    }
}
```

 agentmain的用法或者说修改字节码的方式与premain不同，agentmain是在类已经加载原始类之后，再去修改JVM中的Class对象。既然Client已经启动，这时怎么去修改呢？Sun 公司提供的一套扩展 Attach API，用来向目标 JVM ”附着”（Attach）代理工具程序的。有了它，开发者可以方便的监控一个 JVM，运行一个外加的代理程序。

新写一个类如下

```java
public class AttachToJVM {
    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException {
        //获取本机所有JVM实例
        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        for (VirtualMachineDescriptor vmd : list) {
            //如果是我们刚才运行的DogClient所在的虚拟机
            if(vmd.displayName().endsWith("DogClient")){
                VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());
                virtualMachine.loadAgent("/Users/yue/Desktop/dog-agent-main-1.0.0.jar");
                System.out.println("load dog agent main...");
                virtualMachine.detach();
            }
        }
    }
}
```

 现在，运行DogClient（无需添加任何运行参数），让它打印一会，再启动AttachToJVM，输出如下：

```text
i am a cute dog
i am a cute dog
i am a cute dog
    ... ... 
i am a cute dog
i am a cute dog
i am a cute dog
Transforming class com.maxwell.learning.javaagent.dog.Dog
i am a gentle dog
i am a gentle dog
i am a gentle dog
    ... ...
i am a gentle dog
i am a gentle dog
i am a gentle dog

```

成功在运行时修改了Dog的源码！

## javaagent细节

1.代理 \(agent\) 是在你的main方法前的一个拦截器 \(interceptor\)，也就是在main方法执行之前，执行agent的代码。写一个java agent只需要实现`premain`这个方法：

```text
public static void premain(String agentArgs, Instrumentation inst)
public static void premain(String agentArgs)
```

2. Agent 类必须打成jar包，然后里面的`META-INF/MAINIFEST.MF`必须包含`Premain-Class`这个属性。 

> Manifest Attributes清单：
>
> **Premain-Class** ：包含 premain 方法的类（类的全路径名）。
>
> **Agent-Class** ：包含 agentmain 方法的类（类的全路径名）

> **Boot-Class-Path** ：设置引导类加载器搜索的路径列表。查找类的特定于平台的机制失败后，引导类加载器会搜索这些路径。按列出的顺序搜索路径。列表中的路径由一个或多个空格分开。路径使用分层 URI 的路径组件语法。如果该路径以斜杠字符（“/”）开头，则为绝对路径，否则为相对路径。相对路径根据代理 JAR 文件的绝对路径解析。忽略格式不正确的路径和不存在的路径。如果代理是在 VM 启动之后某一时刻启动的，则忽略不表示 JAR 文件的路径。（可选）

> **Can-Redefine-Classes** ：true表示能重定义此代理所需的类，默认值为 false。 （可选）
>
> **Can-Retransform-Classes** ：true 表示能重转换此代理所需的类，默认值为 false。 （可选）
>
> **Can-Set-Native-Method-Prefix：** true表示能设置此代理所需的本机方法前缀，默认值为 false。（可选）

3. 所有的这些Agent的jar包，都会自动加入到程序的classpath中。所以不需要手动把他们添加到classpath。除非你想指定classpath的顺序。

4. 一个java程序中-javaagent参数的个数是没有限制的，所以可以添加任意多个java agent。所有的java agent会按照你定义的顺序执行。 例如：

`java -javaagent:MyAgent1.jar -javaagent:MyAgent2.jar -jar MyProgram.jar`

程序执行的顺序将会是：

`MyAgent1.premain -> MyAgent2.premain -> MyProgram.main`

5. 放在main函数之后的premain是不会被执行的，例如：

`java -javaagent:MyAgent1.jar -jar MyProgram.jar -javaagent:MyAgent2.jar`

MyAgent2 放在了MyProgram.jar后面，所以MyAgent2中的premain都不会被执行。

6. 每一个java agent 都可以接收一个字符串类型的参数，也就是premain中的agentArgs，例如：

`java -javaagent:MyAgent2.jar=thisIsAgentArgs -jar MyProgram.jar`

MyAgent2中`premain`接收到的`agentArgs`的值将是`thisIsAgentArgs`（不包括双引号）。

7. 参数中的Instrumentation：通过参数中的Instrumentation inst，添加自己定义的ClassFileTransformer，来改变class文件。这里自定义的Transformer实现了transform方法，在该方法中提供了对实际要执行的类的字节码的修改，甚至可以达到执行另外的类方法的地步。

TODO：

①instrument的其他功能，如BootClassPath / SystemClassPath 的动态增补等

②在判断要修改哪些类时，是通过类名equals来判断的，但是有些Java中rt.jar中的类如Thread，ArrayList等，这种方式行不通，因为ClassFileTransformer好像拦截不到这些类，因为下面的这个方法没有这些类的输出，但是对于其他的如ThreadPoolExecutor等又能拦截到。有待解决...

```java
public byte[] transform(ClassLoader loader, 
                            String classFile, Class<?> classBeingRedefined,
                            ProtectionDomain protectionDomain, 
                            byte[] classFileBuffer) {
```

## 参考

[java.lang.Instrument 代理Agent使用](http://www.importnew.com/22466.html)

[java 8 docs Package java.lang.instrument](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html)

[JVM源码分析之javaagent原理完全解读](http://www.infoq.com/cn/articles/javaagent-illustrated)

[Apache Maven JAR Plugin](https://maven.apache.org/plugins/maven-jar-plugin/)

[How to create a manifest file with Maven](https://www.mkyong.com/maven/how-to-create-a-manifest-file-with-maven/)

[Instrumentation 新功能](https://www.ibm.com/developerworks/cn/java/j-lo-jse61/index.html)：写的非常全面，强烈推荐！

[探秘 Java 热部署三（Java agent agentmain）](https://www.jianshu.com/p/6096bfe19e41)：对agentmain写的非常详细，推荐！

[transmittable-thread-local](https://github.com/alibaba/transmittable-thread-local)：premain的实际应用，对JDK的Runnable和Callable进行了装饰。

[Github JavaLanguage javaagent](https://github.com/maxwellyue/JavaLanguage/tree/master/javaagent)：文中示例代码的地址，可以实际运行来体验一下

扩展阅读

[技术问答集锦（13）Java Instrument原理](https://juejin.im/post/5ad5ac7351882555784e7667)

[谈谈Java Intrumentation和相关应用](http://www.fanyilun.me/2017/07/18/%E8%B0%88%E8%B0%88Java%20Intrumentation%E5%92%8C%E7%9B%B8%E5%85%B3%E5%BA%94%E7%94%A8/)

