# GlassFish 启动流程

[asadmin](https://glassfish.org/docs/latest/reference-manual.html#asadmin) 是 [GlassFish](https://glassfish.org/) 的命令行工具，它提供了一系列子命令，使用 `asadmin` 可以让你完成 `Glassfish` 的所有管理任务。

使用 `asadmin` 的子命令 [start-domain](https://glassfish.org/docs/latest/reference-manual.html#start-domain) 可以启动 `GlassFish`。下面将描述 `GlassFish`启动过程的主要流程。先从 `asadmin` 命令的执行开始。

## asadmin 执行流程

`asadmin` 命令的入口是  [org.glassfish.admin.cli.AsadminMain](https://github.com/eclipse-ee4j/glassfish/blob/master/appserver/admin/cli/src/main/java/org/glassfish/admin/cli/AsadminMain.java)， 包含在 `${AS_INSTALL_LIB}/client/appserver-cli.jar`包中。

`AsadminMain` 执行的主要流程如下：

![AsadminMain](https://cdn.mazhen.tech/images/202307191450899.png)

其中的一些关键点：

* 调用`CLICommand.getCommand()`获得启动服务器的子命令。`asadmin` 的所有子命令的都继承自[com.sun.enterprise.admin.cli.CLICommand](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/admin/cli/src/main/java/com/sun/enterprise/admin/cli/CLICommand.java) ，从下列目录或 Jar 中加载：

  * ${com.sun.aas.installRoot}/lib/asadmin

  * ${com.sun.aas.installRoot}/modules/admin-cli.jar

* 所有子命令的执行都是调用`CLICommand.execute(String... argv)` 。

* 启动 `GlassFish` 的子命令实现类为[StartDomainCommand](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/admin/server-mgmt/src/main/java/com/sun/enterprise/admin/servermgmt/cli/StartDomainCommand.java)，内部调用 `GFLauncher.launch()`启动服务器。

* 最终 [GFLauncher](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/admin/launcher/src/main/java/com/sun/enterprise/admin/launcher/GFLauncher.java) 使用 [ProcessBuilder](https://docs.oracle.com/javase/8/docs/api/index.html?java/lang/ProcessBuilder.html) 启动一个新的进程，就是 GlassFish 的主进程。这个新进程的入口是[com.sun.enterprise.glassfish.bootstrap.ASMain](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/bootstrap/src/main/java/com/sun/enterprise/glassfish/bootstrap/ASMain.java)。

另外，如果设置了 `verbose` 或 `watchdog` 参数，作为父进程的`asadmin` 不会退出，一直等到 GlassFish 运行结束：

```java
// If verbose, hang around until the domain stops
if (getInfo().isVerboseOrWatchdog()) {
    wait(glassFishProcess);
}
```

下面分析 `GlassFish` 主进程的启动流程。

## 主进程启动流程

`GlassFish` 主进程的入口是 [com.sun.enterprise.glassfish.bootstrap.ASMain](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/bootstrap/src/main/java/com/sun/enterprise/glassfish/bootstrap/ASMain.java) 的 `main`方法，启动过程的主要流程如下：

![glassfish startup](https://cdn.mazhen.tech/images/202307210950607.png)

启动过程比较复杂，但主要步骤很清晰：

1. 使用 [RuntimeBuilder](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/common/simple-glassfish-api/src/main/java/org/glassfish/embeddable/spi/RuntimeBuilder.java) 创建 [GlassFishRuntime](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/common/simple-glassfish-api/src/main/java/org/glassfish/embeddable/GlassFishRuntime.java)
1. 使用 [GlassFishRuntime](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/common/simple-glassfish-api/src/main/java/org/glassfish/embeddable/GlassFishRuntime.java) 创建 [GlassFish](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/common/simple-glassfish-api/src/main/java/org/glassfish/embeddable/GlassFish.java) 实例
1. 调用 `GlassFish.start()` 启动 `Glassfish` 实例

### 创建 GlassFishRuntime

创建 `GlassFishRuntime` 的主要步骤包括：

1. 创建 [RuntimeBuilder](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/common/simple-glassfish-api/src/main/java/org/glassfish/embeddable/spi/RuntimeBuilder.java)
2. 创建并初始化  OSGi [Framework](https://github.com/osgi/osgi/blob/main/org.osgi.framework/src/org/osgi/framework/launch/Framework.java)
3. 加载 OSGi bundles
4. 启动 OSGi [Framework](https://github.com/osgi/osgi/blob/main/org.osgi.framework/src/org/osgi/framework/launch/Framework.java)
5. 启动 bundles 中的 [BundleActivator](https://github.com/osgi/osgi/blob/main/org.osgi.framework/src/org/osgi/framework/BundleActivator.java)
6. 在 [HK2Main](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/osgi/adapter/src/main/java/org/jvnet/hk2/osgiadapter/HK2Main.java) 的启动过程中查找并注册 HK2 modules

在创建 `GlassFishRuntime`的过程中，[OSGiGlassFishRuntimeBuilder](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/bootstrap/src/main/java/com/sun/enterprise/glassfish/bootstrap/osgi/OSGiGlassFishRuntimeBuilder.java) 会创建并初始化 OSGi [Framework](https://github.com/osgi/osgi/blob/main/org.osgi.framework/src/org/osgi/framework/launch/Framework.java) ，然后使用 [BundleProvisioner](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/bootstrap/src/main/java/com/sun/enterprise/glassfish/bootstrap/osgi/BundleProvisioner.java) 的 [installBundles()](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/bootstrap/src/main/java/com/sun/enterprise/glassfish/bootstrap/osgi/BundleProvisioner.java#L169) 方法向 OSGi 安装 GlassFish 的所有 bundles。

`BundleProvisioner`从哪里找到要加载的 bundles？`config/osgi.properties` 文件中的 `glassfish.osgi.auto.install` 属性定义了 OSGi bundles 的加载路径。`BundleProvisioner.discoverJars()` 方法会扫描这些路径，发现需要加载的 Jar 包。

在完成 bundles 的加载后，`OSGiGlassFishRuntimeBuilder`会调用 `Framework.start()` 启动 OSGi [Framework](https://github.com/osgi/osgi/blob/main/org.osgi.framework/src/org/osgi/framework/launch/Framework.java)。 Framework 的启动过程中，bundles 中的 [BundleActivator](https://github.com/osgi/osgi/blob/main/org.osgi.framework/src/org/osgi/framework/BundleActivator.java) 会被启动。其中两个重要的 `BundleActivator` 是：

* [GlassFishMainActivator](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/bootstrap/src/main/java/com/sun/enterprise/glassfish/bootstrap/osgi/GlassFishMainActivator.java)
* [HK2Main](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/osgi/adapter/src/main/java/org/jvnet/hk2/osgiadapter/HK2Main.java)

[GlassFishMainActivator](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/bootstrap/src/main/java/com/sun/enterprise/glassfish/bootstrap/osgi/GlassFishMainActivator.java) 启动过程中会向 OSGi 中注册 [EmbeddedOSGiGlassFishRuntime](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/bootstrap/src/main/java/com/sun/enterprise/glassfish/bootstrap/osgi/EmbeddedOSGiGlassFishRuntime.java)。

[HK2Main](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/osgi/adapter/src/main/java/org/jvnet/hk2/osgiadapter/HK2Main.java) 会创建 [ModulesRegistry](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/hk2-core/src/main/java/com/sun/enterprise/module/ModulesRegistry.java)。[ModulesRegistry](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/hk2-core/src/main/java/com/sun/enterprise/module/ModulesRegistry.java) 是 HK2 的关键组件，HK2 中的 modules 都注册在这里。在 OSGi 环境下，ModulesRegistry 的具体实现类是 [OSGiModulesRegistryImpl](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/osgi/adapter/src/main/java/org/jvnet/hk2/osgiadapter/OSGiModulesRegistryImpl.java)，它会从所有 bundle Jar 的 `META-INF/hk2-locator` 目录中查找并注册该 bundle 包含的 HK2 modules。

`ModulesRegistry` 和 `HK2Main` 都会注册为 OSGi 的 service。

### 创建 GlassFish 实例

通过`GlassFishRuntime.newGlassFish()` 创建出 GlassFish 实例，这个过程主要做了两件事：

1. 创建出 HK2 的 [ServiceLocator](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/hk2-api/src/main/java/org/glassfish/hk2/api/ServiceLocator.java)
2. 从 [ServiceLocator](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/hk2-api/src/main/java/org/glassfish/hk2/api/ServiceLocator.java) 获取 [ModuleStartup](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/hk2-core/src/main/java/com/sun/enterprise/module/bootstrap/ModuleStartup.java)

在 `EmbeddedOSGiGlassFishRuntime` 中使用 [ModulesRegistry.newServiceLocator()](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/hk2-core/src/main/java/com/sun/enterprise/module/ModulesRegistry.java#L44) 创建出 `ServiceLocator`，然后从 `ServiceLocator` 获取 [ModuleStartup](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/hk2-core/src/main/java/com/sun/enterprise/module/bootstrap/ModuleStartup.java)。在 GlassFish 启动场景获取的是 `ModuleStartup` 的一个具体实现 [AppServerStartup](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/kernel/src/main/java/com/sun/enterprise/v3/server/AppServerStartup.java)。

`ServiceLocator` 是 HK2 service 的注册表，它提供了一系列获取 HK2 service 的方法。

HK2 Module 和 Service 的关系可以看作是容器和内容的关系。Module（容器）包含了一组 Service（内容），并且负责将这些 Service 注册到 `ServiceLocator` 中。当一个 Module 被初始化时，它的所有 Service 都会被注册到 `ServiceLocator` 中，然后这些 Service 就可以被其他 Service 查找和使用。

最后将 `AppServerStartup` 和 `ServiceLocator` 作为构造函数的参数，创建出 `GlassFish` 实例。

### 启动 Glassfish 实例

使用 `GlassFish.start()` 启动 `Glassfish` 实例。其中最关键的步骤是调用 `AppServerStartup.start()`，分级启动 HK2 的 service。HK2 的 service 可以指定启动级别，级别越低，越先启动。

`AppServerStartup.start()` 运行完成，所有 service 启动，Glassfish 完成启动并运行。
