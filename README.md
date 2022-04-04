# How to create GCC ToolChain for PinePhone (us Qt Cross-Compile)
Revision Date : 2022/04/04<br>
<br><br>

# Preface  
**This article creates a GCC ToolChain for PinePhone.**  
<br>
You can create GCC ToolChain with a newer version of GCC that matches your environment.  
<br>
Here, I have confirmed build tests with SUSE Enterprise 15 SP3 and openSUSE Leap 15.3.  
My PinePhone's OS is Mobian(Phosh) and Manjaro ARM(Phosh).  
<br>
When building GCC ToolChain, please adapt to each user's environment.  
<br>
*Note:*  
*I have confirmed build tests with SUSE Enterprise 15 SP3 and openSUSE Leap 15.3,*  
*GCC 10.2, GDB 11.2 and Binutils 2.38*  
<br><br>

# 1. SSH Setting (PinePhone)
Get the latest updates on PinePhone.  

    # Mobian
    sudo apt-get update  
    sudo apt-get dist-upgrade  
    sudo shutdown -r now  

    # Manjaro
    sudo pacman -Syyu
    sudo shutdown -r now
<br>

Install SSH server on PinePhone.  

    # Mobian
    sudo apt-get install openssh-server  

    # Manjaro
    sudo pacman -S --needed openssh
<br>

Configure the SSH Server to start automatically, and start the SSH Server.  

    # Mobian
    sudo systemctl enable ssh  
    sudo systemctl start ssh  

    # Manjaro
    sudo systemctl enable sshd  
    sudo systemctl start sshd  
<br>


# 2. Install necessary dependencies for PinePhone
Install the dependencies required to build the GCC ToolChain.  
In Addition, you will also install the libraries required for Qt Cross-Compilation.  

    # Mobian
    sudo apt-get install  build-essential cmake unzip pkg-config gdbserver python python3 gfortran gdbserver python python3 \  
                          ccache libicu-dev icu-devtools libhd-dev libsctp1 libsctp-dev libatspi2.0-0 libatspi2.0-dev libzstd1 libzstd-dev \  
                          libinput-bin libinput-dev libts0 libts-bin libts-dev libmtdev1 libmtdev-dev libevdev2 libevdev-dev \  
                          libblkid-dev libffi-dev libglib2.0-dev libglib2.0-dev-bin libmount-dev \  
                          libpcre16-3 libpcre3-dev libpcre32-3 libpcrecpp0v5 libselinux1-dev libsepol1-dev libwacom-dev libassimp-dev libassimp5 \  
                          libfontconfig1-dev libdbus-1-dev libnss3-dev libxkbcommon-dev libjpeg-dev libasound2-dev libudev-dev libxcb-xinerama0 libxcb-xinerama0-dev libpugixml1v5 \  
                          libsqlite3-dev libxslt1-dev libssl-dev \  
                          libatspi2.0-0 libatspi2.0-dev libsctp1 libsctp-dev \  
                          libwayland-bin libwayland-dev libwayland-egl++0 libwayland-egl-backend-dev libwayland-client++0 libwayland-client-extra++0 libwayland-cursor++0 wayland-scanner++ \
                          waylandpp-dev libweston-9-dev libgles2-mesa-dev libegl-dev libgegl-dev libegl1-mesa-dev libgles-dev libwayland-egl1-mesa
    
    # Manjaro
    sudo pacman -S --needed base-devel rsync vi vim util-linux-libs glib2 make cmake unzip pkg-config \
                            gdb gdb-common gdbm gcc gcc-libs gcc-fortran python2 python3 \
                            ccache icu lksctp-tools python-atspi zstd libinput libtsm mtdev \
                            libevdev libffi pcre pcre2 libwacom assimp fontconfig dbus dbus-c++ nss \
                            libxkbcommon alsa-lib libxinerama pugixml sqlite libxslt openssl ffmpeg \
                            wayland wayland-utils wayland-protocols egl-wayland waylandpp \
                            waylandpp wrapland wlc wayfire glew-wayland glfw-wayland libva1 \
                            mesa mesa-utils glu libglvnd libb2 lttng-ust libproxy
<br>

Use the rsync command to synchronize the files between Linux PC and PinePhone.  
However, **some of the files to be synchronized require root privileges.**  
<br>
Therefore, add the following settings to the /etc/sudoers file so that all files can be synchronized even by ordinary users.  
With the following settings, the rsync command will be executed with super user privileges if necessary.  

    echo "$USER ALL=NOPASSWD:$(which rsync)" | sudo tee --append /etc/sudoer
<br>

Restart PinePhone just in case.  

    sudo shutdown -r now
<br><br>


# 3. Download PinePhone's System Root (Linux PC)
It is necessary to synchronize with the root directory of PinePhone, create the system root directory.  

    mkdir -p ~/<System Root PinePhone> && \  
             ~/<System Root PinePhone>/usr && \  
             ~/<System Root PinePhone>/usr/share  
