- 'setup' flow 

Comparison
```
bitbake -b foo_1.0.bb <----- 'foo_1.0.bb' is the recipe name
bitbake -b foo_1.0.bb -c clean <----- 'clean' is the task
bitbake foo <----- 'foo' is the package or PROVIDES

bblayers.conf: determine target layers by setting BBLAYERS
bitbake.conf: define variables such as DEPENDS, WORKDIR
layer.conf: each layer has one and it sets variables like BBPATH and BBFILES
local.conf: to govern the build behavior, e.g. BB_NUMBER_THREADS

BBPATH: specify where to search .conf and .bbclass
BBFILES: specify where to search .bb and .bbappend
```

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
[allarch.bbclass]
     (none)

[autotools.bbclass]
     inherit siteinfo.bbclass
     inherit siteconfig.bbclass

[autotools-brokensep.bbclass]
     inherit autotools.bbclass

[base.bbclass]
     inherit patch.bbclass
     inherit staging.bbclass
     inherit mirrors.bbclass
     inherit utils.bbclass
     inherit utility-tasks.bbclass
     inherit metadata_scm.bbclass
     inherit logging.bbclass

[bash-completion.bbclass]
     (none)

[binconfig.bbclass]
     (none)

[binconfig-disabled.bbclass]
     (none)

[bin_package.bbclass]
     (none)

[chrpath.bbclass]
     (none)

[cmake.bbclass]
     (none)

[cml1.bbclass] config management?
     inherit terminal.bbclass

[core-image.bbclass]
     inherit image.bbclass

[cpan-base.bbclass]
     inherit perl-version.bbclass

[cpan_build.bbclass]
     inherit cpan-base.bbclass
     inherit perlnative.bbclass

[cross.bbclass]
     inherit relocatable.bbclass
     inherit nopackages.bbclass

[cross-canadian.bbclass]
     (none)

[crosssdk.bbclass]
     inherit cross.bbclass

[dbus-dir.bbclass]
     (none)

[deploy.bbclass]
     (none)

[devupstream.bbclass]
     (none)

[distutils3-base.bbclass]
     inherit distutils-common-base.bbclass
     inherit python3native.bbclass
     inherit python3targetconfig.bbclass

[distutils3.bbclass]
     inherit distutils3-base.bbclass

[distutils-common-base.bbclass]
     (none)

[dos2unix.bbclass]
     (none)

[features_check.bbclass]
     (none)

[fontcache.bbclass]
     inherit qemu.bbclass

[fs-uuid.bbclass]
     (none)

[gconf.bbclass]
     (none)

[gettext.bbclass]
     (none)

[gi-docgen.bbclass]
     (none)

[gio-module-cache.bbclass]
     inherit qemu.bbclass

[gitpkgv.bbclass]
     (none)

[gnomebase.bbclass]
     inherit autotools.bbclass
     inherit pkgconfig.bbclass

[goarch.bbclass]
     (none)

[go.bbclass]
     inherit goarch.bbclass

[gobject-introspection.bbclass]
     inherit python3native.bbclass
     inherit gobject-introspection-data.bbclass

[gobject-introspection-data.bbclass]
     (none)

[go-mod.bbclass]
     inherit go.bbclass

[gpe.bbclass]
     inherit gettext.bbclass

[grub-efi-cfg.bbclass]
     inherit fs-uuid.bbclass

[gsettings.bbclass]
     (none)

[gtk-doc.bbclass]
     inherit python3native.bbclass
     inherit pkgconfig.bbclass
     inherit qemu.bbclass

[gtk-icon-cache.bbclass]
     inherit features_check.bbclass

[gtk-immodules-cache.bbclass]
     inherit qemu.bbclass

[image-artifact-names.bbclass] prepare info for image creation and rootfs popuolation
     (none)

