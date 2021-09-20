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
     
image_types.bbclass
     inherit siteinfo.bbclass
     inherit kernel-arch.bbclass
     inherit image-artifact-names.bbclass
     
siteinfo.bbclass
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
     

     
```
