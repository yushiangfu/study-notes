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
```
