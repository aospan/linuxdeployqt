# linuxdeployqt [![Build Status](https://travis-ci.org/probonopd/linuxdeployqt.svg?branch=master)](https://travis-ci.org/probonopd/linuxdeployqt) [![Codacy Badge](https://api.codacy.com/project/badge/Grade/93b4a359057e412b8a7673b4b61d7cb7)](https://www.codacy.com/app/probonopd/linuxdeployqt?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=probonopd/linuxdeployqt&amp;utm_campaign=Badge_Grade) [![discourse](https://img.shields.io/badge/forum-discourse-orange.svg)](http://discourse.appimage.org/t/linuxdeployqt-new-linux-deployment-tool-for-qt/57) [![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/probonopd/AppImageKit?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge) [![irc](https://img.shields.io/badge/IRC-%23AppImage%20on%20freenode-blue.svg)](https://webchat.freenode.net/?channels=AppImage)

This Linux Deployment Tool for Qt, `linuxdeployqt`, takes an application as input and makes it self-contained by copying in the Qt libraries and plugins that the application uses into a bundle. This can optionally be put into an [AppImage](http://appimage.org/), and, using [fpm](https://github.com/probonopd/linuxdeployqt/issues/9), into cross-distro deb and rpm packages.

## Differences to macdeployqt
This tool is conceptually based on the [Mac Deployment Tool](http://doc.qt.io/qt-5/osx-deployment.html), `macdeployqt` in the tools applications of the Qt Toolkit, but has been changed to a slightly different logic and other tools needed for Linux.

* Instead of an `.app` bundle for macOS, this produces an [AppDir](http://rox.sourceforge.net/desktop/AppDirs.html) for Linux
* Instead of a `.dmg` disk image for macOS, this produces an [AppImage](http://appimage.org/) for Linux which is quite similar to a dmg but executes the contained application rather than just opening a window on the desktop from where the application can be launched

## Known issues

__This may not be fully working yet.__ See [GitHub Issues](https://github.com/probonopd/linuxdeployqt/issues) for known issues. Use with care, run with maximum verbosity, submit issues and pull requests. Help is appreciated.

## Installation

Please download __linuxdeployqt-x86_64.AppImage__ from the [Releases](https://github.com/probonopd/linuxdeployqt/releases) page and `chmod a+x` it. If you would like to build `linuxdeployqt` from source instead, see [BUILDING.md](https://github.com/probonopd/linuxdeployqt/blob/master/BUILDING.md).

## Usage

```
Usage: linuxdeployqt app-binary [options]

Options:
   -verbose=<0-3>      : 0 = no output, 1 = error/warning (default), 2 = normal, 3 = debug
   -no-plugins         : Skip plugin deployment
   -appimage           : Create an AppImage
   -no-strip           : Don't run 'strip' on the binaries
   -bundle-non-qt-libs : Also bundle non-core, non-Qt libraries
   -executable=<path>  : Let the given executable use the deployed libraries too
   -qmldir=<path>      : Scan for QML imports to bundle from the given directory, determined by Qt's qmlimportscanner
   -always-overwrite   : Copy files even if the target file exists
   -qmake=<path>       : The qmake executable to use
   -no-translations    : Skip deployment of translations

linuxdeployqt takes an application as input and makes it
self-contained by copying in the Qt libraries and plugins that
the application uses.
```

#### Simplest example

Given that a desktop file should be provided with an AppImage, `linuxdeployqt` can use that to determine the parameters of the build.

`linuxdeployqt ./path/to/appdir/usr/share/application_name.desktop`

Where the _desktop_ file specifies the executable to be run (with `EXEC=`), the name of the applications and an icon.
See [desktop file specification](https://specifications.freedesktop.org/desktop-entry-spec/desktop-entry-spec-latest.html).

For a more detailed example, see "Using linuxdeployqt with Travis CI" below.

#### Checking library inclusion

Open in Qt Creator and build your application. Run it from the command line and inspect it with `ldd` to make sure the correct libraries from the correct locations are getting loaded, as `linuxdeployqt` will use `ldd` internally to determine from where to copy libraries into the bundle.

#### QMake configuration

__Important:__ By default, `linuxdeployqt` deploys the Qt instance that qmake on the $PATH points to, so make sure that it is the correct one. Verify that qmake finds the correct Qt instance like this before running the `linuxdeployqt` tool:

```
qmake -v

QMake version 3.0
Using Qt version 5.7.0 in /tmp/.mount_QtCreator-5.7.0-x86_64/5.7/gcc_64/lib
```

If this does not show the correct path to your Qt instance that you want to be bundled, then adjust your `$PATH` to find the correct `qmake`.

Alternatively, use the `-qmake` command line option to point the tool directly to the qmake executable to be used.

#### Remove unecessary files

Before running linuxdeployqt it may be wise to delete unneeded files that you do not wish to distribute from the build directory. These may be autogenerated during the build. You can delete them like so:

```
find $HOME/build-*-*_Qt_* \( -name "moc_*" -or -name "*.o" -or -name "qrc_*" -or -name "Makefile*" -or -name "*.a" \) -exec rm {} \;
```

Alternatively, you could use `$DESTDIR`.

## Using linuxdeployqt with Travis CI

A common use case for `linuxdeployqt` is to use it on Travis CI after the `make` command. The following example illustrates how to use `linuxdeployqt` with Travis CI. Create a `.travis.yml` file similar to this one (be sure to customize it, e.g., change `APPNAME` to the name of your application as it is spelled in the `Name=` entry of the `.desktop` file):

```
language: cpp
compiler: gcc
sudo: require
dist: trusty

before_install:
  - sudo add-apt-repository ppa:beineri/opt-qt592-trusty -y
  - sudo apt-get update -qq

install:
  - sudo apt-get -y install qt59base
  - source /opt/qt*/bin/qt*-env.sh

script:
  - qmake CONFIG+=release PREFIX=/usr
  - make -j$(nproc)
  - make INSTALL_ROOT=appdir -j$(nproc) install ; find appdir/
  - wget -c -q "https://github.com/probonopd/linuxdeployqt/releases/download/continuous/linuxdeployqt-continuous-x86_64.AppImage"
  - chmod a+x linuxdeployqt-continuous-x86_64.AppImage
  - unset QTDIR; unset QT_PLUGIN_PATH ; unset LD_LIBRARY_PATH
  - export VERSION=$(git rev-parse --short HEAD) # linuxdeployqt uses this for naming the file
  - ./linuxdeployqt-continuous-x86_64.AppImage appdir/usr/share/applications/*.desktop -bundle-non-qt-libs
  - ./linuxdeployqt-continuous-x86_64.AppImage appdir/usr/share/applications/*.desktop -appimage

after_success:
  - find ./appdir -executable -type f -exec ldd {} \; | grep " => /usr" | cut -d " " -f 2-3 | sort | uniq
  - # curl --upload-file ./APPNAME*.AppImage https://transfer.sh/APPNAME-git.$(git rev-parse --short HEAD)-x86_64.AppImage
  - wget -c https://github.com/probonopd/uploadtool/raw/master/upload.sh
  - bash upload.sh ./APPNAME*.AppImage*
  
branches:
  except:
    - # Do not build tags that we create when we upload to GitHub Releases
    - /^(?i:continuous)/
```

When you save your change, then Travis CI should build and upload an AppImage for you. More likely than not, some fine-tuning will still be required.

For this to work, you need to enable Travis CI for your repository as [described here](https://travis-ci.org/getting_started) __prior to merging this__, if you haven't already done so.

By default, qmake `.pro` files generated by Qt Creator unfortunately don't support `make install` out of the box. In this case you will get

```
make: Nothing to be done for `install'.
find: `appdir/': No such file or directory
```

If `qmake` does not allow for `make install` or does not install the desktop file and icon, then you need to change your `.pro` file it similar to https://github.com/probonopd/FeedTheMonkey/blob/master/FeedTheMonkey.pro.

```
  - make INSTALL_ROOT=appdir install ; find appdir/
```

__CMake__ wants `DESTDIR` instead:

```
  - cmake . -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr
  - make -j$(nproc)
  - make DESTDIR=appdir -j$(nproc) install ; find appdir/
```

__autotools__ (the dinosaur that spends precious minutes "checking...") wants `DESTDIR` too but insists on an absolute link which we can feed it using readlink:

```
  - ./configure --prefix=/usr
  - make -j$(nproc)
  - make install DESTDIR=$(readlink -f appdir) ; find appdir/
```

Caution if you encounter

```
qmake PREFIX=/usr CONFIG+=use_qt_paths
```

Here, `CONFIG+=use_qt_paths` needs to be removed, otherwise it will install everything under the Qt installation paths in `/opt/qt58` when using the beineri ppa.

The exception is that you are building Qt libraries that _should_ be installed to the same location where Qt resides on your system, from where it will be picked up by `linuxdeployqt`.

### Sending Pull Requests on GitHub

`linuxdeployqt` is great for upstream application projects that want to release their software in binary form to Linux users quickly and without much overhead. If you would like to see a particular application use `linuxdeployqt`, then sending a Pull Request may be an option to get the upstream application project to consider it. You can use the following template text for Pull Requests but make sure to customize it to the project in question.

```
This PR, when merged, will compile this application on [Travis CI](https://travis-ci.org/) upon each `git push`, and upload an [AppImage](http://appimage.org/) to your GitHub Releases page.

Providing an [AppImage](http://appimage.org/) would have, among others, these advantages:
- Applications packaged as an AppImage can run on many distributions (including Ubuntu, Fedora, openSUSE, CentOS, elementaryOS, Linux Mint, and others)
- One app = one file = super simple for users: just download one AppImage file, [make it executable](http://discourse.appimage.org/t/how-to-make-an-appimage-executable/80), and run
- No unpacking or installation necessary
- No root needed
- No system libraries changed
- Works out of the box, no installation of runtimes needed
- Optional desktop integration with `appimaged`
- Optional binary delta updates, e.g., for continuous builds (only download the binary diff) using AppImageUpdate
- Can optionally GPG2-sign your AppImages (inside the file)
- Works on Live ISOs
- Can use the same AppImages when dual-booting multiple distributions
- Can be listed in the [AppImageHub](https://appimage.github.io/apps) central directory of available AppImages
- Can double as a self-extracting compressed archive with the `--appimage-extract` parameter

[Here is an overview](https://appimage.github.io/apps) of projects that are already distributing upstream-provided, official AppImages.

__PLEASE NOTE:__ For this to work, you need to enable Travis CI for your repository as [described here](https://travis-ci.org/getting_started) __prior to merging this__, if you haven't already done so. Also, You need to set up `GITHUB_TOKEN` in Travis CI for this to work; please see https://github.com/probonopd/uploadtool.
If you would like to see only one entry for the Pull Request in your project's history, then please enable [this GitHub functionality](https://help.github.com/articles/configuring-commit-squashing-for-pull-requests/) on your repo. It allows you to squash (combine) the commits when merging.

If you have questions, AppImage developers are on #AppImage on irc.freenode.net.
```

## Projects using linuxdeployqt

These projects are already using [Travis CI](http://travis-ci.org/) and linuxdeployqt to provide AppImages of their builds:
- https://github.com/probonopd/ImageMagick
- https://github.com/Subsurface-divelog/subsurface/
- https://github.com/jimevins/glabels-qt
- https://travis-ci.org/NeoTheFox/RepRaptor
- https://github.com/electronpass/electronpass-desktop
- https://github.com/lirios/browser
- https://github.com/jeena/FeedTheMonkey
- https://github.com/labsquare/fastQt/
- https://github.com/sqlitebrowser/sqlitebrowser/
- https://github.com/neuro-sys/tumblr-downloader-client
- https://github.com/LongSoft/UEFITool
- https://github.com/dannagle/PacketSender
- https://github.com/nuttyartist/notes
- https://github.com/leozide/leocad/
- https://github.com/Blinkinlabs/PatternPaint
- https://github.com/fathomssen/redtimer
- https://github.com/coryo/amphetype2
- https://github.com/chkmue/MyQtTravisTemplateProject
- https://github.com/chkmue/qttravisCI_1
- https://github.com/eteran/edb-debugger
- https://github.com/crapp/labpowerqt/
- https://github.com/probonopd/linuxdeployqt/ obviously ;-)
- https://github.com/xdgurl/xdgurl
- https://github.com/QNapi/qnapi

This project is already using linuxdeployqt in a custom Jenkins workflow:
- https://github.com/appimage-packages/

These projects are already using linuxdeployqt:
- Autodesk EAGLE for Linux http://www.autodesk.com/products/eagle/free-download
- https://github.com/bjorn/tiled/
- https://github.com/evpo/EncryptPad
- https://github.com/grahamrow/Muview2
- https://github.com/freemountain/quark/
- https://github.com/Mr0815/geraetepruefung/

This project on GitLab uses linuxdeployqt:

- https://gitlab.com/rpdev/opentodolist

These can be bundled successfully using linuxdeployqt:

- https://github.com/probonopd/tiled/blob/patch-1/.travis.yml
- https://gitlab.com/rpdev/opentodolist/issues/96


## Contributing

One great way to contribute is to send Pull Requests to the application projects you'd like to see use linuxdeployqt, as described above. You are also welcome to contribute to linuxdeployqt development itself. Please discuss in the [forum](http://discourse.appimage.org/t/linuxdeployqt-new-linux-deployment-tool-for-qt/57) or using GitHub issues and Pull Requests.

## Contact

The developers are in the channel #AppImage on irc.freenode.net
