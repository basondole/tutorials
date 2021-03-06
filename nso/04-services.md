# Services
### Creating service for deploying radius server
Creating an Yang Module using NSO Bash commands
```
basondole@netbox:~/nso/ncs-run/packages$ ncs-make-package --service-skeleton template simple_radius --augment /ncs:services
basondole@netbox:~/nso/ncs-run/packages$ cd simple_radius/
basondole@netbox:~/nso/ncs-run/packages/simple_radius$ nano src/yang/simple_radius.yang
basondole@netbox:~/nso/ncs-run/packages/simple_radius$ cat src/yang/simple_radius.yang
module simple_radius {
  namespace "http://com/example/simple_radius";
  prefix simple_radius;

  import ietf-inet-types {
    prefix inet;
  }
  import tailf-ncs {
    prefix ncs;
  }
  <b>import tailf-common { prefix tailf; }</b>

  augment /ncs:services {
  list simple_radius {
    tailf:info "deploy radius server config for iosrx nxos junos";
    key server-ip;

    uses ncs:service-data;
    ncs:servicepoint "simple_radius";

    leaf server-ip {
      tailf:info "ipv4 address of the radius server";
      type inet:ipv4-address;
    }

    leaf-list device {
      tailf:info "device to deploy the service to";
      type leafref {
        path "/ncs:devices/ncs:device/ncs:name";
      }
    }

    leaf-list secret {
      tailf:info "secret key";
      type string;
    }

  }
  } // augment /ncs:services {
}
basondole@netbox:~/nso/ncs-run/packages/simple_radius$ cd src/
basondole@netbox:~/nso/ncs-run/packages/simple_radius/src$ make
/home/basondole/nso/nso-5.1.0.1/bin/ncsc  `ls simple_radius-ann.yang  > /dev/null 2>&1 && echo "-a simple_radius-ann.yang"` \
              -c -o ../load-dir/simple_radius.fxs yang/simple_radius.yang
basondole@netbox:~/nso/ncs-run/packages/simple_radius/src$
```

### Creating xml template
We will create device template based on the actual CLI configuration.
Below are the configuration options
- server ip
- authentication port
- accounting port
- key  

We configure this on the NSO and then we produce the xml formatted config.
```
basondole@netbox:~/ncs-run/packages/simple_radius/src$ ncs_cli -C
basondole@ncs# config
Entering configuration mode terminal
basondole@ncs(config)# devices device pycon-iosxr config cisco-ios-xr:radius-server host 10.10.1.100 auth-port 1812 acct-port 1813
basondole@ncs(config-radius-host)# commit dry-run outformat xml
result-xml {
    local-node {
        data <devices xmlns="http://tail-f.com/ns/ncs">
               <device>
                 <name>pycon-iosxr</name>
                 <config>
                   <radius-server xmlns="http://tail-f.com/ned/cisco-ios-xr">
                     <host>
                       <id>10.10.1.100</id>
                       <auth-port>1812</auth-port>
                       <acct-port>1813</acct-port>
                     </host>
                   </radius-server>
                 </config>
               </device>
             </devices>
    }
}
basondole@ncs(config-radius-host)# end
Uncommitted changes found, commit them? [yes/no/CANCEL] no
basondole@ncs#basondole@ncs# exit
```

We repeat the above procedure to get xml config for all our devices `ios` `iosxr` `nxos` and `junos`
then we edit the xml template by adding the xml config from above process