<br>

    rsync -avz --rsync-path="sudo rsync" --delete --rsh="ssh" <PinePhone's User Name>@<PinePhone's IP Address or Host Name>:/lib ~/<System Root PinePhone>/  
    rsync -avz --rsync-path="sudo rsync" --delete --rsh="ssh" <PinePhone's User Name>@<PinePhone's IP Address or Host Name>:/usr/lib ~/<System Root PinePhone>/usr  
    rsync -avz --rsync-path="sudo rsync" --delete --rsh="ssh" <PinePhone's User Name>@<PinePhone's IP Address or Host Name>:/usr/include ~/<System Root PinePhone>/usr  
    rsync -avz --rsync-path="sudo rsync" --delete --rsh="ssh" <PinePhone's User Name>@<PinePhone's IP Address or Host Name>:/usr/share/pkgconfig ~/<System Root PinePhone>/usr/share  
<br><br>


# 5. Install GCC ToolChain build necessary dependencies (Linux PC)
    sudo zypper install \
       patterns-base-basesystem patterns-devel-base-devel_basis \
       patterns-devel-C-C++-devel_C_C++ gcc gcc-c++ \
       make tar git pkg-config m4 gperf gawk bison flex ncurses-devel \
       gmp-devel mpfr-devel mpc-devel isl-devel python3-devel
<br><br>


# 6. Install GCC ToolChain
## 6.1. Download and Install Binutils from Source Code (Linux PC)
Access the official Binutils Web site to download and extract the Binutils source code.  
https://ftp.gnu.org/gnu/binutils/  

    tar xf binutils-<version>.tar.xz  
    cd binutils-<version>
<br>

Build and install Binutils.  

    mkdir build && cd build
    
    ../configure \
    --prefix=<Binutils's Install Directory>              \
    --build=x86_64-pc-linux-gnu                          \
    --host=x86_64-pc-linux-gnu                           \
    --target=aarch64-linux-gnu                           \
    --disable-gdb --disable-nls --disable-multilib --disable-werror \
    --enable-gold --enable-lto --enable-plugins --enable-relro      \
    --with-sysroot=<PinePhone's System Root Directory>              \
    CFLAGS="-g0 -O3 -fstack-protector-strong"                       \
    CXXFLAGS="-g0 -O3 -fstack-protector-strong"

    make -j $(nproc)

    make install
<br><br>


## 6.2. Download and Install GCC from Source Code (Linux PC)
Access the official GCC Web site to download and extract the GCC source code.  
https://gcc.gnu.org  

    tar xf gcc-<version>.tar.xz  
    cd gcc-<version>
<br>

Build and install GCC.  

    mkdir build && cd build
    
    export PATH="/<Binutils's Install Directory>/bin:$PATH" && \
    ../configure -v                          \
    --prefix=<Binutils's Install Directory>  \
    --build=x86_64-pc-linux-gnu              \
    --host=x86_64-pc-linux-gnu               \
    --target=aarch64-linux-gnu               \
    --enable-languages=c,c++                 \
    --disable-bootstrap --disable-multilib --disable-nls \
    --with-sysroot=<PinePhone's System Root Directory> \
    CFLAGS="-g0 -O3 -fstack-protector-strong" \
    CXXFLAGS="-g0 -O3 -fstack-protector-strong"
    
    make -j $(nproc)
    
    make install-strip
<br>


## 6.3. Download and Install GDB from Source Code (Linux PC)
Access the official GDB Web site to download and extract the GDB source code.  
https://ftp.gnu.org/gnu/gdb/  

    tar xf gdb-<version>.tar.xz  
    cd gdb-<version>
<br>

Build and install GDB.  

    mkdir build && cd build
    
    [ -z "${PYTHON}" ] && export PYTHON=$(command -s python3); \
    PYTHON_LIBDIR=$("${PYTHON}" -c "import sysconfig; print(sysconfig.get_config_var('LIBDIR'))"); \
    export PATH="/<Binutils's Install Directory>/bin:$PATH" && \
    ../configure                            \
    --prefix=<Binutils's Install Directory> \
    --build=x86_64-pc-linux-gnu             \
    --host=x86_64-pc-linux-gnu              \
    --target=aarch64-linux-gnu              \
    --with-python="${PYTHON}" LDFLAGS="-L${PYTHON_LIBDIR} -static-libstdc++" \
    --disable-multilib --disable-nls        \
    --with-sysroot=<PinePhone's System Root Directory>  \ # may not be necessary
    CFLAGS="-g0 -O3 -fstack-protector-strong"           \ # may not be necessary
    CXXFLAGS="-g0 -O3 -fstack-protector-strong"           # may not be necessary
    
    make -j $(nproc)
    
    make install
    # or
    make -C gdb install  # Minimal installation
<br><br>


# 7. Configuration for Qt Creator  
Launch Qt Creator.  
<br>

* Setting up the Qt compiler GCC ToolChain.  
Qt Creator - [Tool] - [Option] - [Kits] - [Compiler] -[Add] - [GCC] - [C]  
/<GCC ToolChain's Directory>/bin/aarch64-linux-gnu-gcc  
Qt Creator - [Tool] - [Option] - [Kits] - [Compiler] -[Add] - [GCC] - [C++]  
/<GCC ToolChain's Directory>/bin/aarch64-linux-gnu-g++  

* Setting up the Qt Debugger GCC ToolChain.  
Qt Creator - [Tool] - [Option] - [Kits] - [Debugger] -[Add]  
/<GCC ToolChain's Directory>/bin/aarch64-linux-gnu-gdb  
<br>

Make sure you can debug Qt project.  
<br><br>
