# Tài liệu triển khai Openstack Rocky sử dụng Kolla
---
## Mô hình

Cấu hình tối thiểu AIO
- CPU >= 6
- Ram >= 8GB
- Disk 2 ổ:
  - vda - OS (>= 50 GB)
  - vdb - Cinder LVM (>= 50 GB)

Sử dụng 3 Card mạng:
- eth0: Dải Pulic API + SSH
- eth1: Dải Provider VM (Không có IP)
- eth2: Dải Internal, Admin API (Không thể SSH, dùng dải NAT KVM)

## Cài đặt

### Phần 1: Chuẩn bị

Cấu hình Network

```
echo "Setup IP eth0"
nmcli c modify eth0 ipv4.addresses 10.10.30.51/24
nmcli c modify eth0 ipv4.gateway 10.10.30.1
nmcli c modify eth0 ipv4.dns 8.8.8.8
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes

cat << EOF > /etc/sysconfig/network-scripts/ifcfg-eth1
TYPE=Ethernet
BOOTPROTO=none
NAME=eth1
DEVICE=eth1
ONBOOT=yes
EOF

echo "Setup IP eth2"
nmcli c modify eth2 ipv4.addresses 10.10.32.51/24
nmcli c modify eth2 ipv4.method manual
nmcli con mod eth2 connection.autoconnect yes
```

Tắt Firewall, SELinux
```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld
```

Update hệ điều hành
```
yum install -y epel-release
yum update -y
```

Cấu hình đồng bộ thời gian
```
timedatectl set-timezone Asia/Ho_Chi_Minh

yum -y install chrony
sed -i 's/server 0.centos.pool.ntp.org iburst/ \
server 1.vn.pool.ntp.org iburst \
server 0.asia.pool.ntp.org iburst \
server 3.asia.pool.ntp.org iburst/g' /etc/chrony.conf
sed -i 's/server 1.centos.pool.ntp.org iburst/#/g' /etc/chrony.conf
sed -i 's/server 2.centos.pool.ntp.org iburst/#/g' /etc/chrony.conf
sed -i 's/server 3.centos.pool.ntp.org iburst/#/g' /etc/chrony.conf

systemctl enable chronyd.service
systemctl restart chronyd.service
chronyc sources
```

Cài đặt môi trường Python
```
yum install -y git wget gcc python-devel python-pip yum-utils byobu vim 
yum install -y libffi-devel openssl-devel libselinux-python
```

Cài đặt PIP
```
sudo pip install pip==9.0.0
```

Lưu ý:
- Không cài bản `pip` mới nhất, dẫn tới không thể cài kolla-ansible

Cài đặt lvm
```sh 
yum install lvm2 -y
```

Tạo phân vùng Cinder LVM
```
pvcreate /dev/vdb 
vgcreate cinder-volumes /dev/vdb
```

Khởi động lại, tại bước này nên Snapshot VM
```
init 6
```

### Phần 2: Cài đặt Ansible

Cài đặt Ansible
```
pip install ansible==2.5
```

