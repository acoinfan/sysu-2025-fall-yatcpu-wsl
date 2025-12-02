# Ubuntu24.04 (wsl)

## 1.安装jdk17

`sudo apt install openjdk-17-jdk`

### 检查安装结果

```shell
$ java --version
openjdk 17.0.16 2025-07-15
```

#### 若不为jdk17
```shell
$ update-alternatives --config java
There are 2 choices for the alternative java (providing /usr/bin/java).

  Selection    Path                                         Priority   Status
------------------------------------------------------------
* 0            /usr/lib/jvm/java-21-openjdk-amd64/bin/java   2111      auto mode
  1            /usr/lib/jvm/java-17-openjdk-amd64/bin/java   1711      manual mode
  2            /usr/lib/jvm/java-21-openjdk-amd64/bin/java   2111      manual mode

  Press <enter> to keep the current choice[*], or type selection number:
```

根据Selection切换到对应版本 (此处number应该为2)

## 2.安装部分工具
```shell
sudo apt install -y git \
    clang \
    make \
    gnupg \
    scala \
    coreutils \
    cmake \
    llvm \
    lld
```

## 3.安装libtinfo5
```shell
cd /tmp
wget http://security.ubuntu.com/ubuntu/pool/universe/n/ncurses/libtinfo5_6.3-2ubuntu0.1_amd64.deb
sudo apt install ./libtinfo5_6.3-2ubuntu0.1_amd64.deb
cd -
```

## 4.下载代码仓库
```shell
git clone --recursive https://github.com/PurplePower/2025-fall-yatcpu-repo
```

## 5.安装sbt包
```shell
echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" | sudo tee /etc/apt/sources.list.d/sbt.list
echo "deb https://repo.scala-sbt.org/scalasbt/debian /" | sudo tee /etc/apt/sources.list.d/sbt_old.list
curl -sL "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x2EE0EA64E40A89B84B2DF73499E82A75642AC823" | sudo tee /etc/apt/trusted.gpg.d/sbt.asc
sudo apt-get update
sudo apt-get install sbt
# up to 2025-08-19
```

## 6.安装 Verilator
```shell
cd $HOME
sudo apt-get install git perl python3 make autoconf g++ \
    flex bison ccache libgoogle-perftools-dev numactl perl-doc
git clone https://github.com/verilator/verilator
cd verilator
git checkout v4.228
git clean -fdx
autoconf
./configure

make -j$(nproc) && sudo make install
```

## 7.测试
进入2025-fall-yatcpu-repo仓库根目录

```shell
cd lab1
sbt "testOnly riscv.singlecycle.RegisterFileTest"
[info] welcome to sbt 1.9.6 (Ubuntu Java 17.0.16)
[info] loading settings for project lab1-build from plugins.sbt ...
[info] loading project definition from /home/a_coin_fan/code/cpu/2025-fall-yatcpu-repo/lab1/project
[info] loading settings for project root from build.sbt ...
[info] set current project to yatcpu (in build file:/home/a_coin_fan/code/cpu/2025-fall-yatcpu-repo/lab1/)
[warn] javaOptions will be ignored, fork is set to false
[info] RegisterFileTest:
[info] Register File of Single Cycle CPU
[info] - should read the written content
[info] - should x0 always be zero
[info] - should read the writing content
[info] Run completed in 4 seconds, 893 milliseconds.
[info] Total number of tests run: 3
[info] Suites: completed 1, aborted 0
[info] Tests: succeeded 3, failed 0, canceled 0, ignored 0, pending 0
[info] All tests passed.
[success] Total time: 6 s, completed Mar 7, 2025, 11:45:14 AM
```
