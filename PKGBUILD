# Maintainer: 7Ji <pugokughin@gmail.com>

_desc="AArch64 vendor kernel for Orange Pi Zero3"
_pkgver_main=6.1.31
_pkgver_suffix=orangepi-zero3
_pkgver_uname="${_pkgver_main}-${_pkgver_suffix}"
_orangepi_repo='linux-orangepi'
_orangepi_commit='3495b5ee0594566c9fed930b96b1cae90600412e' # orange-pi-6.1-sun50iw9
_srcname="${_orangepi_repo}-${_orangepi_commit}"
pkgbase=linux-aarch64-${_pkgver_suffix}
pkgname=(
  "${pkgbase}"
  "${pkgbase}-headers"
)
pkgver="${_pkgver_main}"
pkgrel=1
arch=('aarch64')
url="https://github.com/orangepi-xunlong/${_orangepi_repo}"
license=('GPL2')
makedepends=( # Since we don't build the doc, most of the makedeps for other linux packages are not needed here
  'kmod' 'bc' 'dtc' 'uboot-tools'
)
options=(!strip !distcc)
source=(
  "${_srcname}.tar.gz::${url}/archive/${_orangepi_commit}.tar.gz"
  'config'
  'linux.preset'
)
sha256sums=(
  '5ca25d5f6392ff098f0cc0d2caefcd7f1ad4be3ec41df790dedb894ad765643b'
  'e9f76147c62c8945fb1df6e2fde35f587912cf50e36b5b974c8ad11981759d61'
  'bdcd6cbf19284b60fac6d6772f1e0ec2e2fe03ce7fe3d7d16844dd6d2b5711f3'
)

prepare() {
  cd "${_srcname}"

  # echo "Patching kernel so it could be built with GCC 12..."
  # patch -p1 < ../gcc12-fixups.patch

  echo "Setting version..."
  scripts/setlocalversion --save-scmversion
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux}" > localversion.20-pkgname

  # Prepare the configuration file
  cat "${srcdir}/config" > '.config'
}

build() {
  cd "${_srcname}"

  # get kernel version, which will be used later for modules
  make prepare
  make -s kernelrelease > version

  # Host LDFLAGS or other LDFLAGS set by makepkg/user is not helpful for building kernel: it should links nothing outside of itself
  unset LDFLAGS
  # Only need normal Image, as most Rockchip devices does not need/support Image.gz
  # Image and modules are built in the same run to make sure they're compatible with each other
  # -@ enables symbols in dtbs, so overlay is possible
  make ${MAKEFLAGS} DTC_FLAGS="-@" Image modules dtbs
}

_package() {
  pkgdesc="The Linux Kernel and module - ${_desc}"
  depends=(
    'coreutils'
    'initramfs'
    'kmod'
  )
  optdepends=(
    'uboot-legacy-initrd-hooks: to generate uboot legacy initrd images'
    'linux-firmware: firmware images needed for some devices'
    'wireless-regdb: to set the correct wireless channels of your country'
  )
  
  # 当包被升级或卸载时，应当备份的文件（的路径）序列。
  backup=(
    "etc/mkinitcpio.d/${pkgbase}.preset"
  )

  cd "${_srcname}"

  # Install modules
  echo "Installing modules..."

  # INSTALL_MOD_STRIP=1 这个选项用于指示在安装模块时剥离（strip）符号信息。
  # 这可以减小模块文件的大小，通常在发布时使用以减少存储空间和提高性能。
  make INSTALL_MOD_PATH="${pkgdir}/usr" INSTALL_MOD_STRIP=1 modules_install

  # Install DTBs, not to target pkg, but in srcdir, so the later package() routine could use them
  make INSTALL_DTBS_PATH="${srcdir}/dtbs" dtbs_install

  # Install pkgbase
  local _dir_module="${pkgdir}/usr/lib/modules/$(<version)"
  echo "${pkgbase}" | install -D -m 644 /dev/stdin "${_dir_module}/pkgbase"

  # Install kernel image (this is technically not vmlinuz, but I name it this way to utilize mkinitcpio's existing hooks)
  install -Dm644 arch/arm64/boot/Image "${_dir_module}/vmlinuz"

  # Remove hbuild and source links, which points to folders used when building (i.e. dead links)
  rm -f "${_dir_module}/"{build,source}

  # install mkinitcpio preset file
  sed "s|%PKGBASE%|${pkgbase}|g" ../linux.preset |
    install -Dm644 /dev/stdin "${pkgdir}/etc/mkinitcpio.d/${pkgbase}.preset"

  # Install DTB
  echo 'Installing DTBs for Allwinner SoCs...'
  install -d -m 755 "${pkgdir}/boot/dtbs/${pkgbase}"
  cp -t "${pkgdir}/boot/dtbs/${pkgbase}" -a "${srcdir}/dtbs/allwinner"
}

_package-headers() {
  pkgdesc="Header files and scripts for building modules for linux kernel - ${_desc}"
  # Compiling modules against the tree needs gcc-wrapper.py clang-wrapper.py,
  # which both need Python.
  # Discussed in https://github.com/7Ji/archrepo/issues/5
  # About why depends instead of optdepends, this is a similar decision to
  # https://bugs.archlinux.org/task/69654
  depends=('python')
  
  # Mostly copied from alarm's linux-aarch64 and modified
  cd "${_srcname}"
  local _builddir="${pkgdir}/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "${_builddir}" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "${_builddir}/kernel" -m644 kernel/Makefile
  install -Dt "${_builddir}/arch/arm64" -m644 arch/arm64/Makefile
  cp -t "${_builddir}" -a scripts

  echo "Installing headers..."
  cp -t "${_builddir}" -a include
  cp -t "${_builddir}/arch/arm64" -a arch/arm64/include
  install -Dt "${_builddir}/arch/arm64/kernel" -m644 arch/arm64/kernel/asm-offsets.s


  install -Dt "${_builddir}/drivers/md" -m644 drivers/md/*.h
  install -Dt "${_builddir}/net/mac80211" -m644 net/mac80211/*.h

  # https://bugs.archlinux.org/task/13146
  install -Dt "${_builddir}/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # https://bugs.archlinux.org/task/20402
  install -Dt "${_builddir}/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "${_builddir}/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "${_builddir}/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # https://bugs.archlinux.org/task/71392
  install -Dt "${_builddir}/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "${_builddir}/{}" \;

  echo "Removing unneeded architectures..."
  local _arch
  for _arch in "${_builddir}"/arch/*/; do
    [[ ${_arch} = */arm64/ ]] && continue
    echo "Removing $(basename "${_arch}")"
    rm -r "${_arch}"
  done

  echo "Removing documentation..."
  rm -r "${_builddir}/Documentation"

  echo "Removing broken symlinks..."
  find -L "${_builddir}" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "${_builddir}" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -Sib "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v ${STRIP_SHARED} "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v ${STRIP_STATIC} "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v ${STRIP_BINARIES} "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v ${STRIP_SHARED} "$file" ;;
    esac
  done < <(find "${_builddir}" -type f -perm -u+x ! -name vmlinux -print0)

  echo "Stripping vmlinux..."
  strip -v $STRIP_STATIC "${_builddir}/vmlinux"

  echo "Adding symlink..."
  mkdir -p "${pkgdir}/usr/src"
  ln -sr "${_builddir}" "$pkgdir/usr/src/$pkgbase"
}

for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done