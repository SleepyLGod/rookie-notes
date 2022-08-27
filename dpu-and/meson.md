---
description: meson 入门介绍
---

# 😁 meson

### Install

Meson是基于python3实现，至少需要python3.5才能运行，默认采用ninja作为后端。\
在Ubuntu下最简单的是通过pip3安装

```shell
sudo apt-get install python3 python3-pip ninja-build
sudo pip3 install meson
```

也可以只将meson安装到当前用户目录下

```shell
pip3 install --user meson
```

这种方式会将meson安装到`~/.local/bin`目录下，因此需要将这个目录增加到PATH中。

### Compile Configuration

下面是个例子：

```shell
$ cat > meson.build << EOF
> project('mesontest', 'c')
> executable('mesontest', test.c)
> EOF

$ meson builddir && cd builddir
$ ninja
$ ./mesontest
hello meson.
```

meson通过`meson.build`文件配置编译语言及文件，`project`指定项目名称及语言类型，`executable`指定可执行文件名及源文件。

`meson builddir (meson build)` 用于执行构建；

`cd builddir` 和 `ninja` 用于执行编译，其中 ninja 是后端编译器，相当于 make。

下面用 `meson configure` 查看meson内置的选项、默认值及可选值：

项目可以通过 `meson_options.txt` 来增加项目特有的选项。

```shell
meson configure
...she
Project options:
  Option  Default Value  Possible Values            Description
  gtk_doc auto           [enabled, disabled, auto]  Generate API documentation with gtk-doc
...
```

在生成编译配置时，可以通过 -D 指定编译选项：

```shell
meson builddir -Dprefix=/usr -Dgtk_doc=disabled -Dtests=disabled
cd builddir && ninja -j8
meson install
```

可以在源码根目录通过 configure更新编译选项，再执行ninja重新编译：

```shell
1 $ meson configure builddir -Dprefix=/home/dev/tmp
```

### Referance

[https://mesonbuild.com/Reference-manual.html](https://mesonbuild.com/Reference-manual.html)









