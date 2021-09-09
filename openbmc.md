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
