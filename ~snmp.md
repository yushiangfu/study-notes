```                                                                                                            
 agent/snmp_vars.c                                                                                          
 [init_agent] : register all kinds of handlers and callbacks                                                
 |                                                                                                          
 |--> [init_kmem] return 1 in our case                                                                      
 |                                                                                                          
 |--> [setup_tree] (???)                                                                                    
 |                                                                                                          
 |--> [init_agent_read_config] register config handlers                                                     
 |                                                                                                          
 |--> [_init_agent_callback_transport] setup transport/session, open transport, copy session and add to list
 |                                                                                                          
 |--> [snmp_register_callback] callback = _warn_if_all_disabled()                                           
 |                                                                                                          
 |--> [netsnmp_init_helpers] register helpers                                                               
 |                                                                                                          
 |--> [init_traps] (empty)                                                                                  
 |                                                                                                          
 |--> [netsnmp_container_init_list] register container handlers                                             
 |                                                                                                          
 |--> [init_agent_sysORTable] register callbacks for sysORTable                                             
 |                                                                                                          
 |--> [agentx_config_init] register agentx handlers                                                         
 |                                                                                                          
 |--> if application role = sub_agent                                                                       
 |    -                                                                                                     
 |    +--> [subagent_init] init callback session, register callback for sub_agent                           
 |                                                                                                          
 |--> [netsnmp_udp_agent_config_tokens_register] register config handler for token 'com2sec'                
 |                                                                                                          
 |--> [netsnmp_udp6_agent_config_tokens_register] register config handler for token 'com2sec6'              
 |                                                                                                          
 |--> [netsnmp_unix_agent_config_tokens_register] register config handler for token 'com2secunix'           
 |                                                                                                          
 |--> [netsnmp_certs_agent_init]                                                                            
 |                                                                                                          
 +--> [netsnmp_certs_agent_init] register config handlers for certificate                                   
```

```                                                                                 
 agent/agent_read_config.c                                                       
 [init_agent_read_config] : register config handlers                             
 |                                                                               
 |--> [register_app_config_handler] token = 'authtrapenable'                     
 |                                  parser = snmpd_parse_config_authtrap()       
 |                                                                               
 |--> [register_app_config_handler] token = 'pauthtrapenable'                    
 |                                  parser = snmpd_parse_config_authtrap()       
 |                                                                               
 |--> if role is master_agent                                                    
 |    |                                                                          
 |    |--> [register_app_config_handler] token = 'trapsink'                      
 |    |                                  parser = snmpd_parse_config_trapsink()  
 |    |                                                                          
 |    |--> [register_app_config_handler] token = 'trap2sink'                     
 |    |                                  parser = snmpd_parse_config_trap2sink() 
 |    |                                                                          
 |    |--> [register_app_config_handler] token = 'informsink'                    
 |    |                                  parser = snmpd_parse_config_informsink()
 |    |                                                                          
 |    +--> [register_app_config_handler] token = 'trapsess'                      
 |                                       parser = snmpd_parse_config_trapsess()  
 |                                                                               
 |--> [register_app_config_handler] token = 'trapcommunity'                      
 |                                  parser = snmpd_parse_config_trapcommunity()  
 |                                                                               
 |--> [register_app_config_handler] token = 'agentuser'                          
 |                                  parser = netsnmp_parse_agent_user()          
 |                                                                               
 |--> [register_app_config_handler] token = 'agentgroup'                         
 |                                  parser = netsnmp_parse_agent_group()         
 |                                                                               
 |--> [register_app_config_handler] token = 'agentaddress'                       
 |                                  parser = snmpd_set_agent_address()           
 |                                                                               
 |--> [netsnmp_ds_register_config] * n                                           
 |                                                                               
 +--> [netsnmp_init_handler_conf] register parsers, add a bunch of pairs         
```

```                                                                         
 agent/agent_handler.c                                                   
 [netsnmp_init_handler_conf] : register parsers, add a bunch of pairs    
 |                                                                       
 |--> [snmpd_register_config_handler] token = 'injectHandler'            
 |                                    parser = parse_injectHandler_conf()
 |                                                                       
 |--> [snmp_register_callback] parser = handler_mark_inject_handler_done 
 |                                                                       
 +--> [se_add_pair_to_slist] * n                                         
```

```                                                                                                         
 agent/snmp_vars.c                                                                                       
 [_init_agent_callback_transport] : setup transport/session, open transport, copy session and add to list
 -                                                                                                       
 +--> [netsnmp_callback_open] setup transport/session, open transport, copy session and add to list      
```

