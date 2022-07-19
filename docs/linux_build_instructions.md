# Checking out and building Thorium on Linux <img src="https://github.com/Alex313031/Thorium/blob/main/logos/NEW/build_light.svg#gh-dark-mode-only" width="48"> <img src="https://github.com/Alex313031/Thorium/blob/main/logos/NEW/build_dark.svg#gh-light-mode-only" width="48">

There are instructions for other platforms here in the Thorium Docs directory.

## System requirements

*   A x64 machine with at least 8GB of RAM. 16GB or more is highly
    recommended.
*   At least 75GB of free disk space.
*   You must have Git and Python v3.6+ installed already (and `python3` must point
    to a Python v3.6+ binary (i.e. in your path or as default python install).

Most development is done on Ubuntu (currently 18.04, Bionic Beaver). Ubuntu 16.04 no longer works. 20.04 and Debian 11 will work.
There are some instructions for other distros below, but they are mostly unsupported.

__The scripts to build thorium assume that depot_tools and chromium are both in $HOME!__

## Install *depot_tools*

Clone the `depot_tools` repository:

```shell
$ git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```

Add *depot_tools* to the end of your *$PATH* (you will probably want to put this
in your `~/.bashrc` or `~/.zshrc`). When cloning *depot_tools* to your home directory **do not** use `~` on PATH,
otherwise `gclient runhooks` will fail to run. Rather, you should use either
`$HOME` or the absolute path. So, assuming you cloned *depot_tools* to *$HOME*:

```shell
$ export PATH="$PATH:${HOME}/depot_tools" or $ export PATH="$PATH:/home/alex/depot_tools"
```

## Get the code

Create a *chromium* directory for the checkout and change to it.

```shell
$ mkdir ~/chromium && cd ~/chromium
```

Run the *fetch* tool from depot_tools to check out the code and its
dependencies.

```shell
$ fetch --nohooks chromium
```

The `--nohooks` flag is ommitted on other platforms, we just use it on linux to explicitly run the hooks
later, after installing the prerequisites.
`fetch` and `repo` are used to download, rebase, and sync all Google repositories, including Chromium, ChromiumOS, 
Android, Fuchsia, Infra, Monorail, GN, etc.

If you don't want the full repo history, you can save a lot of time by
adding the `--no-history` flag to `fetch`. This is equivalent to a shallow git clone with a depth of 1.

Expect the command to take 20 minutes on a fast (150mbps+) connection, and many
hours on slower ones.

If you've already installed the build dependencies on the machine (from another
checkout, for example), you can omit the `--nohooks` flag and *fetch*
will automatically execute `gclient runhooks` at the end.

When *fetch* completes, it will have created a hidden `.gclient` file and a
directory called `src` in the *chromium* directory. The remaining instructions
assume you have switched to the `src` directory, so:

```shell
$ cd src
```

### Install additional build dependencies

