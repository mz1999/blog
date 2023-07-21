# GlassFish Startup Process

[asadmin](https://glassfish.org/docs/latest/reference-manual.html#asadmin) is a command-line tool for  [GlassFish](https://glassfish.org/) , which provides a series of subcommands. Using `asadmin`, you can complete all management tasks of `GlassFish`.

The subcommand [start-domain](https://glassfish.org/docs/latest/reference-manual.html#start-domain) of `asadmin` can start `GlassFish`. The following will describe the main process of `GlassFish` startup, starting from the execution of the `asadmin` command.

## asadmin Execution Process

The entry point of the `asadmin` command is  [org.glassfish.admin.cli.AsadminMain](https://github.com/eclipse-ee4j/glassfish/blob/master/appserver/admin/cli/src/main/java/org/glassfish/admin/cli/AsadminMain.java), which is included in the `${AS_INSTALL_LIB}/client/appserver-cli.jar` package.

The main process of `AsadminMain` execution is as follows:

![AsadminMain](https://raw.githubusercontent.com/mz1999/material/master/images/202307211620151.png)

Some key points:

* The `CLICommand.getCommand()` is called to obtain the subcommand to start the GlassFish. All subcommands of `asadmin` inherit from [com.sun.enterprise.admin.cli.CLICommand](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/admin/cli/src/main/java/com/sun/enterprise/admin/cli/CLICommand.java) and are loaded from the following directories or Jars:

  * ${com.sun.aas.installRoot}/lib/asadmin

  * ${com.sun.aas.installRoot}/modules/admin-cli.jar

* All subcommands are executed by calling `CLICommand.execute(String... argv)`.
* The implementation class of the subcommand to start `GlassFish` is [StartDomainCommand](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/admin/server-mgmt/src/main/java/com/sun/enterprise/admin/servermgmt/cli/StartDomainCommand.java), which internally calls `GFLauncher.launch()` to start the `GlassFish`.
* Finally, [GFLauncher](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/admin/launcher/src/main/java/com/sun/enterprise/admin/launcher/GFLauncher.java) uses [ProcessBuilder](https://docs.oracle.com/javase/8/docs/api/index.html?java/lang/ProcessBuilder.html)  to start a new process, which is the main process of `GlassFish`. The entry point of this new process is [com.sun.enterprise.glassfish.bootstrap.ASMain](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/bootstrap/src/main/java/com/sun/enterprise/glassfish/bootstrap/ASMain.java).

In addition, if the `verbose` or `watchdog`  is set, the parent process `asadmin` will not exit and will wait until `GlassFish` runs to the end:

```java
// If verbose, hang around until the domain stops
if (getInfo().isVerboseOrWatchdog()) {
    wait(glassFishProcess);
}
```

Next, we analyze the startup process of the `GlassFish` main process.

## Main Process Startup Process

The entry point of the `GlassFish` main process is the `main` method of  [com.sun.enterprise.glassfish.bootstrap.ASMain](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/bootstrap/src/main/java/com/sun/enterprise/glassfish/bootstrap/ASMain.java) , and the main process of the startup process is as follows:

![glassfish startup](https://raw.githubusercontent.com/mz1999/material/master/images/202307211631595.png)

The startup process is complicated, but the main steps are clear:

1. Create [GlassFishRuntime](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/common/simple-glassfish-api/src/main/java/org/glassfish/embeddable/GlassFishRuntime.java) using [RuntimeBuilder](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/common/simple-glassfish-api/src/main/java/org/glassfish/embeddable/spi/RuntimeBuilder.java)
1. Create a  [GlassFish](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/common/simple-glassfish-api/src/main/java/org/glassfish/embeddable/GlassFish.java) instance using [GlassFishRuntime](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/common/simple-glassfish-api/src/main/java/org/glassfish/embeddable/GlassFishRuntime.java)
1. Start the `Glassfish` instance by calling `GlassFish.start()`

### Creating GlassFishRuntime

The main steps to create `GlassFishRuntime` include:

1. Create [RuntimeBuilder](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/common/simple-glassfish-api/src/main/java/org/glassfish/embeddable/spi/RuntimeBuilder.java)
2. Create and initialize the OSGi [Framework](https://github.com/osgi/osgi/blob/main/org.osgi.framework/src/org/osgi/framework/launch/Framework.java)
3. Load OSGi bundles
4. Start the OSGi [Framework](https://github.com/osgi/osgi/blob/main/org.osgi.framework/src/org/osgi/framework/launch/Framework.java)
5. Start the [BundleActivator](https://github.com/osgi/osgi/blob/main/org.osgi.framework/src/org/osgi/framework/BundleActivator.java) in the bundles
6. During the startup process of [HK2Main](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/osgi/adapter/src/main/java/org/jvnet/hk2/osgiadapter/HK2Main.java), it searches for and registers HK2 modules.

During the creation of `GlassFishRuntime`, [OSGiGlassFishRuntimeBuilder](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/bootstrap/src/main/java/com/sun/enterprise/glassfish/bootstrap/osgi/OSGiGlassFishRuntimeBuilder.java) will create and initialize the OSGi Framework, and then use the [installBundles()](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/bootstrap/src/main/java/com/sun/enterprise/glassfish/bootstrap/osgi/BundleProvisioner.java#L169) method of [BundleProvisioner](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/bootstrap/src/main/java/com/sun/enterprise/glassfish/bootstrap/osgi/BundleProvisioner.java) to install all bundles of GlassFish to OSGi.

Where does `BundleProvisioner` find the bundles to load? The `glassfish.osgi.auto.install` property in the `${com.sun.aas.installRoot}/config/osgi.properties` file defines the loading path of OSGi bundles. The `discoverJars()` method of `BundleProvisioner` will scan these paths and discover the Jar packages that need to be loaded.

After completing the loading of bundles, `OSGiGlassFishRuntimeBuilder` will call `Framework.start()` to start the OSGi Framework. During the startup process of the OSGi Framework, the  [BundleActivator](https://github.com/osgi/osgi/blob/main/org.osgi.framework/src/org/osgi/framework/BundleActivator.java)  in the bundles will be started. Two important `BundleActivators` are:

* [GlassFishMainActivator](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/bootstrap/src/main/java/com/sun/enterprise/glassfish/bootstrap/osgi/GlassFishMainActivator.java)
* [HK2Main](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/osgi/adapter/src/main/java/org/jvnet/hk2/osgiadapter/HK2Main.java)

During the startup process of [GlassFishMainActivator](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/bootstrap/src/main/java/com/sun/enterprise/glassfish/bootstrap/osgi/GlassFishMainActivator.java) , [EmbeddedOSGiGlassFishRuntime](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/bootstrap/src/main/java/com/sun/enterprise/glassfish/bootstrap/osgi/EmbeddedOSGiGlassFishRuntime.java) will be registered in OSGi.

[HK2Main](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/osgi/adapter/src/main/java/org/jvnet/hk2/osgiadapter/HK2Main.java) will create [ModulesRegistry](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/hk2-core/src/main/java/com/sun/enterprise/module/ModulesRegistry.java). [ModulesRegistry](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/hk2-core/src/main/java/com/sun/enterprise/module/ModulesRegistry.java) is a key component of HK2. All modules in HK2 are registered here. In the OSGi environment, the specific implementation class of `ModulesRegistry` is  [OSGiModulesRegistryImpl](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/osgi/adapter/src/main/java/org/jvnet/hk2/osgiadapter/OSGiModulesRegistryImpl.java), which will find and register the HK2 modules contained in the `META-INF/hk2-locator` directory of all bundle Jars.

`ModulesRegistry` and `HK2Main` will be registered as OSGi's service.

### Creating GlassFish Instance

Create a GlassFish instance through `GlassFishRuntime.newGlassFish()`. This process mainly does two things:

1. Create HK2's [ServiceLocator](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/hk2-api/src/main/java/org/glassfish/hk2/api/ServiceLocator.java)
2. Get [ModuleStartup](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/hk2-core/src/main/java/com/sun/enterprise/module/bootstrap/ModuleStartup.java) from [ServiceLocator](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/hk2-api/src/main/java/org/glassfish/hk2/api/ServiceLocator.java)

In `EmbeddedOSGiGlassFishRuntime`, use  [ModulesRegistry.newServiceLocator()](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/hk2-core/src/main/java/com/sun/enterprise/module/ModulesRegistry.java#L44) to create `ServiceLocator`, and then get [ModuleStartup](https://github.com/eclipse-ee4j/glassfish-hk2/blob/master/hk2-core/src/main/java/com/sun/enterprise/module/bootstrap/ModuleStartup.java) from `ServiceLocator`. In the GlassFish startup scenario, the specific implementation of `ModuleStartup` is [AppServerStartup](https://github.com/eclipse-ee4j/glassfish/blob/master/nucleus/core/kernel/src/main/java/com/sun/enterprise/v3/server/AppServerStartup.java).

`ServiceLocator` is the registry of HK2 services, which provides a series of methods to get HK2 service.

The relationship between HK2 `Module` and `Service` can be regarded as the relationship between container and content. `Module` (container) contains a group of `Service` (content) and is responsible for registering these Services in `ServiceLocator`. When a `Module` is initialized, all its Services will be registered in `ServiceLocator`, and then these Services can be found and used by other Services.

Finally, create a GlassFish instance with `AppServerStartup` and `ServiceLocator` as the parameters of the constructor.

### Starting GlassFish Instance

Use `GlassFish.start()` to start the `Glassfish` instance. The most critical step is to call `AppServerStartup.start()`, which starts the HK2 service in stages. The service of HK2 can specify the startup level, the lower the level, the earlier the startup.

After `AppServerStartup.start()` runs, all services start, and Glassfish completes startup and runs.
