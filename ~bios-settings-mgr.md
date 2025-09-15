## Index

- [Introduction](#introduction)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

- D-Bus Hierarchy

```
xyz.openbmc_project.BIOSConfigManager
    /xyz/openbmc_project/bios_config/manager
        yz.openbmc_project.BIOSConfig.Manager
            [P] BaseBIOSTable: A D-Bus dictionary that holds the full BIOS table, including all attributes, their types, descriptions, and default values.
            [P] pendingAttributes: A D-Bus dictionary that stores a list of attributes with their new values, waiting for a reboot to be applied.
            [P] ResetBIOSSettings: A property that indicates a BIOS reset is pending.
            [M] SetAttribute: Sets a new pending value for a specific BIOS attribute.
            [M] GetAttribute: Retrieves the current and pending values for a given attribute.

xyz.openbmc_project.BIOSConfigManager
    /xyz/openbmc_project/bios_config/password
        xyz.openbmc_project.BIOSConfig.Password
            [P] PasswordInitialized: A boolean property indicating if a password has been set.
            [M] ChangePassword: Changes the BIOS setup password.

xyz.openbmc_project.BIOSConfigManager
    /xyz/openbmc_project/bios_config/secure_boot
        xyz.openbmc_project.BIOSConfig.SecureBoot
            [P] CurrentBoot: Indicates the UEFI Secure Boot state during the current boot cycle.
            [P] PendingEnable: A boolean indicating if the Secure Boot setting will take effect on the next boot.
            [P] Mode: The current UEFI Secure Boot mode (e.g., Setup, User, Audit).
```

## <a name="reference"></a> Reference
- [Remote BIOS Configuration](https://github.com/openbmc/bios-settings-mgr)
