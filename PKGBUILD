# AArch64 multi-platform
# Maintainer: Jat-faan Wong
# Contributor: Jat-faan Wong, Guoxin "7Ji" Pu, Joshua-Riek 

_panthor_base=aa54fa4e0712616d44f2c2f312ecc35c0827833d
_panthor_branch=linux-6.1-stan-rkr3-panthor
pkgbase=linux-aarch64-rockchip-bsp6.1-fydetab-git
pkgname=("${pkgbase}"{,-headers})
pkgver=6.1.75.r1272327.g7348ed5d
pkgrel=3
arch=('aarch64')
license=('GPL2')
url="https://github.com/Linux-for-Fydetab-Duo"
_desc="for Fydetab Duo" 
makedepends=('cpio' 'xmlto' 'docbook-xsl' 'kmod' 'inetutils' 'bc' 'git' 'dtc')
options=('!strip')
_srcname='linux-rockchip'
source=(
  "git+${url}/${_srcname}.git#branch=noble"
  'custom_reconfig'
  "panthor.patch::https://github.com/hbiyik/linux/compare/${_panthor_base}...${_panthor_branch}.patch"
  "gpu_pll_tune.patch::https://github.com/hbiyik/linux/commit/e4fd428dd34fe13cbd5fa6ed79e2f787bc7655b0.patch"
)

sha512sums=(
  'SKIP'
  '59ed71981bf0e180f40c55925fe91692e96ab47aff9d67a230097382244d63961e8ddddeecb97bfeed9918bcf3616adcda1c9bcc2f0d29b3cab94337b7d2249d'
  '89161d6da2a5487b3ce520125fd4cd3c03b674c390bab3884fd824caf8cc36186074b696c1295ca3c0f6d14e57bf829a450f1c9faac207788832bfb32872424b'
  'SKIP'
)

pkgver() {
  cd "${_srcname}"
  printf "%s.%s%s%s.r%s.g%s" \
    "$(grep '^VERSION = ' Makefile|awk -F' = ' '{print $2}')" \
    "$(grep '^PATCHLEVEL = ' Makefile|awk -F' = ' '{print $2}')" \
    "$(grep '^SUBLEVEL = ' Makefile|awk -F' = ' '{print $2}'|grep -vE '^0$'|sed 's/.*/.\0/')" \
    "$(grep '^EXTRAVERSION = ' Makefile|awk -F' = ' '{print $2}'|tr -d -|sed -E 's/rockchip[0-9]+//')" \
    "$(git rev-list --count HEAD)" \
    "$(git rev-parse --short=8 HEAD)"
}

prepare() {
  cd "${_srcname}"
  
  rm -rf localversion*
  echo "Setting version..."
  echo "-rockchip" > localversion.10-pkgname
  #echo "-r$(git rev-list --count HEAD)" > localversion.20-revision

  for p in $srcdir/*.patch; do
    echo "Patching with ${p}"
    patch -p1 -N -i $p
  done

  echo "Preparing config..."
  scripts/kconfig/merge_config.sh -m debian.rockchip/config/config.common.ubuntu ../custom_reconfig
}

build() {
  cd "${_srcname}"

  make olddefconfig prepare
  make -s kernelrelease > version

  unset LDFLAGS
  make ${MAKEFLAGS} Image modules
  make ${MAKEFLAGS} DTC_FLAGS="-@" dtbs
}

_package() {
  pkgdesc="The ${_srcname} kernel, ${_desc}"
  depends=('coreutils' 'kmod' 'initramfs')
  optdepends=('wireless-regdb: to set the correct wireless channels of your country')

  cd "${_srcname}"
  
  # install dtbs
  make INSTALL_DTBS_PATH="${pkgdir}/boot/dtbs/${pkgbase}" dtbs_install

  # install modules
  make INSTALL_MOD_PATH="${pkgdir}/usr" INSTALL_MOD_STRIP=1 modules_install

  # copy kernel
  local _dir_module="${pkgdir}/usr/lib/modules/$(<version)"
  install -Dm644 arch/arm64/boot/Image "${_dir_module}/vmlinuz"

  # remove reference to build host
  rm -f "${_dir_module}/"{build,source}

  # used by mkinitcpio to name the kernel
  echo "${pkgbase}" | install -D -m 644 /dev/stdin "${_dir_module}/pkgbase"
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the ${_srcname} kernel, ${_desc}"
  depends=("python")

  cd "${_srcname}"
  local builddir="${pkgdir}/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map version
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/arm64" -m644 arch/arm64/Makefile
  cp -t "$builddir" -a scripts

  # add xfs and shmem for aufs building
  mkdir -p "$builddir"/{fs/xfs,mm}

  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/arm64" -a arch/arm64/include
  install -Dt "$builddir/arch/arm64/kernel" -m644 arch/arm64/kernel/asm-offsets.s
  mkdir -p "$builddir/arch/arm"
  cp -t "$builddir/arch/arm" -a arch/arm/include

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

  # https://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # https://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # https://bugs.archlinux.org/task/71392
  install -Dt "$builddir/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  echo "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch = */arm64/ || $arch == */arm/ ]] && continue
    echo "Removing $(basename "$arch")"
    rm -r "$arch"
  done

  echo "Removing documentation..."
  rm -r "$builddir/Documentation"

  echo "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -bi "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v $STRIP_SHARED "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v $STRIP_STATIC "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v $STRIP_BINARIES "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name -print0)

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

for _p in ${pkgname[@]}; do
  eval "package_${_p}() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#${pkgbase}}
  }"
done