Once you have checked out the code, and assuming you're using Ubuntu, run the
[*`install-build-deps.sh`*](https://chromium.googlesource.com/chromium/src/+/main/build/install-build-deps.sh) script.

```shell
$ ./build/install-build-deps.sh
```

You can run it with the flag `--help` to see arguments. For example, you would want `--lib32` if building for 32 bit Linux, `--arm` for building
a Raspberry Pi release, `--chromeos-fonts` for building Thorium for ThoriumOS, and `--quick-check` just to verify needed libraries are installed.

You may need to adjust the build dependencies for other distros. There are
some [notes](#notes) at the end of this document, but we make no guarantees
for their accuracy, as distros get updated over time.

### Run the hooks

Once you've run `install-build-deps` at least once, you can now run the
Chromium-specific hooks, which will download additional binaries and other
things like LLVM and a Debian Sysroot.:

```shell
$ gclient runhooks
```

*Optional*: You can also [build with API
keys](https://www.chromium.org/developers/how-tos/api-keys) if you want your
build to talk to some Google services like Google Sync, Translate, and GeoLocation. Thorium has its own keys
in a private repository,  if you are a builder or would like access to them, contact me. Otherwise, for personal or development builds, 
you can create your own keys and add yourself to [google-browser-signin-testaccounts](https://groups.google.com/u/1/a/chromium.org/g/google-browser-signin-testaccounts)
to enable Sync.

## Setting up the build

Chromium and Thorium use [Ninja](https://ninja-build.org) as their main build tool, along with
a tool called [GN](https://gn.googlesource.com/gn/+/refs/heads/main/README.md)
to generate `.ninja` files in the build output directory. You can create any number of *build directories*
with different configurations. To create a build directory, run:

```shell
$ gn args out/thorium
```

You can list all the possible build arguments and pipe it to a text file by running:

```shell
$ gn args out/thorium --list >> /path/to/ARGS.list
```

* You only have to run this once for each new build directory, Ninja will
  update the build files as needed.
* You can replace `thorium` with another name, but
  it should be a subdirectory of `out`. Note that if you choose another name, the `trunk.sh` and `build.sh` scripts will not work.
* For information on the args.gn that Thorium uses, see [ABOUT_GN_ARGS.md](https://github.com/Alex313031/Thorium/blob/main/docs/ABOUT_GN_ARGS.md).  
* For other build arguments, including release settings, see [GN build
  configuration](https://www.chromium.org/developers/gn-build-configuration).
  The default will be a vanilla Chromium debug component build matching the current host
  operating system and CPU.
* For more info on GN, run `gn help` on the command line or read the
  [quick start guide](https://gn.googlesource.com/gn/+/main/docs/quick_start.md).

#### ccache

You can use [ccache](https://ccache.dev) to speed up local builds.

Increase your ccache hit rate by setting `CCACHE_BASEDIR` to a parent directory
that the working directories all have in common (e.g.,
`/home/yourusername/development`). Consider using
`CCACHE_SLOPPINESS=include_file_mtime` (since if you are using multiple working
directories, header times in svn sync'ed portions of your trees will be
different - see
[the ccache troubleshooting section](https://ccache.dev/manual/latest.html#_troubleshooting)
for additional information). If you use symbolic links from your home directory
to get to the local physical disk directory where you keep those working
development directories, consider putting

    alias cd="cd -P"

in your `.bashrc` so that `$PWD` or `cwd` always refers to a physical, not
logical directory (and make sure `CCACHE_BASEDIR` also refers to a physical
parent).

If you tune ccache correctly, a second working directory that uses a branch
tracking trunk and is up to date with trunk and was gclient sync'ed at about the
same time should build chrome in about 1/3 the time, and the cache misses as
reported by `ccache -s` should barely increase.

This is especially useful if you use
[git-worktree](http://git-scm.com/docs/git-worktree) and keep multiple local
working directories going at once.

## Build Thorium

Build Thorium (the "chrome" target), as well as [chrome_sandbox](https://chromium.googlesource.com/chromium/src/+/HEAD/docs/linux/sandboxing.md), [chromedriver](https://chromedriver.chromium.org/home), and [thorium_shell](https://github.com/Alex313031/Thorium/tree/main/thorium_shell#readme) (based on [content_shell](https://chromium.googlesource.com/chromium/src/+/HEAD/docs/testing/web_tests_in_content_shell.md#as-a-simple-browser) ), using the `build.sh`
in the root of the Thorium repo (where the # is the number of jobs):

```shell
$ ./build.sh 8
```

You could also manually issue the command (where -j is number of jobs):

```shell
$ autoninja -C ~/chromium/src/out/thorium chrome chrome_sandbox chromedriver thorium_shell -j8
```

`autoninja` is a wrapper that automatically provides optimal values for the
arguments passed to `ninja`. *build.sh* uses a custom autoninja in the *depot_tools* directory in Thorium.

You can get a list of all of the other build targets from GN by running `gn ls
out/thorium` from the command line. To compile one, pass the GN label to Ninja
with no preceding "//" (so, for `//chrome/test:unit_tests` use `autoninja -C
out/thorium chrome/test:unit_tests`).

## Run Chromium

Once it is built, you can simply run the browser:

```shell
$ out/Default/chrome
```

## Running test targets

First build the unit_tests binary by running the command:

```shell
$ autoninja -C out/Default unit_tests
```

You can run the tests by running the unit_tests binary. You can also limit which
tests are run using the `--gtest_filter` arg, e.g.:

```shell
$ out/Default/unit_tests --gtest_filter="PushClientTest.*"
```

You can find out more about GoogleTest at its
[GitHub page](https://github.com/google/googletest).

## Update your checkout

To update an existing checkout, you can run

```shell
$ git rebase-update
$ gclient sync
```

The first command updates the primary Chromium source repository and rebases
any of your local branches on top of tip-of-tree (aka the Git branch
`origin/main`). If you don't want to use this script, you can also just use
`git pull` or other common Git commands to update the repo.

The second command syncs dependencies to the appropriate versions and re-runs
hooks as needed.

## Tips, tricks, and troubleshooting

### Linker Crashes

If, during the final link stage:

```
LINK out/Debug/chrome
```

You get an error like:

```
collect2: ld terminated with signal 6 Aborted terminate called after throwing an instance of 'std::bad_alloc'
collect2: ld terminated with signal 11 [Segmentation fault], core dumped
```

you are probably running out of memory when linking. You *must* use a 64-bit
system to build. Try the following build settings (see [GN build
configuration](https://www.chromium.org/developers/gn-build-configuration) for
other settings):

*   Build in release mode (debugging symbols require more memory):
    `is_debug = false`
*   Turn off symbols: `symbol_level = 0`
*   Build in component mode (this is for development only, it will be slower and
    may have broken functionality): `is_component_build = true`

### More links

*   Information about [building with Clang](../clang.md).
*   You may want to [use a chroot](using_a_chroot.md) to
    isolate yourself from versioning or packaging conflicts.
*   Cross-compiling for ARM? See [LinuxChromiumArm](chromium_arm.md).
*   Want to use Eclipse as your IDE? See
    [LinuxEclipseDev](eclipse_dev.md).
*   Want to use your built version as your default browser? See
    [LinuxDevBuildAsDefaultBrowser](dev_build_as_default_browser.md).

## Next Steps

If you want to contribute to the effort toward a Chromium-based browser for
Linux, please check out the [Linux Development page](development.md) for
more information.

## Notes for other distros <a name="notes"></a>

### Arch Linux

Instead of running `install-build-deps.sh` to install build dependencies, run:

```shell
$ sudo pacman -S --needed python perl gcc gcc-libs bison flex gperf pkgconfig \
nss alsa-lib glib2 gtk3 nspr freetype2 cairo dbus libgnome-keyring \
xorg-server-xvfb xorg-xdpyinfo
```

For the optional packages on Arch Linux:

*   `php-cgi` is provided with `pacman`
*   `wdiff` is not in the main repository but `dwdiff` is. You can get `wdiff`
    in AUR/`yaourt`

### Crostini (Debian based)

First install the `file` and `lsb-release` commands for the script to run properly:

```shell
$ sudo apt-get install file lsb-release
```

Then invoke install-build-deps.sh with the `--no-arm` argument,
because the ARM toolchain doesn't exist for this configuration:

```shell
$ sudo install-build-deps.sh --no-arm
```

### Fedora

Instead of running `build/install-build-deps.sh`, run:

```shell
su -c 'yum install git python bzip2 tar pkgconfig atk-devel alsa-lib-devel \
bison binutils brlapi-devel bluez-libs-devel bzip2-devel cairo-devel \
cups-devel dbus-devel dbus-glib-devel expat-devel fontconfig-devel \
freetype-devel gcc-c++ glib2-devel glibc.i686 gperf glib2-devel \
gtk3-devel java-1.*.0-openjdk-devel libatomic libcap-devel libffi-devel \
libgcc.i686 libgnome-keyring-devel libjpeg-devel libstdc++.i686 libX11-devel \
libXScrnSaver-devel libXtst-devel libxkbcommon-x11-devel ncurses-compat-libs \
nspr-devel nss-devel pam-devel pango-devel pciutils-devel \
pulseaudio-libs-devel zlib.i686 httpd mod_ssl php php-cli python-psutil wdiff \
xorg-x11-server-Xvfb'
```

The fonts needed by Blink's web tests can be obtained by following [these
instructions](https://gist.github.com/pwnall/32a3b11c2b10f6ae5c6a6de66c1e12ae).
For the optional packages:

* `php-cgi` is provided by the `php-cli` package.
* `sun-java6-fonts` is covered by the instructions linked above.

### Gentoo

You can just run `emerge www-client/chromium`.

### OpenSUSE

Use `zypper` command to install dependencies:

(openSUSE 11.1 and higher)

```shell
sudo zypper in subversion pkg-config python perl bison flex gperf \
     mozilla-nss-devel glib2-devel gtk-devel wdiff lighttpd gcc gcc-c++ \
     mozilla-nspr mozilla-nspr-devel php5-fastcgi alsa-devel libexpat-devel \
     libjpeg-devel libbz2-devel
```

For 11.0, use `libnspr4-0d` and `libnspr4-dev` instead of `mozilla-nspr` and
`mozilla-nspr-devel`, and use `php5-cgi` instead of `php5-fastcgi`.

(openSUSE 11.0)

```shell
sudo zypper in subversion pkg-config python perl \
     bison flex gperf mozilla-nss-devel glib2-devel gtk-devel \
     libnspr4-0d libnspr4-dev wdiff lighttpd gcc gcc-c++ libexpat-devel \
     php5-cgi alsa-devel gtk3-devel jpeg-devel
```

The Ubuntu package `sun-java6-fonts` contains a subset of Java of the fonts used.
Since this package requires Java as a prerequisite anyway, we can do the same
thing by just installing the equivalent openSUSE Sun Java package:

```shell
sudo zypper in java-1_6_0-sun
```

WebKit is currently hard-linked to the Microsoft fonts. To install these using `zypper`

```shell
sudo zypper in fetchmsttfonts pullin-msttf-fonts
```

To make the fonts installed above work, as the paths are hardcoded for Ubuntu,
create symlinks to the appropriate locations:

```shell
sudo mkdir -p /usr/share/fonts/truetype/msttcorefonts
sudo ln -s /usr/share/fonts/truetype/arial.ttf /usr/share/fonts/truetype/msttcorefonts/Arial.ttf
sudo ln -s /usr/share/fonts/truetype/arialbd.ttf /usr/share/fonts/truetype/msttcorefonts/Arial_Bold.ttf
sudo ln -s /usr/share/fonts/truetype/arialbi.ttf /usr/share/fonts/truetype/msttcorefonts/Arial_Bold_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/ariali.ttf /usr/share/fonts/truetype/msttcorefonts/Arial_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/comic.ttf /usr/share/fonts/truetype/msttcorefonts/Comic_Sans_MS.ttf
sudo ln -s /usr/share/fonts/truetype/comicbd.ttf /usr/share/fonts/truetype/msttcorefonts/Comic_Sans_MS_Bold.ttf
sudo ln -s /usr/share/fonts/truetype/cour.ttf /usr/share/fonts/truetype/msttcorefonts/Courier_New.ttf
sudo ln -s /usr/share/fonts/truetype/courbd.ttf /usr/share/fonts/truetype/msttcorefonts/Courier_New_Bold.ttf
sudo ln -s /usr/share/fonts/truetype/courbi.ttf /usr/share/fonts/truetype/msttcorefonts/Courier_New_Bold_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/couri.ttf /usr/share/fonts/truetype/msttcorefonts/Courier_New_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/impact.ttf /usr/share/fonts/truetype/msttcorefonts/Impact.ttf
sudo ln -s /usr/share/fonts/truetype/times.ttf /usr/share/fonts/truetype/msttcorefonts/Times_New_Roman.ttf
sudo ln -s /usr/share/fonts/truetype/timesbd.ttf /usr/share/fonts/truetype/msttcorefonts/Times_New_Roman_Bold.ttf
sudo ln -s /usr/share/fonts/truetype/timesbi.ttf /usr/share/fonts/truetype/msttcorefonts/Times_New_Roman_Bold_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/timesi.ttf /usr/share/fonts/truetype/msttcorefonts/Times_New_Roman_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/verdana.ttf /usr/share/fonts/truetype/msttcorefonts/Verdana.ttf
sudo ln -s /usr/share/fonts/truetype/verdanab.ttf /usr/share/fonts/truetype/msttcorefonts/Verdana_Bold.ttf
sudo ln -s /usr/share/fonts/truetype/verdanai.ttf /usr/share/fonts/truetype/msttcorefonts/Verdana_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/verdanaz.ttf /usr/share/fonts/truetype/msttcorefonts/Verdana_Bold_Italic.ttf
```

The Ubuntu package `sun-java6-fonts` contains a subset of Java of the fonts used.
Since this package requires Java as a prerequisite anyway, we can do the same
thing by just installing the equivalent openSUSE Sun Java package:

```shell
sudo zypper in java-1_6_0-sun
```

WebKit is currently hard-linked to the Microsoft fonts. To install these using `zypper`

```shell
sudo zypper in fetchmsttfonts pullin-msttf-fonts
```

To make the fonts installed above work, as the paths are hardcoded for Ubuntu,
create symlinks to the appropriate locations:

```shell
sudo mkdir -p /usr/share/fonts/truetype/msttcorefonts
sudo ln -s /usr/share/fonts/truetype/arial.ttf /usr/share/fonts/truetype/msttcorefonts/Arial.ttf
sudo ln -s /usr/share/fonts/truetype/arialbd.ttf /usr/share/fonts/truetype/msttcorefonts/Arial_Bold.ttf
sudo ln -s /usr/share/fonts/truetype/arialbi.ttf /usr/share/fonts/truetype/msttcorefonts/Arial_Bold_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/ariali.ttf /usr/share/fonts/truetype/msttcorefonts/Arial_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/comic.ttf /usr/share/fonts/truetype/msttcorefonts/Comic_Sans_MS.ttf
sudo ln -s /usr/share/fonts/truetype/comicbd.ttf /usr/share/fonts/truetype/msttcorefonts/Comic_Sans_MS_Bold.ttf
sudo ln -s /usr/share/fonts/truetype/cour.ttf /usr/share/fonts/truetype/msttcorefonts/Courier_New.ttf
sudo ln -s /usr/share/fonts/truetype/courbd.ttf /usr/share/fonts/truetype/msttcorefonts/Courier_New_Bold.ttf
sudo ln -s /usr/share/fonts/truetype/courbi.ttf /usr/share/fonts/truetype/msttcorefonts/Courier_New_Bold_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/couri.ttf /usr/share/fonts/truetype/msttcorefonts/Courier_New_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/impact.ttf /usr/share/fonts/truetype/msttcorefonts/Impact.ttf
sudo ln -s /usr/share/fonts/truetype/times.ttf /usr/share/fonts/truetype/msttcorefonts/Times_New_Roman.ttf
sudo ln -s /usr/share/fonts/truetype/timesbd.ttf /usr/share/fonts/truetype/msttcorefonts/Times_New_Roman_Bold.ttf
sudo ln -s /usr/share/fonts/truetype/timesbi.ttf /usr/share/fonts/truetype/msttcorefonts/Times_New_Roman_Bold_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/timesi.ttf /usr/share/fonts/truetype/msttcorefonts/Times_New_Roman_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/verdana.ttf /usr/share/fonts/truetype/msttcorefonts/Verdana.ttf
sudo ln -s /usr/share/fonts/truetype/verdanab.ttf /usr/share/fonts/truetype/msttcorefonts/Verdana_Bold.ttf
sudo ln -s /usr/share/fonts/truetype/verdanai.ttf /usr/share/fonts/truetype/msttcorefonts/Verdana_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/verdanaz.ttf /usr/share/fonts/truetype/msttcorefonts/Verdana_Bold_Italic.ttf
```

And then for the Java fonts:

```shell
sudo mkdir -p /usr/share/fonts/truetype/ttf-lucida
sudo find /usr/lib*/jvm/java-1.6.*-sun-*/jre/lib -iname '*.ttf' -print \
     -exec ln -s {} /usr/share/fonts/truetype/ttf-lucida \;
```