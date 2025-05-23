### 1、將RHEL ISO上傳至bastion中
```
[root@bastion ~]# mkdir /var/repo

[root@bastion ~]# mount -o loop rhel-baseos-9.1-x86_64-dvd.iso /var/repo/
mount: /var/repo: WARNING: source write-protected, mounted read-only.
```

### 2、設定Repository
```
[root@bastion ~]#
cat > /etc/yum.repos.d/rhel9-local.repo << EOF
[Local-BaseOS]
name=Red Hat Enterprise Linux 9 - BaseOS
metadata_expire=-1
gpgcheck=1
enabled=1
baseurl=file:///var/repo//BaseOS/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
[Local-AppStream]
name=Red Hat Enterprise Linux 9 - AppStream
metadata_expire=-1
gpgcheck=1
enabled=1
baseurl=file:///var/repo//AppStream/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF

[root@bastion ~]# yum clean all
Updating Subscription Management repositories.
Unable to read consumer identity
This system is not registered with an entitlement server. You can use subscription-manager to register.
0 files removed

[root@bastion ~]# subscription-manager clean
All local data removed

[root@bastion ~]# sed -i 's/enabled=1/enabled=0/' /etc/yum/pluginconf.d/subscription-manager.conf
```

### 3、關閉防火牆
```
[root@bastion ~]# systemctl stop firewalld

[root@bastion ~]# systemctl disable firewalld
Removed "/etc/systemd/system/multi-user.target.wants/firewalld.service".
Removed "/etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service".

[root@bastion ~]# setenforce 0

[root@bastion ~]# sed -i 's/enforcing/disabled/' /etc/selinux/config
```

### 4、安裝Bind
```
[root@bastion ~]# yum install bind bind-utils -y
Red Hat Enterprise Linux 9 - BaseOS
.......略........
Complete!

[root@bastion ~]# vi /etc/named.conf
options {
        listen-on port 53 { any; };
#       listen-on-v6 port 53 { ::1; };
        allow-query     { any; };
        forwarders { 8.8.8.8; };
        dnssec-enable no;
        dnssec-validation no;
zone "ocp.andy.com" IN {
        type master;
        file "named.sno.andy.com";
};

zone "25.12.172.in-addr.arpa" IN {
        type master;
        file "rev.25.12.172";
};
```
```
[root@bastion ~]# vi /var/named/named.sno.andy.com
$TTL 1D
@       IN SOA  @ bastion.sno.andy.com. (
                                        2020040819      ; serial
                                        3H      ; refresh
                                        15M     ; retry
                                        1W      ; expire
                                        1D )    ; minimum
@       IN NS   bastion.sno.andy.com.
@       IN A    172.12.25.41
bastion         IN      A       172.12.25.41
master          IN      A       172.12.25.51
api             IN      A       172.12.25.51
*.apps          IN      A       172.12.25.51
```

```
[root@bastion ~]# vi /var/named/rev.25.12.172
$TTL 1D
@       IN SOA  @ bastion.sno.andy.com. (
                                        2020040819      ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
@       IN NS   bastion.sno.andy.com.
25.12.172.in-addr.arpa      IN      PTR     bastion.sno.andy.com
51   IN      PTR     bastion.sno.andy.com.
51   IN      PTR     master.sno.andy.com.
51   IN      PTR     api.sno.andy.com.
51   IN      PTR     apps.sno.andy.com.
```
```
[root@bastion ~]# chgrp named /var/named/named.sno.andy.com

[root@bastion ~]# chmod 640 /var/named/named.sno.andy.com

[root@bastion ~]# chgrp named /var/named/rev.25.12.172

[root@bastion ~]# chmod 640 /var/named/rev.25.12.172

[root@bastion ~]# systemctl enable named
Created symlink /etc/systemd/system/multi-user.target.wants/named.service → /usr/lib/systemd/system/named.service.

[root@bastion ~]# systemctl start named

[root@bastion ~]# systemctl status named
● named.service - Berkeley Internet Name Domain (DNS)
     Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
     Active: active (running) since Fri 2025-05-09 16:42:12 CST; 4s ago
    Process: 36786 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMED>
    Process: 36789 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
.......略........
```