```                                                                                                
 snmplib/transports/snmpCallbackDomain.c                                                        
 [netsnmp_callback_open] : setup transport/session, open transport, copy session and add to list
 |                                                                                              
 |--> [netsnmp_callback_transport] alloc and setup 'transport', add to list                     
 |                                                                                              
 |--> [snmp_sess_init] init snmp, setup session                                                 
 |                                                                                              
 |--> further setup session                                                                     
 |                                                                                              
 +--> [snmp_add_full] open transport, copy session and add to list                              
```

```                                                                        
 snmplib/transports/snmpCallbackDomain.c                                
 [netsnmp_callback_transport] : alloc and setup 'transport', add to list
 |                                                                      
 |--> alloc 'transport'                                                 
 |                                                                      
 |--> alloc 'callback info'                                             
 |                                                                      
 |--> setup 'callback info', save in 'transport'                        
 |                                                                      
 |--> [pipe]                                                            
 |                                                                      
 |--> install ops to 'transport'                                        
 |                                                                      
 +--> [netsnmp_transport_add_to_list] register 'transport' to list      
```

```                                                                                               
 snmplib/snmp_api.c                                                                            
 [snmp_sess_init] : init snmp, setup session                                                   
 |                                                                                             
 |--> [_init_snmp] generate random req_id/msg_id, regisiter protocol/port for snmp and snmptrap
 |                                                                                             
 +--> setup session                                                                            
```

```                                                                                            
 snmplib/snmp_api.c                                                                         
 [_init_snmp] : generate random req_id/msg_id, regisiter protocol/port for snmp and snmptrap
 |                                                                                          
 |--> [snmp_res_init] do nothing bc of disabled config                                      
 |                                                                                          
 |--> [netsnmp_tdomain_init] print info                                                     
 |                                                                                          
 |--> get random req_id and msg_id                                                          
 |                                                                                          
 +--> regisiter protocol/port for snmp and snmptrap                                         
```

```                                                                                  
 snmplib/snmp_api.c                                                               
 [snmp_add_full] : open transport, copy session and add to list                   
 |                                                                                
 |--> [snmp_sess_add_ex] init snmp, open transport, copy session and install hooks
 |                                                                                
 +--> add copied session to list                                                  
```

```                                                                                                     
 snmplib/snmp_api.c                                                                                  
 [snmp_sess_add_ex] : init snmp, open transport, copy session and install hooks                      
 |                                                                                                   
 |--> [_init_snmp] generate random req_id/msg_id, regisiter protocol/port for snmp and snmptrap      
 |                                                                                                   
 |--> [netsnmp_sess_config_and_open_transport] save config in 'transport', open (as client or server)
 |                                                                                                   
 |--> if ->f_setup_session exists                                                                    
 |    -                                                                                              
 |    +--> call ->f_setup_session(), e.g.,                                                           
 |         [netsnmp_tlsbase_session_init] setup security fields of session                           
 |                                                                                                   
 |--> [snmp_sess_copy] copy session                                                                  
 |                                                                                                   
 +--> install arg hooks to internal of duplicated session                                            
```

```                                                                                                  
 snmplib/snmp_api.c                                                                               
 [netsnmp_sess_config_and_open_transport] : save config in 'transport', open (as client or server)
 |                                                                                                
 |--> [netsnmp_sess_config_transport] duplicate config data and save in 'transport'               
 |                                                                                                
 |--> if ->f_open exists                                                                          
 |    -                                                                                           
 |    +--> call ->f_open(), e.g.,                                                                 
 |         [netsnmp_tlstcp_open] connect (as client) or accept (as server)                        
 |                                                                                                
 +--> label 'opened' in transport                                                                 
```

```                                                                                                                    
 snmplib/snmp_api.c                                                                                                 
 [netsnmp_sess_config_transport] : duplicate config data and save in 'transport'                                    
 -                                                                                                                  
 +--> if configuration is provided                                                                                  
      -                                                                                                             
      +--> if transport has config                                                                                  
           |                                                                                                        
           |--> get iterator from configuration                                                                     
           |                                                                                                        
           +--> for each config_data                                                                                
                -                                                                                                   
                +--> call ->f_config()                                                                              
                     e.g., [netsnmp_tlsbase_config] given token, duplicate value and save in 'transport' accordingly
```

```                                                                                               
 snmplib/transports/snmpTLSTCPDomain.c                                                         
 [netsnmp_tlstcp_open] : connect (as client) or accept (as server)                             
 |                                                                                             
 |--> if flag has specified 'client'                                                           
 |    -                                                                                        
 |    +--> [netsnmp_tlstcp_open_client] setup ctx/bio/ssl_layer, connect and verify server cert
 |                                                                                             
 +--> else                                                                                     
      -                                                                                        
      +--> [netsnmp_tlstcp_open_server] alloc bio and do accept, setup ssl_ctx                 
```

