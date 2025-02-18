# Swift Cross Compilation Toolchains

This project extends the work of [Johannes Weiss](https://github.com/weissi) and [Helge Heß](https://github.com/AlwaysRightInstitute/swift-mac2arm-x-compile-toolchain) to create MacOS cross-compilers which target Ubuntu on amd64 and arm64, respectively.  All real credit goes to them.

This version extends the previous work by:

1. adding targets for armv7 (with armv6 to come shortly, hopefully)
2. incorporating Swift 5.0
3. creating a complete runtime library which can be used with Docker containers or natively to provide swift applications with precisely the libraries they were originally compiled with. 

IMO, the last point is the most important.  This makes is possible to deploy “distro-less” docker containers of your swift applications which are extremely small.  I am currently working on several R/Pi servers as examples which use this technique.

## Easy way to get started:

Just use the installers at: [github](https://github.com/CSCIX65G/SwiftCrossCompilers/releases).  Then skip the hard way immediately below and proceed directly to: `Using your cross-compiler`

## Hard way - Build your own: 

### Requirements
Homebrew coreutils and jq installed with:

    brew install coreutils jq

### Building the  toolchain on MacOS

To start, create, the directory and fetch the code to do the build (it’s just a complicated bash script):

```
git clone https://github.com/CSCIX65G/SwiftCrossCompilers.git
cd SwiftCrossCompilers
```
To build an arm64 cross compiler (for R/Pi 64-bit OSes):

    ./build_cross_compiler Configurations/arm64-5.0-RELEASE.json

To build an arm32v7 cross compiler (for R/Pi 32-bit OSes on armv7, e.g. R/Pi 2 and 3):

    ./build_cross_compiler Configurations/armv7-5.0-RELEASE.json

To build an amd64 cross compiler (one that will run on a std cloud instance or in Docker on your mac):

    ./build_cross_compiler Configurations/amd64-5.0-RELEASE.json

Each call to the build script will take several minutes to complete. (Particularly the steps where it:

* fetches the toolchain 
* fetches and parses the Ubuntu package files and 
* builds the ld.gold linker. 

are the long steps 

When it does finish, you should get a message saying all is well and that the directories Toolchains, SDKs, and Destinations, populated with various (arm64|arm32|amd64) files have been produced.

The cross compilers end up under: `./InstallPackagers/SwiftCrossCompiler/Developer` by default.  If you don't want to make installer packages, you can simply copy the Toolchains, SDKs and Destinations directories from there to `/Library/Developer`.  NB If you wish to change the installed location from `/Library/Developer` you will need to change the files under Destinations to match your new installed location.  Changes required in the file should be obvious.

## Using your cross compiler
In this directory, you may now do the following (based on which cross compiler you have built)

    cd helloworld
    swift build --destination ../Destinations/arm64-ubuntu-bionic.json
    swift build --destination ../Destinations/amd64-ubuntu-bionic.json
    swift build --destination ../Destinations/armhf-ubuntu-bionic.json

If this finishes successfully you have an arm64 executable in `.build/aarch64-unknown-linux/debug/helloworld`, an amd64 executable in `.build/x86_64-unknown-linux/debug/helloworld` and an armv7 executable in `.build/armhf-unknown-linux/debug/helloworld`

The same technique may be used on any of your own code which compiles with the Swift Package Manager.

### Remote execution on a Raspberry Pi

Now type:

    ./build-arm64.sh

This will build the helloworld program and put it into a docker image called helloworld:arm64-latest.  Transport that container to the Pi along with the script `run-arm64.sh` .  That script will run and print "Successful launch!" at the console.

Remote debugging on the Pi
Now for the more experimental stuff (I'm still debugging remote lldb):

Now on your mac type:

    /Library/Developer/Toolchains/arm64-swift.xctoolchain/usr/bin/lldb .build/aarch64-unknown-linux/debug/helloworld

This will produce an lldb prompt.   (NB, you must use the version of lldb that ships with the toolchain, not the Xcode default)

    env LD_LIBRARY_PATH=/swift-runtime//usr/lib/swift/linux:/swift-runtime/usr/lib/aarch64-linux-gnu:/swift-runtime/lib/aarch64-linux-gnu
    platform connect  connect://[IP or FQDN of your R/Pi]:9293

The first command tells the lldb-server running on your Pi where to find the swift runtime libraries. The last command should produce output similar to:

```
  Platform: remote-linux
    Triple: aarch64-*-linux-gnu
OS Version: 4.14.95 (4.14.95-hypriotos-v8)
    Kernel: #1 SMP PREEMPT Thu Jan 31 15:56:11 UTC 2019
  Hostname: e16dev-dev
 Connected: yes
WorkingDir: /debug
```

The first time you launch lldb you may need to say:
platform select remote-linux

Now say:

    run

(I find that sometimes you need exit lldb and rerun the steps above to get the remote process to launch successfully).

Running takes a while especially on first launch, because it has to copy the `helloworld` file to the R/Pi, but you should eventually see something like:

```
Process 32 launched: '.build/aarch64-unknown-linux/debug/helloworld' (aarch64)
Hello, world!
Process 32 exited with status = 0 (0x00000000) 
```

Congrats, you have built and remote debugged an arm64 executable on your Mac.


