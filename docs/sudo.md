
# 如何让用户拥有sudo权限

先使用`visudo` 查看当前的配置，这个命令编辑的是`/etc/sudoers`文件。可以直接在这个文件中为用户设置`sudo`权限：

```
# User privilege specification
root    ALL=(ALL:ALL) ALL
adp     ALL=(ALL) ALL
```

也可以看看哪个`group`有`root`权限，然后将用户加入这个`group`。例如下面的配置，`admin`组有root权限：

```
# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL
```

可以将用户加入admin组，自然就有了sudo权限：

```
usermod -a -G admin [user]
```

如果提示`admin`不存在，可以先创建这个组，再将用户加入这个group：

```
groupadd admin
usermod -a -G admin [user]
```

如果不想编辑`/etc/sudoers`，可以在`/etc/sudoers.d/`目录下，为需要`sudo`权限的用户创建独立的文件，在文件中分别为用户授权，格式和`/etc/sudoers`一样：

```
adp ALL=(ALL)  ALL
```

修改文件权限：

```
chmod 440 adp
```

这样做的好处每个用户都有独立的配置文件，是方便管理。

最后，**建议**将`/sbin` 和 `/usr/sbin` 加入到用户路径。

```
PATH=$PATH:/usr/sbin:/sbin
```