```                                                                                       
 snmplib/transports/snmpTLSTCPDomain.c                                                 
 [netsnmp_tlstcp_open_client] : setup ctx/bio/ssl_layer, connect and verify server cert
 |                                                                                     
 |--> [sslctx_client_setup] alloc ssl_ctx, verify and save cert                        
 |                                                                                     
 |--> [BIO_new_connect] alloc bio                                                      
 |                                                                                     
 |--> [BIO_do_connect] ask bio to connect                                              
 |                                                                                     
 |--> [SSL_new] create ssl_layer (on top of bio)                                       
 |                                                                                     
 |--> [SSL_set_bio] bios ssl_layer and bio                                             
 |                                                                                     
 |--> [SSL_connect] ask ssl_layer to connect (over bio)                                
 |                                                                                     
 |--> [netsnmp_tlsbase_verify_server_cert]                                             
 |                                                                                     
 +--> save fd of bio in 'transport'                                                    
```

```                                                                       
 snmplib/transports/snmpTLSBaseDomain.c                                
 [sslctx_client_setup] : alloc ssl_ctx, verify and save cert           
 |                                                                     
 |--> alloc and setup ssl_ctx                                          
 |                                                                     
 |--> [netsnmp_cert_find] find id cert                                 
 |                                                                     
 |--> [SSL_CTX_use_certificate] save cert in ssl_ctx                   
 |                                                                     
 |--> [SSL_CTX_use_PrivateKey] save priv_key in ssl_ctx                
 |                                                                     
 |--> [SSL_CTX_check_private_key] check if pub/priv keys are compatible
 |                                                                     
 |--> while cert exists                                                
 |    -                                                                
 |    +--> [SSL_CTX_add_extra_chain_cert] save chained cert in ssl_ctx 
 |                                                                     
 |--> [netsnmp_cert_find] find peer cert                               
 |                                                                     
 |--> if found                                                         
 |    -                                                                
 |    +--> [netsnmp_cert_trust] verify peer cert                       
 |                                                                     
 |--> if tls_base has cert                                             
 |    -                                                                
 |    +--> [_trust_this_cert]                                          
 |                                                                     
 +--> [_sslctx_common_setup]                                           
```

```                                                                      
 snmplib/transports/snmpTLSTCPDomain.c                                
 [netsnmp_tlstcp_open_server] : alloc bio and do accept, setup ssl_ctx
 |                                                                    
 |--> [BIO_new_accept] alloc bio for later acception                  
 |                                                                    
 |--> [BIO_new_accept]                                                
 |                                                                    
 |--> [sslctx_server_setup] alloc ssl_ctx, verify and save cert       
 |                                                                    
 +--> save fd of bio in 'transport'                                   
```

```                                                                                    
 agent/helpers/all_helpers.c                                                        
 [netsnmp_init_helpers] : register helpers                                          
  |                                                                                 
  |--> [netsnmp_init_debug_helper] create and register 'debug' handler              
  |                                                                                 
  |--> [netsnmp_init_serialize] create and register 'serialize' handler             
  |                                                                                 
  |--> [netsnmp_init_read_only_helper] create and register 'read_only' handler      
  |                                                                                 
  |--> [netsnmp_init_bulk_to_next_helper] create and register 'bulk_to_next' handler
  |                                                                                 
  |--> [netsnmp_init_table_dataset] register table parsers                          
  |                                                                                 
  |--> [netsnmp_init_row_merge] register 'row_merge' handler                        
  |                                                                                 
  +--> [netsnmp_register_handler_by_name] register 'stash_cache' handler            
```

```                                                                                      
 snmplib/container.c                                                                  
 [netsnmp_container_init_list] : register container handlers                          
 |                                                                                    
 |--> [netsnmp_container_get_binary_array] alloc container and setup                  
 |                                                                                    
 |--> [netsnmp_container_binary_array_init] register container handler                
 |                                                                                    
 |--> [netsnmp_container_ssll_init] register container_ssll handler                   
 |                                                                                    
 |--> [netsnmp_container_null_init] register container_null handler                   
 |                                                                                    
 |--> [netsnmp_container_register] register 'table_container' handler                 
 |                                                                                    
 |--> [netsnmp_container_register] register 'linked_list' handler                     
 |                                                                                    
 |--> [netsnmp_container_register] register 'ssll_container' handler                  
 |                                                                                    
 |--> [netsnmp_container_register_with_compare] register 'cstring' handler            
 |                                                                                    
 |--> [netsnmp_container_register_with_compare] register 'string' handler             
 |                                                                                    
 +--> [netsnmp_container_register_with_compare] register 'string_binary_array' handler
```
