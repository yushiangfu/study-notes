- 'setup' flow 

What does it do when we _source ./setup romulus_?
```
setup
     search all the valid configuration files
     if match
          oe-init-build-env
            └─ oe-buildenv-internal: determine build folder, add bin & scripts path into PATH
            └─ oe-setup-builddir: prepare default local.conf & bblayers.conf
          cd to build folder
  
```
- Dependency (for bitbake obmc-phosphor-image)
```
base.bbclass
     inherit patch.bbclass
     inherit staging.bbclass
     inherit mirrors.bbclass
     inherit utils.bbclass
     inherit utility-tasks.bbclass
     inherit metadata_scm.bbclass
     inherit logging.bbclass
     
logging
     (none)
     
metadata_scm.bbclass
     (none)
     
utility-tasks.bbclass
     (none)
     
utils.bbclass
     (none)
     
mirrors.bbclass
     (none)
     
staging.bbclass
     (none)
     
patch.bbclass
     inherit terminal.bbclass

obmc-phosphor-image.bbclass
     inherit core-image.bbclass
     inherit obmc-phosphor-utils.bbclass
     
core-image.bbclass
     inherit image.bbclass
     
image.bbclass
     inherit rootfs_rpm.bbclass
     inherit image_types.bbclass
     inherit image_types_phosphor.bbclass <--------- where sets this?
     inherit phosphor-rootfs-postcommands.bbclass <--------- where sets this?
     inherit license_image.bbclass <--------- where sets this?
     inherit populate_sdk_ext.bbclass
     inherit image_types_wic.bbclass
     inherit rootfs-postcommands.bbclass
     inherit image-postinst-intercepts.bbclass
     
rootfs_rpm.bbclass
     (none)
     
image_types.bbclass
     inherit siteinfo.bbclass
     inherit kernel-arch.bbclass
     inherit image-artifact-names.bbclass
     
siteinfo.bbclass
     (none)
     
image-artifact-names.bbclass
     (none)
     
image_types_phosphor.bbclass
     inherit image_version.bbclass
     
image_version.bbclass
     (none)
     
populate_sdk_ext.bbclass
     inherit populate_sdk_base.bbclass
     
populate_sdk_base.bbclass
     inherit meta.bbclass
     inherit image-postinst-intercepts.bbclass
     inherit image-artifact-names.bbclass
     
image_types_wic.bbclass
     (none)
     
rootfs-postcommands.bbclass
     inherit image-artifact-names.bbclass

obmc-phosphor-utils.bbclass
     (none)
     
autotools.bbclass
     inherit siteinfo.bbclass
     inherit siteconfig.bbclass
     
siteconfig.bbclass
     (none)
     
pkgconfig.bbclass
     (none)
     
gettext.bbclass
     (none)
     
python3-dir.bbclass
     (none)
     
features_check.bbclass
     (none)
     
systemd.bbclass
     (none)
     
native.bbclass
     inherit relocatable.bbclass
     inherit nopackages.bbclass
     
relocatable.bbclass
     inherit chrpath.bbclass
     
chrpath.bbclass
     (none)
     
nopackages.bbclass
     (none)
     
perlnative.bbclass
     (none)
     
autotools-brokensep.bbclass
     autotools.bbclass
     
pypi.bbclass
     (none)
     
setuptools3.bbclass
     inherit distutils3.bbclass

distutils3.bbclass
     inherit distutils3-base.bbclass
     
distutils3-base.bbclass
     inherit distutils-common-base
     inherit python3native
     inherit python3targetconfig
     
distutils-common-base.bbclass
     (none)
     
python3native.bbclass
     inherit python3-dir
     
python3targetconfig.bbclass
     inherit python3native
     
ptest.bbclass
     (none)

update-rc.d.bbclass
     (none)

cmake.bbclass
     (none)
     
useradd.bbclass
     inherit useradd_base
     
useradd_base.bbclass
     (none)
     
multilib_header.bbclass
     inherit siteinfo.bbclass
     
multilib_script.bbclass
     inherit update-alternatives
     
update-alternatives.bbclass
     (none)
     
cpan-base.bbclass
     inherit perl-version.bbclass
     
perl-version.bbclass
     (none)
     
python3targetconfig.bbclass
     inherit python3native
     
bash-completion.bbclass
     (none)
     
module.bbclass
     inherit module-base.bbclass
     inherit kernel-module-split.bbclass
     inherit pkgconfig.bbclass

module-base.bbclass
     inherit kernel-arch.bbclass
     
kernel-arch.bbclass
     (none)
     
kernel-module-split.bbclass
     (none)
     
nativesdk.bbclass
     (none)
     
packagegroup.bbclass
     inherit allarch.bbclass
     
allarch.bbclass
     (none)
     
meta.bbclass
     (none)
     
meson.bbclass
     inherit python3native.bbclass
     inherit meson-routines.bbclass
     
meson-routines.bbclass
     inherit siteinfo.bbclass

openpower-fru-vpd.bbclass
     (none)
     
mrw-xml.bbclass
     (none)
     
obmc-phosphor-dbus-service.bbclass
     inherit dbus-dir.bbclass
     inherit obmc-phosphor-utils.bbclass
     inherit obmc-phosphor-systemd
     
dbus-dir.bbclass
     (none)
     
obmc-phosphor-systemd
     inherit obmc-phosphor-utils
     inherit systemd.bbclass
     inherit useradd.bbclass
     
phosphor-dbus-yaml.bbclass
     (none)
     
phosphor-logging-yaml-provider.bbclass
     inherit phosphor-dbus-yaml
     
openpower-occ-control.bbclass
     (none)
     
obmc-phosphor-ipmiprovider-symlink.bbclass
     inherit obmc-phosphor-utils
     
phosphor-ipmi-host-whitelist.bbclass
     (none)
     
phosphor-ipmi-host.bbclass
     (none)
     
phosphor-ipmi-fru.bbclass
     (none)
     
phosphor-mapper.bbclass
     inherit phosphor-mapperdir.bbclass
     inherit obmc-phosphor-utils.bbclass
     
phosphor-mapperdir.bbclass
     (none)
     
dos2unix.bbclass
     (none)
     
scons.bbclass
     inherit python3native.bbclass
     
phosphor-logging.bbclass
     (none)
     
phosphor-inventory-manager.bbclass
     (none)
     
kernel.bbclass     
     inherit linux-kernel-base.bbclass
     inherit kernel-module-split.bbclass
     inherit kernel-fitimage.bbclass
     inherit kernel-arch.bbclass
     inherit deploy.bbclass
     inherit cml1.bbclass
     inherit kernel-artifact-names.bbclass
     inherit kernel-devicetree.bbclass
     
linux-kernel-base.bbclass
     (none)
     
kernel-fitimage.bbclass
     inherit kernel-uboot.bbclass
     inherit kernel-artifact-names.bbclass
     inherit uboot-sign.bbclass
     
kernel-uboot.bbclass
     (none)

kernel-artifact-names.bbclass
     inherit image-artifact-names.bbclass
     
uboot-sign.bbclass
     inherit uboot-config.bbclass
     
uboot-config.bbclass
     (none)
     
obmc-phosphor-kernel-version.bbclass
     (none)
     
deploy.bbclass
     (none)
     
cml1.bbclass
     inherit terminal.bbclass
     
terminal.bbclass
     (none)

kernel-devicetree.bbclass
     (none)
     
linux-yocto.inc
     inherit kernel.bbclass
     inherit kernel-yocto.bbclass
     
kernel-yocto.bbclass
     (none)
     
cross.bbclass
     inherit relocatable.bbclass
     inherit nopackages.bbclass
     
uboot-extlinux-config.bbclass
     (none)
     
socsec-sign.bbclass
     (none)
     
npm.bbclass.bbclass
     inherit python3native.bbclass
     
obmc-phosphor-pydbus-service.bbclass
     inherit obmc-phosphor-dbus-service.bbclass
     inherit obmc-phosphor-py-daemon.bbclass
     
obmc-phosphor-py-daemon
     inherit allarch.bbclass
     inherit obmc-phosphor-systemd.bbclass
     
skeleton-gdbus.bbclass
     inherit skeleton.bbclass
     
skeleton.bbclass
     inherit skeleton-rev.bbclass
     
skeleton-rev.bbclass
     (none)

phosphor-settings-manager.bbclass
     (none)
     
cpan_build.bbclass
     inherit cpan-base.bbclass
     inherit perlnative.bbclass
     
mrw-rev.bbclass
     (none)
     
obmc-xmlpatch.bbclass
     inherit python3native.bbclass
     inherit obmc-phosphor-utils.bbclass
     
obmc-phosphor-sdbus-service.bbclass
     inherit obmc-phosphor-dbus-service.bbclass
     
obmc-phosphor-debug-tarball.bbclass
     inherit image.bbclass
     
skeleton-python.bbclass
     inherit setuptools3.bbclass
     inherit skeleton.bbclass
     inherit allarch.bbclass
     
phosphor-fan.bbclass
     (none)
     
skeleton-sdbus.bbclass
     inherit skeleton.bbclass
     
phosphor-debug-collector.bbclass
     (none)
     
phosphor-dbus-monitor.bbclass
     (none)
     
obmc-phosphor-discovery-service.bbclass
     inherit obmc-phosphor-utils.bbclass

ptest-perl.bbclass
     inherit ptest.bbclass
     
gobject-introspection.bbclass
     inherit python3native.bbclass
     inherit gobject-introspection-data.bbclass
     
gobject-introspection-data.bbclass
     (none)

qemu.bbclass
     (none)
     
gtk-immodules-cache.bbclass
     inherit qemu.bbclass
     
gtk-doc.bbclass
     inherit python3native.bbclass
     inherit pkgconfig.bbclass
     inherit qemu.bbclass
     
gitpkgv.bbclass
     (none)
     
mime.bbclass
     (none)
     
mime-xdg.bbclass
     (none)
     
gtk-icon-cache.bbclass
     inherit features_check.bbclass
     
gconf.bbclass
     (none)
     
binconfig-disabled.bbclass
     (none)
     
lib_package.bbclass
     (none)
     
binconfig.bbclass
     (none)
     
vala.bbclass
     (none)
     
texinfo.bbclas
     (none)
     
gsettings.bbclass
     (none)
     
manpages.bbclass
     inherit qemu.bbclass
     
waf.bbclass
     (none)
     
kernelsrc.bbclass
     inherit linux-kernel-base.bbclass
     
gpe.bbclass
     inherit gettext.bbclass
     
fontcache.bbclass
     qemu.bbclass
     
gnomebase.bbclass
     inherit autotools.bbclass
     inherit pkgconfig.bbclass
     
upstream-version-is-even.bbclass
     (none)
     
ptest-gnome.bbclass
     inherit ptest.bbclass
     
go-mod.bbclass
     inherit go.bbclass
     
go.bbclass
     inherit goarch.bbclass
     
goarch.bbclass
     (none)
     
waf-samba.bbclass
     inherit qemu.bbclass
     inherit python3native.bbclass
     
xmlcatalog.bbclass
     (none)
     
bin_package.bbclass
     (none)
     
devupstream.bbclass
     (none)
     
linux-dummy.bbclass
     (none)
     
gi-docgen.bbclass
     (none)
     
relative_symlinks.bbclass
     (none)
     
pixbufcache.bbclass
     inherit qemu.bbclass
     
gio-module-cache.bbclass
     inherit qemu.bbclass

crosssdk.bbclass
     inherit cross.bbclass
     
cross-canadian.bbclass
     (none)

systemd-boot-cfg.bbclass
     inherit fs-uuid.bbclass
     
fs-uuid.bbclass
     (none)
     
populate_sdk.bbclass
     inherit populate_sdk_base.bbclass

toolchain-scripts-base.bbclass
     (none)
     
toolchain-scripts.bbclass
     inherit toolchain-scripts-base.bbclass
     inherit siteinfo.bbclass
     inherit kernel-arch.bbclass

libc-package.bbclass
     inherit qemu.bbclass
     
grub-efi-cfg.bbclass
     inherit fs-uuid.bbclass
```
