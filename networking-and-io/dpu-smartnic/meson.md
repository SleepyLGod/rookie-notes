---
description: Meson introduction
---

# Meson

### Install

Meson is a build system implemented in Python 3. The current official documentation requires Python 3.10 or newer. Meson packages in older distribution repositories may require lower Python versions, but new projects should prepare the environment according to the current official requirement. Meson uses Ninja as its default backend.\
On Ubuntu, the simplest installation method is through `pip3`:

```shell
sudo apt-get install python3 python3-pip ninja-build
sudo pip3 install meson
```

You can also install Meson only under the current user's directory:

```shell
pip3 install --user meson
```

This installs Meson under `~/.local/bin`, so this directory needs to be added to `PATH`.

### Compile Configuration

The following is an example:

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

Meson configures the compilation language and files through `meson.build`. `project` specifies the project name and language type, and `executable` specifies the executable name and source files.

`meson builddir` or `meson build` is used to generate the build directory.

`cd builddir` and `ninja` are used to compile the project. Ninja is the backend build tool, roughly similar to `make` in this workflow.

Use `meson configure` to view Meson's built-in options, default values, and possible values:

Projects can add project-specific options through `meson_options.txt`.

```shell
meson configure
...she
Project options:
  Option  Default Value  Possible Values            Description
  gtk_doc auto           [enabled, disabled, auto]  Generate API documentation with gtk-doc
...
```

When generating the build configuration, use `-D` to specify compilation options:

```shell
meson builddir -Dprefix=/usr -Dgtk_doc=disabled -Dtests=disabled
cd builddir && ninja -j8
meson install
```

From the source root directory, `configure` can update compilation options, and then `ninja` can rebuild:

```shell
1 $ meson configure builddir -Dprefix=/home/dev/tmp
```

### Reference

[https://mesonbuild.com/Reference-manual.html](https://mesonbuild.com/Reference-manual.html)