```
basondole@netbox:~/nso/ncs-run/packages/simple_radius/src$ cd ../
basondole@netbox:~/nso/ncs-run/packages/simple_radius$ nano templates/simple_radius-template.xml
basondole@netbox:~/nso/ncs-run/packages/simple_radius$ cat templates/simple_radius-template.xml
<config-template xmlns="http://tail-f.com/ns/config/1.0"
                 servicepoint="simple_radius">
  <devices xmlns="http://tail-f.com/ns/ncs">
    <device>
      <!--
          Select the devices from some data structure in the service
          model. In this skeleton the devices are specified in a leaf-list.
          Select all devices in that leaf-list:
      -->
      <name>{/device}</name>
      <config>
        <!--
            Add device-specific parameters here.
        -->
 
        <radius-server xmlns="http://tail-f.com/ned/cisco-nx">
          <host>
            <id>{/server-ip}</id>
            <key>
              <encryption>0</encryption>
              <password>{/secret}</password>
            </key>
          </host>
        </radius-server>

        <radius-server xmlns="http://tail-f.com/ned/cisco-ios-xr">
          <host>
            <id>{/server-ip}</id>
            <auth-port>1812</auth-port>
            <acct-port>1813</acct-port>
            <key>
              <key-container>
                <key>{/secret}</key>
              </key-container>
            </key>
          </host>
        </radius-server>

        <configuration xmlns="http://xml.juniper.net/xnm/1.1/xnm">
          <system>
            <radius-server>
              <name>{/server-ip}</name>
              <port>1812</port>
              <accounting-port>1813</accounting-port>
              <secret>{/secret}</secret>
            </radius-server>
          </system>
        </configuration>

      </config>
    </device>
  </devices>
</config-template>
```

Adding the service to nso
<pre>
basondole@netbox:~/ncs-run/packages/simple_radius$ ncs_cli -C

basondole connected from 192.168.56.1 using ssh on netbox
<b>basondole@ncs# packages reload</b>

>>> System upgrade is starting.
>>> Sessions in configure mode must exit to operational mode.
>>> No configuration changes can be performed until upgrade has completed.
>>> System upgrade has completed successfully
.
.
reload-result {
    package simple_radius
    result true
}
</pre>

Deploying the service on the router

<pre>
admin@ncs# conf
Entering configuration mode terminal
admin@ncs(config)# services simple_radius 10.10.1.100 device netsim-junos-00 secret fisi321
admin@ncs(config-simple_radius-10.10.1.100)# exit
admin@ncs(config)# services simple_radius 10.10.1.100 device netsim-nx-00 secret fisi321
admin@ncs(config-simple_radius-10.10.1.100)# exit
admin@ncs(config)# services simple_radius 10.10.1.100 device netsim-xr-00 secret fisi321
admin@ncs(config-simple_radius-10.10.1.100)# exit
admin@ncs(config)#
admin@ncs(config)# commit dry-run
cli {
    local-node {
        data  devices {
                  device netsim-junos-00 {
                      config {
                          junos:configuration {
                              system {
             +                    # first
             +                    radius-server 10.10.1.100 {
             +                        port 1812;
             +                        accounting-port 1813;
             +                        secret fisi321;
             +                    }
                              }
                          }
                      }
                  }
                  device netsim-nx-00 {
                      config {
                          nx:radius-server {
             +                host 10.10.1.100 {
             +                    key {
             +                        encryption 0;
             +                        password fisi321;
             +                    }
             +                }
                          }
                      }
                  }
                  device netsim-xr-00 {
                      config {
                          cisco-ios-xr:radius-server {
             +                host 10.10.1.100 {
             +                    auth-port 1812;
             +                    acct-port 1813;
             +                    key {
             +                        key-container {
             +                            key fisi321;
             +                        }
             +                    }
             +                }
                          }
                      }
                  }
              }
              services {
             +    simple_radius 10.10.1.100 {
             +        device [ netsim-junos-00 netsim-nx-00 netsim-xr-00 ];
             +    }
              }
    }
}
admin@ncs(config)# commit
Commit complete.
admin@ncs(config)#
</pre>

## Verification

<pre>
basondole@netbox:~/nso/ncs-run/packages/simple_radius$ ncs_cli -u admin

admin connected from 192.168.56.1 using ssh on netbox
admin@ncs> configure
[edit]
admin@ncs% show services simple_radius | tab

SERVER IP    DEVICE                                         SECRET
-------------------------------------------------------------------------
10.10.1.100  [ netsim-junos-00 netsim-nx-00 netsim-xr-00 ]  [ fisi321 ]

[ok][2019-12-28 18:31:01]

[edit]
</pre>