### 5、產生SSH
```
[root@bastion ~]# ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa
Generating public/private rsa key pair.
Created directory '/root/.ssh'.
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:rW2mrwax/1k3Fh6R20SyeM+mGeYVeMc+9CX2sj720bM root@bastion
The key's randomart image is:
+---[RSA 4096]----+
|              . .|
|             ..* |
|            ..B+=|
|      .  .   oo@=|
|       oS .   B.X|
|      o  o   + @o|
|       o. + . Xo.|
|        o+ o +o.+|
|       .o++  ..E.|
+----[SHA256]-----+

[root@bastion ~]# eval "$(ssh-agent -s)"
Agent pid 36820

[root@bastion ~]# ssh-add ~/.ssh/id_rsa
Identity added: /root/.ssh/id_rsa (root@bastion)

[root@bastion ~]# cat /root/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDM2xUFcTq2jLrf8/K7x0sryZKfDPgUiu/E4sT
.......略........
```


### 6、建置sno * 1
> 需紀錄網卡 Mac address及設定每台VM EnableUUID（如下）
```
1.Open the Host Client, and log in to the ESXi.
2.Locate the Windows Server 2016 virtual machine for which you are enabling the disk UUID attribute, and power off the virtual machine.
3.After power-off, right-click the virtual machine, and choose Edit Settings.
4.Click VM Options tab, and select Advanced.
5.Click Edit Configuration in Configuration Parameters.
6.Click Add parameter.
7.In the Key column, type disk.EnableUUID.
8.In the Value column, type TRUE.
9.Click OK and click Save.
```

### 7、設定Redhat Console-1
> https://console.redhat.com/
```
1.Create Clasrer
2.Select "Datacenter" and "Bare Metal (x86_64)"
3.Select "Interactive"
4.Enter Cluster name / Base domain(網域名稱) / OpenShift version：4.18.11 / Number of control plane nodes：1 (Single Node OpenShift) / Host network configuration：Static IP
5.Enter DNS / Machine network(網段非IP) / Default gateway
6.Enter VM Mac address
7.Not selected Operators
8.Select "Add Host" - "Full Image" / Paste SSH public key / Generate Discovery ISO - Download Discovery ISO
9.Start VM using ISO
10.Next至 "Install cluster"
```

> 需等待約40分鐘，完成後需紀錄密碼

### 8、驗證
> 設定DNS為bastion
```
連至https://console-openshift-console.apps.sno.andy.com/ (sno：Cluster name / andy.com：Base domain)
檢查"Compute" - "Nodes"，是否有一個MAster、Worker，並確認Status為Ready
```

### 9、OC Tool installation
> GUI右上角"Command Line Tools"，下載"Download oc for Linux for x86_64"，上傳至bastion
```
[root@bastion ~]# tar xvf oc.tar
oc

[root@bastion ~]# cp oc /usr/bin/

[root@bastion ~]# oc version
Client Version: 4.18.0-202504151633.p0.geb9bc9b.assembly.stream.el9-eb9bc9b
Kustomize Version: v5.4.2

[root@bastion ~]# oc login api.sno.andy.com:6443 -u kubeadmin -p MteEJ-Efzfh-74tZU-nbKVx
The server uses a certificate signed by an unknown authority.
You can bypass the certificate check, but any data you send to the server could be intercepted by others.
Use insecure connections? (y/n): y
WARNING: Using insecure TLS client config. Setting this option is not supported!
Login successful.
You have access to 74 projects, the list has been suppressed. You can list all projects with 'oc projects'
Using project "default".
Welcome! See 'oc help' to get started.

[root@bastion ~]# oc get node
NAME   STATUS   ROLES                         AGE   VERSION
api    Ready    control-plane,master,worker   37m   v1.31.7
```

### 10、備註
單機版安裝Unity CSI需將values.yaml中，改controllerCount: 1。