Lưu ý:
- Không cài đặt bản Ansible mới nhất, sẽ lỗi trong quá trình cài Kolla (https://github.com/geerlingguy/ansible-role-varnish/issues/90)

cấu hình Ansible
```
mkdir -p /etc/ansible
cd /etc/ansible
cp ansible.cfg ansible.cfg.bak

cat << EOF > /etc/ansible/ansible.cfg
[defaults]
host_key_checking=False
pipelining=True
forks=100
[inventory]
[privilege_escalation]
[paramiko_connection]
[ssh_connection]
[persistent_connection]
[accelerate]
[selinux]
[colors]
[diff]
EOF
```

### Phần 3: Triển khai Kolla

Cài đặt Kolla từ source github
```
cd ~
git clone https://github.com/openstack/kolla.git -b stable/rocky
git clone https://github.com/openstack/kolla-ansible.git -b stable/rocky

cd ~
pip install ./kolla
cd kolla
python setup.py install

cd ~
pip install ./kolla-ansible
cd kolla-ansible
python setup.py install

sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
```

Lưu ý:
- Yêu cầu Kolla từ `source github`. Sử dụng gói trên `pypi` có dẫn tới lỗi do version không tương thích.
- Lưu ý: Phải sử dụng github source tương ứng phiên bản Openstack muốn cài (queen => stable/queen, rocky => stable/rocky)

Cấu hình dịch vụ Kolla
```
cp -r /usr/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/
```

Kiểm tra
```
$ ls /etc/kolla/
globals.yml passwords.yml
```

Cấu hình file Inventory ansible 
```
cd /opt/
cp /usr/share/kolla-ansible/ansible/inventory/* .
```
Kiểm tra
```
$ ls /opt/
all-in-one  multinode
```
Lưu ý:
- `all-in-one`: Triển khai Kolla Openstack AIO
- `multinode`: Triển khai Kolla Openstack Multi node

Sinh mật khẩu mặc định cho service Openstack
```
kolla-genpwd
cat /etc/kolla/passwords.yml
```
Lưu ý:
- File `/etc/kolla/passwords.yml` sẽ chứa mật khẩu các dịch vụ Openstack

Cấu hình Kolla Openstack Rocky
```
cp /etc/kolla/globals.yml /etc/kolla/globals.yml.bak
cat << EOF > /etc/kolla/globals.yml
---
kolla_base_distro: "centos"
kolla_install_type: "source"
openstack_release: "rocky"

# Không sử dụng HA Controller (VIP RabbitMQ, MariaDB v.v)
# enable_haproxy: "no"

# Dải Mngt + admin, internal API
kolla_internal_vip_address: "10.10.32.50"
network_interface: "eth2"

# Dải Mngt Provider
neutron_external_interface: "eth1"

# Dải external (dành riêng API Public)
kolla_external_vip_address: "10.10.30.50"
kolla_external_vip_interface: "eth0"

# cat /proc/cpuinfo | grep vmx | wc -l 
# Nếu > 0 có thể dùng KVM 
nova_compute_virt_type: "qemu"

enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
enable_cinder_backup: "no"

glance_enable_rolling_upgrade: "no"
ironic_dnsmasq_dhcp_range:
tempest_image_id:
tempest_flavor_ref_id:
tempest_public_network_id:
tempest_floating_network_name:
EOF
```
Lưu ý:
- Nếu sử dụng `enable_haproxy: no` có thể dẫn tới 1 số lỗi không mong muốn, tốt nhất nên sử dụng HAProxy và sử dụng IP VIP

### Phần 4: Cài đặt Openstack Rocky bằng Kolla

Khởi tạo môi trường (Docker, các repo sử dụng khi cài Openstack Rocky)
```
cd /opt/
kolla-ansible -i ./all-in-one bootstrap-servers
```

Kiểm tra cấu hình trước khi triển khai
```
kolla-ansible -i ./all-in-one prechecks
```

Cài đặt Openstack Rocky bằng Kolla
```
kolla-ansible -i ./all-in-one deploy
```

Lưu ý:
- Sau khi cài đặt tất cả trạng thái đều là `OK` => Cài đặt thành công

VD:
```
PLAY RECAP *****************************************************************
localhost                  : ok=294  changed=180  unreachable=0    failed=0   

```

Sinh file quản trị Openstack
```
cd /opt/
kolla-ansible -i ./all-in-one post-deploy
```

Bổ sung file `admin-openrc`
```
cd /root

keystone_admin_password=$(cat /etc/kolla/passwords.yml | grep keystone_admin_password | awk '{print $2}')


cat << EOF> /etc/kolla/admin-openrc.sh
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=$keystone_admin_password
export OS_AUTH_URL=http://10.10.32.50:35357/v3
export OS_INTERFACE=internal
export OS_IDENTITY_API_VERSION=3
export OS_REGION_NAME=RegionOne
export OS_AUTH_PLUGIN=password
EOF

ln -s /etc/kolla/admin-openrc.sh /root
```

Kiểm tra file
```sh 
cat /etc/kolla/admin-openrc.sh
cat /root/admin-openrc.sh
```

Cài đặt openstack client
Lưu ý:
- Không nên cài đặt openstack client trực tiếp trên Node Openstack AIO ==> Ảnh hưởng tới môi trường Python, và có thể dẫn tới 1 số lỗi không mong muốn
- Issue: https://ask.openstack.org/en/question/120582/importerror-cannot-import-name-decorate/
- Cách fix => Dùng 1 trong 2 cách bên dưới

#### 2 cách fix openstack client
##### Cách 1: Cài đặt Openstack client tại 1 node CentOS khác
```
yum install centos-release-openstack-rocky -y
yum install python-openstackclient -y
```

Lưu ý: `KHÔNG CÀI ĐẶT TẠI NODE OPENSTACK AIO` mà là một node mới hoàn toàn 

##### Cách 2: Cài đặt Openstack client tại virtualenv, bảo đảm verion `pip` là mới nhất

Thực hiện
```
pip install virtualenv
virtualenv openstack-client
source openstack-client/bin/activate
```

Kiểm tra
```
(openstack-client) [root@kolla ~]# pip --version
pip 19.3.1 from /root/openstack-client/lib/python2.7/site-packages/pip (python 2.7)
```

Cài đặt Openstack client
```
(openstack-client) [root@kolla ~]# pip install python-openstackclient
```

Kiểm tra
```
(openstack-client) [root@kolla ~]# source admin-openrc.sh

(openstack-client) [root@kolla ~]# openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2019-12-17T04:26:24+0000                                                                                                                                                                |
| id         | gAAAAABd9wdwJ4X0RgnU3Y22nidxNjH-1D5BdP2jH7r_oyPjEz8ZulkQP373pKAvlgDbc4ZvBr8EiMIqmeSuPoVPzkRTz7WIXKxFQZ-5HmSYEaoe1sJ8QN-JXTd3P3Dk2c5O0wf4gkz9L1Fu5wJttvfvRV3QcDF5F5DbAoxDTaz6tc6wP0LW_o4 |
| project_id | b1bc524f62154097a1e8e9bd4fa018a1                                                                                                                                                        |
| user_id    | 7b164685047d48d79f1c3be863f274d4                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

> Nếu lõi 
```sh 
(openstack-client) [root@kolla ~]# openstack token issue 
Traceback (most recent call last):
  File "/root/openstack-client/bin/openstack", line 5, in <module>
    from openstackclient.shell import main
  File "/root/openstack-client/lib/python2.7/site-packages/openstackclient/shell.py", line 24, in <module>
    from osc_lib import shell
  File "/root/openstack-client/lib/python2.7/site-packages/osc_lib/shell.py", line 33, in <module>
    from osc_lib.cli import client_config as cloud_config
  File "/root/openstack-client/lib/python2.7/site-packages/osc_lib/cli/client_config.py", line 18, in <module>
    from openstack.config import exceptions as sdk_exceptions
  File "/root/openstack-client/lib/python2.7/site-packages/openstack/__init__.py", line 16, in <module>
    import openstack.config
  File "/root/openstack-client/lib/python2.7/site-packages/openstack/config/__init__.py", line 17, in <module>
    from openstack.config.loader import OpenStackConfig  # noqa
  File "/root/openstack-client/lib/python2.7/site-packages/openstack/config/loader.py", line 33, in <module>
    from openstack.config import cloud_region
  File "/root/openstack-client/lib/python2.7/site-packages/openstack/config/cloud_region.py", line 44, in <module>
    from openstack import proxy
  File "/root/openstack-client/lib/python2.7/site-packages/openstack/proxy.py", line 24, in <module>
    from openstack import resource
  File "/root/openstack-client/lib/python2.7/site-packages/openstack/resource.py", line 49, in <module>
    from openstack import utils
  File "/root/openstack-client/lib/python2.7/site-packages/openstack/utils.py", line 13, in <module>
    import queue
ImportError: No module named queue
(openstack-client) [root@kolla ~]# 
```

Xử lý như sau 

- Chỉnh sửa file `/root/openstack-client/lib/python2.7/site-packages/openstack/utils.py` dòng 13
- Chỉnh sửa file `/root/openstack-client/lib/python2.7/site-packages/openstack/cloud/openstackcloud.py` dòng 14 

Nội dung chỉnh sửa đổi từ `import queue` thành `import Queue as queue`
```sh 
#import queue
import Queue as queue
```

Kiểm tra lại cho đến khi get được token thì thành công 


#### Tiến hành up Images cirros lên OPS
```
cd /root
source openstack-client/bin/activate
source admin-openrc.sh
/usr/share/kolla-ansible/init-runonce
```

### 1 số vấn đề có thể xuất hiện khi cài Cinder

Log cinder-volume báo lỗi `2019-12-15 13:08:35.753 7 ERROR cinder.cmd.volume [-] Configuration for cinder-volume does not specify "enabled_backends". Using DEFAULT section to configure drivers is not supported since Oc`
- Nguyên nhân: Do thiếu config `enable_cinder_backend_lvm` trong file `/etc/kolla/globals.yml`
- Thêm, thực hiện câu lệnh: `kolla-ansible -i ./all-in-one deploy`

Không tạo được VM (nhưng tạo được volume)
- Lỗi do thiết `tgtd` container. LOG: http://paste.openstack.org/show/787596/
- Kiểm tra `docker ps -a` check `tgtd` container, Kết quả http://paste.openstack.org/show/787603/
- Nếu không có `tgtd` container thực hiện câu lệnh `kolla-ansible -i ./all-in-one deploy`

Log cinder-volume báo `2019-12-15 13:46:32.605 37 WARNING cinder.volume.manager [req-3257af8c-e1cc-443d-9325-71266a04a205 - - - - -] Update driver status failed: (config name lvm-1) is uninitialized.`
- Check volume lvm đã tạo hay chưa (lvs, vgs, v.v)
- Thực thiện câu lệnh `kolla-ansible -i ./all-in-one deploy`
- Reboot máy ảo, nếu xuất hiện `cinder--volumes-cinder--volumes--pool_tmeta` và `cinder--volumes-cinder--volumes--pool_tdata` thành công. LOG: http://paste.openstack.org/show/787615/

### Lưu ý:
2 câu lệnh ý nghĩa gần giống nhau

Câu lệnh `kolla-ansible -i ./all-in-one reconfigure` sửa dụng khi
- Thay đổi config nhỏ trong dịch vụ Nova, Cinder

Câu lệnh `kolla-ansible -i ./all-in-one deploy` sửa dụng khi
- Sử dụng khi muốn sử dụng Kolla Deploy 1 service hoặc 1 project mới trong cụm Openstack

### Tài liệu tham khảo

Cài đặt Kolla Openstack Pike
- https://github.com/congto/hdsd-openstack-kolla/blob/master/docs/openstack-pike-kolla-aio.md

Cấu hình Network Kolla
- https://docs.openstack.org/kolla-ansible/latest/admin/advanced-configuration.html#endpoint-network-configuration

Cinder LVM
- https://docs.openstack.org/kolla-ansible/queens/reference/cinder-guide.html

Troubleshotting
- https://docs.openstack.org/kolla-ansible/latest/user/troubleshooting.html