Switching to the `netsim` directory so as to access the emulated devices  
`basondole@netbox:~/nso/ncs-run/packages/simple_radius$ cd ../../netsim/`

### ios xr
<pre>
basondole@netbox:~/nso/ncs-run/netsim$ ncs-netsim cli-i netsim-xr-00

admin connected from 192.168.56.1 using ssh on netbox
netbox> enable
netbox# show run
logging 10.10.1.100
domain name basondole.org
radius-server host 10.10.1.100 auth-port 1812 acct-port 1813
 key fisi321
exit
ntp
 server 10.10.1.100
exit
username fisi
 privilege 15
exit
netbox# exit
</pre>

### nx os
<pre>
basondole@netbox:~/nso/ncs-run/netsim$ ncs-netsim cli-i netsim-nx-00

admin connected from 192.168.56.1 using ssh on netbox
netbox> enable
netbox# show run
no feature ssh
no feature telnet
username fisi password fisi123
ip domain-name basondole.org
!
ntp server 10.10.1.100
logging server 10.10.1.100
radius-server host 10.10.1.100 key 0 fisi321
netbox# exit
</pre>

### junos
<pre>
basondole@netbox:~/nso/ncs-run/netsim$ ncs-netsim cli netsim-junos-00

admin connected from 192.168.56.1 using ssh on netbox
admin@netsim-junos-00>show configuration configuration system
host-name   netsim-junos-00;
domain-name basondole.org;
radius-server 10.10.1.100 {
    port            1812;
    accounting-port 1813;
    secret          fisi321;
}
login {
    user fisi {
        class super-user;
        authentication {
            plain-text-password-value fisi123;
        }
    }
}
syslog {
    host 10.10.1.100;
}
ntp {
    server 10.10.1.100;
}
[ok][2019-12-28 17:54:53]
admin@netsim-junos-00>exit
basondole@netbox:~/nso/ncs-run/netsim$
</pre>


### Using the nso with junos-like cli to deploy service
<pre>
basondole@netbox:~/nso/ncs-run/netsim$ ncs_cli -u admin

admin connected from 192.168.56.1 using ssh on netbox
admin@ncs> configure
Entering configuration mode private
[ok][2019-12-28 18:22:15]

[edit]
admin@ncs% set services simple_radius 10.1.1.1 device pycon-iosxr secret fisi123
[ok][2019-12-28 18:23:36]

[edit]
admin@ncs% show | compare
 services {
+    simple_radius 10.1.1.1 {
+        device [ pycon-iosxr ];
+        secret [ fisi123 ];
+    }
 }
[ok][2019-12-28 18:23:42]

[edit]
admin@ncs% commit check
Validation complete
[ok][2019-12-28 18:24:01]

[edit]
admin@ncs% commit

[edit]
admin@ncs%
</pre>

### Un-deploying and re-ploying service
Un-deploying the service

<pre>
basondole@netbox:~/nso/ncs-run/netsim$ ncs_cli -u admin

admin connected from 192.168.56.1 using ssh on netbox
admin@ncs> configure
Entering configuration mode private
[ok][2019-12-28 19:09:51]

[edit]
admin@ncs% edit services simple_radius 10.10.1.100
[ok][2019-12-28 19:09:56]

[edit services simple_radius 10.10.1.100]
admin@ncs% <b>request un-deploy</b>
[ok][2019-12-28 19:10:05]

[edit services simple_radius 10.10.1.100]
admin@ncs% request check-sync
<b>in-sync false</b>
[ok][2019-12-28 19:10:11]
</pre>

Re-deploying the service
<pre>
[edit services simple_radius 10.10.1.100]
admin@ncs% <b>request re-deploy</b>
[ok][2019-12-28 19:10:19]

[edit services simple_radius 10.10.1.100]
admin@ncs%
System message at 2019-12-28 19:10:19...
Commit performed by admin via ssh using cli.
admin@ncs% request check-sync
<b>in-sync true</b>
[ok][2019-12-28 19:10:22]

[edit services simple_radius 10.10.1.100]
admin@ncs%
</pre>