[image.bbclass]
     inherit rootfs_rpm.bbclass
     inherit image_types.bbclass
     inherit image_types_phosphor.bbclass <--------- which file sets this
     inherit phosphor-rootfs-postcommands.bbclass <--------- which file sets this
     inherit license_image.bbclass <--------- which file sets this
     inherit populate_sdk_ext.bbclass
     inherit image_types_wic.bbclass
     inherit rootfs-postcommands.bbclass
     inherit image-postinst-intercepts.bbclass

[image_types.bbclass]
     inherit siteinfo.bbclass
     inherit kernel-arch.bbclass
     inherit image-artifact-names.bbclass

[image_types_phosphor.bbclass]
     inherit image_version.bbclass

[image_types_wic.bbclass]
     (none)

[image_version.bbclass]
     (none)

[kernel-arch.bbclass]
     (none)

[kernel-artifact-names.bbclass]
     inherit image-artifact-names.bbclass

[kernel-uboot.bbclass] prepare linux.bin, compress it if needed
     (none)

[kernel-uimage.bbclass]
     inherit kernel-uboo

[kernel.bbclass]
     inherit linux-kernel-base.bbclass
     inherit kernel-module-split.bbclass
     inherit kernel-fitimage.bbclass
     inherit kernel-arch.bbclass
     inherit deploy.bbclass
     inherit cml1.bbclass
     inherit kernel-artifact-names.bbclass
     inherit kernel-devicetree.bbclass

[kernel-devicetree.bbclass]
     (none)

[kernel-fitimage.bbclass]
     inherit kernel-uboot.bbclass
     inherit kernel-artifact-names.bbclass
     inherit uboot-sign.bbclass

[kernel-module-split.bbclass] add module list to RDEPENDS_XXX?
     (none)

[kernelsrc.bbclass]
     inherit linux-kernel-base.bbclass

[kernel-uboot.bbclass]
     (none)

[kernel-yocto.bbclass]
     (none)

[libc-package.bbclass]
     inherit qemu.bbclass

[lib_package.bbclass]
     (none)

[linux-dummy.bbclass]
     (none)

[linux-kernel-base.bbclass] provide functions for parsing kernel version & module list?
     (none)

