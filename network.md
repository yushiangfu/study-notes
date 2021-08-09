## <a name="ncsi"></a> NCSI

```
ncsi_probe_channel
    /* Probe */
    for each package, send cmd 'Deslect Package'      
    for each package
        send cmd 'Select Package'
        for each channel, send send cmd 'Clear Initial State'
        for each channel, send send cmd 'Get Version ID'
        for each channel, send send cmd 'Get Capabilities'
        for each channel, send send cmd 'Get Link'
        send cmd 'Deslect Package'      
    choose active channel
    
    /* Configure */
    send cmd 'Select Package'
    send cmd 'Clear Initial State'
    send cmd 'Set VLAN Filter' to clear VID
    send cmd 'Set VLAN Filter' to set VID
    send cmd 'Enable VLAN' or 'Disable VLAN'
    send cmd 'Set MAC Address'
    send cmd 'Enable Broadcast Filter'
    send cmd 'Disable Global Multicast Filter'
    send cmd 'Enable Channel Network TX'
    send cmd 'Enable Channel'
    send cmd 'AEN Enable'
    send cmd 'Get Link'
    start monitor (ncsi_channel_monitor)
    
        

```
