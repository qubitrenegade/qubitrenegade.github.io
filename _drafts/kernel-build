
```
mkdir -p ~/dev/kernel/build && cd "$_"
cp /boot/config-`uname -r` ~/dev/kernel/system.config
git clone -b linux-5.2.y git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git
docker run -it --name kernel-build --mount type=bind,source="${HOME}/dev/kernel",target=/kernel fedora:30
dnf install fedpkg fedora-packager rpmdevtools ncurses-devel pesign qt3-devel libXi-devel gcc-c++ flex bison elfutils-libelf-devel openssl-devel bc
cd /kernel/build
make olddefconfig
make menuconfig
KCFLAGS='-O3 -mtune=native -pipe' KCPPFLAGS='-O3 -mtune=native -pipe' make -j$(nproc) rpm-pkg
```