[linux-yocto.inc
     inherit kernel.bbclass
     inherit kernel-yocto.bbclass

[logging.bbclass] prepare logging functions
     (none)

[manpages.bbclass]
     inherit qemu.bbclass

[meson.bbclass]
     inherit python3native.bbclass
     inherit meson-routines.bbclass

[meson-routines.bbclass]
     inherit siteinfo.bbclass

[meta.bbclass]
     (none)

[metadata_scm.bbclass] prepare SCM variables/functions for git/svn
     (none)

[mime.bbclass]
     (none)

[mime-xdg.bbclass]
     (none)

[mirrors.bbclass] prepare mirror list in case git repo fails and has to fallback
     (none)

[module-base.bbclass]
     inherit kernel-arch.bbclass

[module.bbclass]
     inherit module-base.bbclass
     inherit kernel-module-split.bbclass
     inherit pkgconfig.bbclass

[mrw-rev.bbclass]
     (none)

[mrw-xml.bbclass]
     (none)

[multilib_header.bbclass]
     inherit siteinfo.bbclass

[multilib_script.bbclass]
     inherit update-alternatives.bbclass

[native.bbclass]
     inherit relocatable.bbclass
     inherit nopackages.bbclass

[nativesdk.bbclass]
     (none)

[nopackages.bbclass]
     (none)

[npm.bbclass]
     inherit python3native.bbclass

[obmc-phosphor-dbus-service.bbclass]
     inherit dbus-dir.bbclass
     inherit obmc-phosphor-utils.bbclass
     inherit obmc-phosphor-systemd.bbclass

[obmc-phosphor-debug-tarball.bbclass]
     inherit image.bbclass

[obmc-phosphor-discovery-service.bbclass]
     inherit obmc-phosphor-utils.bbclass

[obmc-phosphor-image.bbclass]
     inherit core-image.bbclass
     inherit obmc-phosphor-utils.bbclass

[obmc-phosphor-ipmiprovider-symlink.bbclass]
     inherit obmc-phosphor-utils.bbclass

[obmc-phosphor-kernel-version.bbclass]
     (none)

[obmc-phosphor-py-daemon.bbclass]
     inherit allarch.bbclass
     inherit obmc-phosphor-systemd.bbclass

[obmc-phosphor-pydbus-service.bbclass]
     inherit obmc-phosphor-dbus-service.bbclass
     inherit obmc-phosphor-py-daemon.bbclass

[obmc-phosphor-sdbus-service.bbclass]
     inherit obmc-phosphor-dbus-service.bbclass

[obmc-phosphor-systemd.bbclass]
     inherit obmc-phosphor-utils.bbclass
     inherit systemd.bbclass
     inherit useradd.bbclass

[obmc-phosphor-utils.bbclass]
     (none)

[obmc-xmlpatch.bbclass]
     inherit python3native.bbclass
     inherit obmc-phosphor-utils.bbclass

[openpower-fru-vpd.bbclass]
     (none)

[openpower-occ-control.bbclass]
     (none)

[package.bbclass]
     inherit packagedata.bbclass
     inherit chrpath.bbclass
     inherit package_pkgdata.bbclass
     inherit insane.bbclass

[packagegroup.bbclass]
     inherit allarch.bbclass

[patch.bbclass] prepare default do_patch()
     inherit terminal.bbclass

[perlnative.bbclass]
     (none)

[perl-version.bbclass]
     (none)

[phosphor-dbus-monitor.bbclass]
     (none)

[phosphor-dbus-yaml.bbclass]
     (none)

[phosphor-debug-collector.bbclass]
     (none)

[phosphor-fan.bbclass]
     (none)

[phosphor-inventory-manager.bbclass]
     (none)

[phosphor-ipmi-fru.bbclass]
     (none)

[phosphor-ipmi-host.bbclass]
     (none)

[phosphor-ipmi-host-whitelist.bbclass]
     (none)

[phosphor-logging.bbclass]
     (none)

[phosphor-logging-yaml-provider.bbclass]
     inherit phosphor-dbus-yaml.bbclass

[phosphor-mapper.bbclass]
     inherit phosphor-mapperdir.bbclass
     inherit obmc-phosphor-utils.bbclass

[phosphor-mapperdir.bbclass]
     (none)

[phosphor-settings-manager.bbclass]
     (none)

[pixbufcache.bbclass]
     inherit qemu.bbclass

[pkgconfig.bbclass]
     (none)

[populate_sdk_base.bbclass]
     inherit meta.bbclass
     inherit image-postinst-intercepts.bbclass
     inherit image-artifact-names.bbclass

[populate_sdk.bbclass]
     inherit populate_sdk_base.bbclass

[populate_sdk_ext.bbclass]
     inherit populate_sdk_base.bbclass

[ptest.bbclass]
     (none)

[ptest-gnome.bbclass]
     inherit ptest.bbclass

[ptest-perl.bbclass]
     inherit ptest.bbclass

[pypi.bbclass]
     (none)

[python3-dir.bbclass]
     (none)

[python3native.bbclass]
     inherit python3-dir.bbclass

[python3targetconfig.bbclass]
     inherit python3native.bbclass

[qemu.bbclass]
     (none)

[relative_symlinks.bbclass]
     (none)

[relocatable.bbclass]
     inherit chrpath.bbclass

[rootfs-postcommands.bbclass]
     inherit image-artifact-names.bbclass

[rootfs_rpm.bbclass]
     (none)

[scons.bbclass]
     inherit python3native.bbclass

[setuptools3.bbclass]
     inherit distutils3.bbclass

[siteconfig.bbclass]
     (none)

[siteinfo.bbclass]
     (none)

[skeleton.bbclass]
     inherit skeleton-rev.bbclass

[skeleton-gdbus.bbclass]
     inherit skeleton.bbclass

[skeleton-python.bbclass]
     inherit setuptools3.bbclass
     inherit skeleton.bbclass
     inherit allarch.bbclass

[skeleton-rev.bbclass]
     (none)

[skeleton-sdbus.bbclass]
     inherit skeleton.bbclass

[socsec-sign.bbclass]
     (none)

[staging.bbclass] prepare populate_sysroot(),
                          do_populate_sysroot_setscene(),
                          do_prepare_recipe_sysroot(),
                          staging_taskhandler()
     (none)

[systemd.bbclass]
     (none)

[systemd-boot-cfg.bbclass]
     inherit fs-uuid.bbclass

[terminal.bbclass] determine terminal for logging
     (none)

[texinfo.bbclas
     (none)

[toolchain-scripts-base.bbclass]
     (none)

[toolchain-scripts.bbclass]
     inherit toolchain-scripts-base.bbclass
     inherit siteinfo.bbclass
     inherit kernel-arch.bbclass

[uboot-config.bbclass]
     (none)

[uboot-extlinux-config.bbclass]
     (none)

[uboot-sign.bbclass]
     inherit uboot-config.bbclass

[update-alternatives.bbclass]
     (none)

[update-rc.d.bbclass]
     (none)

[upstream-version-is-even.bbclass]
     (none)

[useradd_base.bbclass]
     (none)

[useradd.bbclass]
     inherit useradd_base.bbclass

[tility-tasks.bbclass] prepare do_listtasks()/do_clean()/do_checkuri()
     (none)

[utils.bbclass] prepare utility functions
     (none)

[vala.bbclass]
     (none)

[waf.bbclass]
     (none)

[waf-samba.bbclass]
     inherit qemu.bbclass
     inherit python3native.bbclass

[xmlcatalog.bbclass]
     (none)
```

```
+------+                                        
| main |                                        
+-|----+ image_manager_main.cpp
  |    +------------------+                     
  |--> | sd_event_default | alloc sd_event      
  |    +------------------+                     
  |                                             
  |--> prepare manager                          
  |                                             
  |--> prepare watch, bind Manager::processImage
  |                                             
  +--> loop                                     
```

```
+-----------------------+                                                                        
| Manager::processImage | : untar and check manifest, prepare 'version' obj and add to 'versions'
+-----|-----------------+                                                                        
      |                                                                                          
      |--> return error if arg path isn't a file                                                 
      |                                                                                          
      |    +---------+                                                                           
      |--> | mkdtemp | create folder, for tarball extraction                                     
      |    +---------+                                                                           
      |                                                                                          
      |    +----------------+                                                                    
      |--> | Manager::unTar | fork a child and execve 'tar' to untar the uploaded file           
      |    +----------------+                                                                    
      |                                                                                          
      |--> return error if no manifest file                                                      
      |                                                                                          
      |    +-------------------+                                                                 
      |--> | Version::getValue | get 'version' from manifest file                                
      |    +-------------------+                                                                 
      |    +------------------------+                                                            
      |--> | Version::getBMCMachine | get running machine name from system                       
      |    +------------------------+                                                            
      |    +-------------------+                                                                 
      |--> | Version::getValue | get 'MachineName' from manifest file                            
      |    +-------------------+                                                                 
      |    +-------------------+                                                                 
      |--> | Version::getValue | get 'purpose' from manifest file                                
      |    +-------------------+                                                                 
      |    +-------------------+                                                                 
      |--> | Version::getValue | get 'ExtendedVersion' from manifest file                        
      |    +-------------------+                                                                 
      |    +----------------------------+                                                        
      |--> | Version::getRepeatedValues | get 'CompatibleName' from manifest file                
      |    +----------------------------+                                                        
      |    +-----------+                                                                         
      |--> | std::find | check if the specified version has been uploaded already                
      |    +-----------+                                                                         
      |                                                                                          
      |--> if not                                                                                
      |                                                                                          
      +------> create 'version' object and insert to 'versions'                                  
```
