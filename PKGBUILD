# shellcheck disable=SC2034,SC2154,SC2164

# Maintainer: JustKidding <jk@vin.ovh>
# Maintainer: James P. Harvey <jamespharvey20 at gmail dot com>
# Maintainer: Christoph Bayer <chrbayer@criby.de>
# Contributor: Felix Yan <felixonmars@archlinux.org>
# Contributor: Sven-Hendrik Haase <sh@lutzhaase.com>
# Contributor: Thomas Dziedzic < gostrc at gmail >
# Contributor: Mathias Stearn <mathias@10gen.com>
# Contributor: Alec Thomas
# Contributor: Fredy García <frealgagu at gmail dot com>

pkgname=mongodb
_pkgname=mongodb
# #.<odd number>.# releases are unstable development/testing
pkgver=7.0.7
pkgrel=1
pkgdesc="A high-performance, open source, schema-free document-oriented database"
arch=("x86_64" "aarch64")
url="https://www.mongodb.com/"
license=("SSPL-1.0")
depends=('libstemmer' 'snappy' 'boost-libs' 'pcre2' 'yaml-cpp' 'gperftools' 'libunwind')
makedepends=('python-psutil' 'python-setuptools' 'python-regex' 'python-cheetah3'
             'python-yaml' 'python-requests' 'python-pymongo' 'boost' 'mongo-c-driver')
optdepends=('mongodb-tools: mongoimport, mongodump, mongotop, etc'
            'mongosh: interactive shell to connect with MongoDB')
backup=("etc/mongodb.conf")
provides=(mongodb="$pkgver")
options=(!debug)
source=("https://fastdl.mongodb.org/src/mongodb-src-r$pkgver.tar.gz"
        mongodb.sysusers
        mongodb.tmpfiles
        mongodb-5.0.2-skip-reqs-check.patch
        mongodb-5.0.2-no-compass.patch
        mongodb-7.0.2-sconstruct.patch)
sha256sums=('a2dce499bf32271baca14f430e3f57360d2ac70248a473ff1d2f381883ac2696'
            '3757d548cfb0e697f59b9104f39a344bb3d15f802608085f838cb2495c065795'
            'b7d18726225cd447e353007f896ff7e4cbedb2f641077bce70ab9d292e8f8d39'
            '4ff40320e04bf8c3e05cbc662f8ea549a6b8494d1fda64b1de190c88587bfafd'
            '41b75d19ed7c4671225f08589e317295b7abee934b876859c8777916272f3052'
            '078d94d712c3bb86a77d13bf2021299f4db2332c9d5346dba1ceb0cce1ba8492')
sha256sums_aarch64=('6dd9f20e153ff2a3e185d9411e9d9ec54ba8ed29a0a1489828ccb047205cceac')
source_aarch64=(extrapatch-sconstruct.patch)

_scons_args=(
  CC="${CC:-gcc}"
  CXX="${CXX:-g++}"
  AR="${AR:-ar}"
  MONGO_DISTMOD=arch

  --use-system-boost
  --use-system-pcre2
  --use-system-snappy
  --use-system-stemmer
  --use-system-yaml
  --use-system-zlib
  --use-system-zstd
  --use-system-tcmalloc
  --use-system-libunwind
  --use-system-libbson
  --use-system-mongo-c

  --use-sasl-client
  --ssl
  --disable-warnings-as-errors
  --runtime-hardening=off
)

all-flag-vars() {
  echo {C,CXX}FLAGS
}

_filter-var() {
  local f x var=$1 new=()
  shift

  for f in ${!var} ; do
    for x in "$@" ; do
      # Note this should work with globs like -O*
      [[ ${f} == "${x}" ]] && continue 2
    done
    new+=( "${f}" )
  done
  export "${var}"="${new[*]}"
}

filter-flags() {
  local v
  for v in $(all-flag-vars) ; do
    _filter-var "${v}" "$@"
  done
  return 0
}

prepare() {
  cd "${srcdir}/${_pkgname}-src-r${pkgver}"

  # Keep historical Arch dbPath
  sed -i 's|dbPath: /var/lib/mongo|dbPath: /var/lib/mongodb|' rpm/mongod.conf

  # Keep historical Arch conf file name
  sed -i 's|-f /etc/mongod.conf|-f /etc/mongodb.conf|' rpm/mongod.service

  # Keep historical Arch user name (no need for separate daemon group name)
  sed -i 's/User=mongod/User=mongodb/' rpm/mongod.service
  sed -i 's/Group=mongod/Group=mongodb/' rpm/mongod.service
  sed -i 's/chown mongod:mongod/chown mongodb:mongodb/' rpm/mongod.service

  # Remove sysconfig file, used by upstream's init.d script not used on Arch
  sed -i '/EnvironmentFile=-\/etc\/sysconfig\/mongod/d' rpm/mongod.service

  # Make systemd wait as long as it takes for MongoDB to start
  # If MongoDB needs a long time to start, prevent systemd from restarting it every 90 seconds
  # See: https://jira.mongodb.org/browse/SERVER-38086
  sed -i 's/\[Service]/[Service]\nTimeoutStartSec=infinity/' rpm/mongod.service

  if check_option debug y; then
    _scons_args+=(--dbg=on)
  fi

  if check_option lto y; then
    _scons_args+=(--lto=on)
  fi

  # apply gentoo patches
  for file in "$srcdir"/*.patch; do
    echo "Applying patch $file..."
    patch -Np1 -i "$file"
  done
}

build() {
  cd "${srcdir}/${_pkgname}-src-r${pkgver}"

  if check_option debug n; then
    filter-flags '-m*'
    filter-flags '-O?'
  fi

  export SCONSFLAGS="$MAKEFLAGS"
  ./buildscripts/scons.py "${_scons_args[@]}" install-devcore
}

package() {
  cd "${srcdir}/${_pkgname}-src-r${pkgver}"

  # Install binaries
  install -D build/install/bin/mongo "$pkgdir/usr/bin/mongo"
  install -D build/install/bin/mongod "$pkgdir/usr/bin/mongod"
  install -D build/install/bin/mongos "$pkgdir/usr/bin/mongos"

  # Keep historical Arch conf file name
  install -Dm644 "rpm/mongod.conf" "${pkgdir}/etc/${_pkgname}.conf"

  # Keep historical Arch service name
  install -Dm644 "rpm/mongod.service" "${pkgdir}/usr/lib/systemd/system/${_pkgname}.service"

  # Install manpages
  #install -Dm644 "debian/mongo.1" "${pkgdir}/usr/share/man/man1/mongo.1"
  install -Dm644 "debian/mongod.1" "${pkgdir}/usr/share/man/man1/mongod.1"
  install -Dm644 "debian/mongos.1" "${pkgdir}/usr/share/man/man1/mongos.1"

  # Install systemd files
  install -Dm644 "${srcdir}/${_pkgname}.sysusers" "${pkgdir}/usr/lib/sysusers.d/${_pkgname}.conf"
  install -Dm644 "${srcdir}/${_pkgname}.tmpfiles" "${pkgdir}/usr/lib/tmpfiles.d/${_pkgname}.conf"

  # Install license
  install -D LICENSE-Community.txt "$pkgdir/usr/share/licenses/mongodb/LICENSE"
}
# vim:set ts=2 sw=2 et:
