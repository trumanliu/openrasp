# OpenRASP 代码解读

查看 agent/java/boot 工程代码，pom 文件中的配置： 

```xml
<build>
        <finalName>rasp</finalName>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>2.4</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                    </execution>
                </executions>
                <configuration>
                    <archive>
                        <manifestEntries>
                            <Premain-Class>com.baidu.openrasp.Agent</Premain-Class>
                            <Agent-Class>com.baidu.openrasp.Agent</Agent-Class>
                            <Main-Class>com.baidu.openrasp.Agent</Main-Class>
                            <Can-Redefine-Classes>true</Can-Redefine-Classes>
                            <Can-Retransform-Classes>true</Can-Retransform-Classes>
                        </manifestEntries>
                    </archive>
                </configuration>
            </plugin>
        </plugins>
</build>
```

其中 Premain-Class 为 `com.baidu.openrasp.Agent`  （参考阅读1）

查看 Agent 关键代码，premain 为程序入口：

```java
    /**
     * 启动时加载的agent入口方法
     *
     * @param agentArg 启动参数
     * @param inst     {@link Instrumentation}
     */
    public static void premain(String agentArg, Instrumentation inst) {
        init(START_MODE_NORMAL, START_ACTION_INSTALL, inst);
    }
```

此处调用了 init 方法：

```java
 /**
     * attack 机制加载 agent
     *
     * @param mode 启动模式
     * @param inst {@link Instrumentation}
     */
    public static synchronized void init(String mode, String action, Instrumentation inst) {
        try {
            JarFileHelper.addJarToBootstrap(inst);
            readVersion();
            ModuleLoader.load(mode, action, inst);
        } catch (Throwable e) {
            System.err.println("[OpenRASP] Failed to initialize, will continue without security protection.");
            e.printStackTrace();
        }
    }
```

此处加载了 jar 包，读取 rasp 版本，并加载 module，跟踪代码发现，其实只有一个 module 需要加载：

```java
    /**
     * 构造所有模块
     *
     * @param mode 启动模式
     * @param inst {@link java.lang.instrument.Instrumentation}
     */
    private ModuleLoader(String mode, Instrumentation inst) throws Throwable {

        if (Module.START_MODE_NORMAL == mode) {
            setStartupOptionForJboss();
        }
        engineContainer = new ModuleContainer(ENGINE_JAR);
        engineContainer.start(mode, inst);
    }
```

其中 `ENGINE_JAR` 为 `rasp-engine.jar` 。查看 `java/agent/engine` 项目中的 pom配置：

```xml
 <configuration>
 		<archive>
				<manifestEntries>
 						<Rasp-Module-Name>rasp-engine</Rasp-Module-Name>
 						<Rasp-Module-Class>com.baidu.openrasp.EngineBoot</Rasp-Module-Class>
 				</manifestEntries>
 		</archive>
 		<excludes>
		 		<exclude>**/pluginUnitTest/</exclude>
 		</excludes>
 </configuration>
```

boot 里调用的 module 入口类为 EngineBoot，具体代码：

```java
    @Override
    public void start(String mode, Instrumentation inst) throws Exception {
        System.out.println("\n\n" +
                "   ____                   ____  ___   _____ ____ \n" +
                "  / __ \\____  ___  ____  / __ \\/   | / ___// __ \\\n" +
                " / / / / __ \\/ _ \\/ __ \\/ /_/ / /| | \\__ \\/ /_/ /\n" +
                "/ /_/ / /_/ /  __/ / / / _, _/ ___ |___/ / ____/ \n" +
                "\\____/ .___/\\___/_/ /_/_/ |_/_/  |_/____/_/      \n" +
                "    /_/                                          \n\n");
        try {
            Loader.load();
        } catch (Exception e) {
            System.out.println("[OpenRASP] Failed to load native library, please refer to https://rasp.baidu.com/doc/install/software.html#faq-v8-load for possible solutions.");
            e.printStackTrace();
            return;
        }
        if (!loadConfig()) {
            return;
        }
        //缓存rasp的build信息
        Agent.readVersion();
        BuildRASPModel.initRaspInfo(Agent.projectVersion, Agent.buildTime, Agent.gitCommit);
        // 初始化插件系统
        if (!JS.Initialize()) {
            return;
        }
        CheckerManager.init();
        initTransformer(inst);
        if (CloudUtils.checkCloudControlEnter()) {
            CrashReporter.install(Config.getConfig().getCloudAddress() + "/v1/agent/crash/report",
                    Config.getConfig().getCloudAppId(), Config.getConfig().getCloudAppSecret(),
                    CloudCacheModel.getInstance().getRaspId());
        }
        deleteTmpDir();
        String message = "[OpenRASP] Engine Initialized [" + Agent.projectVersion + " (build: GitCommit="
                + Agent.gitCommit + " date=" + Agent.buildTime + ")]";
        System.out.println(message);
        Logger.getLogger(EngineBoot.class.getName()).info(message);
    }
```

此处打印了 banner 信息，加载 native lib，加载配置，读取版本信息， 初始化插件 等工作。

重点查看 native lib 加载，修改点在 `com.baidu.openrasp.nativelibNativeLibraryUtil.loadNativeLibrary()`



参考阅读：

1. [Javassist](https://www.baeldung.com/javassist) 实现 instrument 实例  https://www.baeldung.com/java-instrumentation 