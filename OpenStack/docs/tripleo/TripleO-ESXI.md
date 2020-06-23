## Đối với môi trường ESXI

Trên ESXI không sử dụng được IPMI nên cần xử lý việc inspect các máy ảo với driver manual-management.

- Bước 1 : Thêm cấu hình sau vào file `undercloud.conf`

enabled_hardware_types=redfish,ipmi,idrac,ilo,manual-management

- Bước 2 : Update lại node Undercloud với câu lệnh

```sh
openstack undercloud install
```

- Bước 3 : Thêm các cấu hình sau vào file /var/lib/config-data/ironic/etc/ironic/ironic.conf

```sh
enabled_hardware_types=idrac,ilo,ipmi,manual-management,redfish
enabled_drivers=pxe_drac,pxe_ilo,pxe_ipmitool,fake
```

- Bước 4 : Restart lại dịch vụ ironic

```sh
systemctl restart  tripleo_ironic_api.service                                                            
systemctl restart  tripleo_ironic_conductor.service                                                      
systemctl restart  tripleo_ironic_inspector.service                                                      
systemctl restart  tripleo_ironic_inspector_dnsmasq.service                                              
systemctl restart  tripleo_ironic_neutron_agent.service                                                  
systemctl restart  tripleo_ironic_pxe_http.service                                                       
systemctl restart  tripleo_ironic_pxe_tftp.service  
```

- Bước 5 : Tại file JSON thêm cấu hình `pm_type` như sau :

```sh
        {
          "name": "ceph03",
          "memory": "8192",
          "mac": [
            "00:50:56:aa:83:96"
          ],
          "pm_type": "manual-management",
          "disk": "100",
          "arch": "x86_64",
          "cpu": "8"
        }
```

