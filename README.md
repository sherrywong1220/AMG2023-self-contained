# AMG2023-self-contained

It's a guide for building a self-contained version of AMG2023 on your host. For best results, it is highly recommended to build in a clean environment, such as within a Docker container.

## Getting Started

### Setting up a Docker Container
```bash
docker run -it --name ubuntu-amg ubuntu:22.04  /bin/bash
mkdir /workspace
```

### Installing Necessary Tools in Docker

```bash
apt-get update
apt-get install git wget build-essential python3 python3-pip vim libpmix-dev meson gperf libcap-dev pkg-config libmount-dev
pip3 install jinja2
```

### Cloning the Necessary Codespace
```bash
cd /workspace
git clone https://github.com/sherrywong1220/AMG2023-self-contained.git
```

### Building Fully-Static OpenMPI

```bash
cd /workspace
wget https://download.open-mpi.org/release/open-mpi/v5.0/openmpi-5.0.1.tar.gz
gunzip -c openmpi-5.0.1.tar.gz | tar xf -
mkdir openmpi-5.0.1-install
cd openmpi-5.0.1/

# Disable unneeded features to make your life easier
./configure --prefix=/workspace/openmpi-5.0.1-install --without-memory-manager --disable-dlopen --enable-static --disable-shared --with-psm2=no --with-psm=no --with-ofi=no --without-verbs --without-rdmacm --without-libnuma
make -j32 all
make -j32 install

cd /workspace/openmpi-5.0.1-install/bin
cp mpic++ /usr/local/bin/
cp mpicc  /usr/local/bin/
```

### Building HYPRE
```bash
cd /workspace/hypre-2.30.0/src
./configure
make install
```

### Building Static Library for libudev
```bash
apt download libudev-dev
dpkg-deb -x libudev-dev_249.11-0ubuntu3.12_arm64.deb libudev-dev
cd libudev-dev/

# Normally you can see there is only shared libary (.so files). Debian upstream ends up not providing any static libraries due to the large resulting static files and no demand from users.
# Therefore you need to compile a static library by yourself. Note that libudev is now part of systemd
cd /workspace/systemd-stable-249-stable/
./configure --auto-features=disabled --default-library=static -D standalone-binaries=true -D static-libsystemd=true -D static-libudev=true -D link-udev-shared=false -D link-systemctl-shared=false -D link-networkd-shared=false -D link-timesyncd-shared=false
make
find ./ "libudev*"
cp ./build/libudev.a /usr/local/lib/
```



### Building AMG
```bash
cd /workspace/AMG2023-main-4dad15c/
# show the detailed commmands
make
# Note: The order of libraries is crucial for static linking.
# Compilation may fail on your first execution of `make`. To resolve this, understand the commands in detail and reorganize the library order as needed.

# show detailed build commands
make -n
mpic++ -static -pthread amg.o -o amg -lopen-pal -lpmix -lhwloc -ludev -lz -ldl -lltdl -lrt -lc  -lgcc /workspace/hypre-2.30.0/src/hypre/lib/libHYPRE.a -luuid -levent -lutil -lnuma -lm -lrt --showme


# reorganize the order of library
g++ -static -pthread amg.o -o amg -lopen-pal -lpmix -lhwloc -ludev -lz -ldl -lltdl -lrt -lc -lgcc /workspace/hypre-2.30.0/src/hypre/lib/libHYPRE.a -luuid -levent -lutil -lnuma -lm -lrt -I/workspace/openmpi-5.0.1-install/include -L/workspace/openmpi-5.0.1-install/lib -Wl,-rpath -Wl,/workspace/openmpi-5.0.1-install/lib -Wl,--enable-new-dtags -lmpi -lopen-pal -lpmix -lz -lm -levent_core -levent_pthreads -lhwloc -lz -levent_core -levent_pthreads -lhwloc -ludev -lrt

# Once successfully compiled, verify if the executable is static.
ldd amg
# If it shows `not a dynamic executable`, it is static.
# Run amg
./amg
```

## extract AMG binary
docker cp ubuntu-amg:/workspace/AMG2023-main-4dad15c/amg /home/user/

## References
https://unix.stackexchange.com/questions/718163/trouble-compiling-systemd-standalone-binaries
https://docs.open-mpi.org/en/v5.0.x/building-apps/building-static-apps.html











