# Accumulo with AAD Integration Tooling
This repo contains the extension to apache fluo-muchos to convert the generic Accumulo cluster to Azure AD integrated one with Azure AD Domain Services. The work is shared as the proof-of-concept and as it is.

## Preparation tasks before running the following tooling
After you set up Accumulo with fluo-muchos, complete the following pre-requisite tasks:
1. make FQDN changes to core-site.xml and hdfs-site.xml: i.e find the host name references and replace with fqdn host names. For example, rename aadcluster-1 to aadcluster-1.agceci.onmicrosoft.com in my experiment.
2. Then, enable the TLS on hadoop. Steps are documented in [TLSonHadoop.md](TLSonHadoop.md). If your fluo-muchos includes the internal patch (https://dev.azure.com/AZGlobal/AG%20E2E%5E2%20-%20Secure%20Data%20Estate/_git/fluo-muchos/pullrequest/2794), follow this tutorial ([domain.plus.tls.md](tutorials/domain_plus_tls/domain.plus.tls.md)) and skip last of this steps below.
3. And, you will peer the VNET of your accumulo cluster to the VNET with your AAD Domain Services. Make sure to add custom DNS servers to your VNET and reboot your VMSS before running the th following domain utility tooling. Refer to this document (https://docs.microsoft.com/en-us/azure/active-directory-domain-services/tutorial-configure-networking#configure-dns-servers-in-the-peered-virtual-network).

## Join Azure VMSS (with Centos OS) to Azure Active Directory (AAD) Domain Services & Setup SPN and keytab files
This repo contains ansible playbooks and python helper code to enable the tasks we identified in my previous article (https://github.com/sjyang18/mydocs/blob/dev/accumulo-aad/wo_hdinsight/accumulo_aad_wo_hdinsight.md). The tasks we identified in that article before reconfiguring Accumulo services with kerberos are:
1. Generate your organizational unit (OU) to hold new domain objects (computers, users, service principals) 
2. Join your cluster to your target AAD domain
3. Generate users and service principals and keytab files

To run this playbook, login into a proxy VM of accumulo cluster. Then, git clone this repo and copy the hosts file from fluo-mucho. Add custom variables with your values. Note that in the example below, we set 'accumulo' to a ldap_binddn value, which mean this user exists in my AAD and it is a member of `AAD DC Administrators`. You will be prompted for the password of this user during playbook execution.

```
[accumulomaster]
vmss25xip000000

[workers]
vmss25xip000003

[accumulo:children]
accumulomaster
workers

[zookeeper]
AAA

[namenode]
XXX

[journalnode]
YYY

[zkfc]
ZZZ

[resourcemanager]

[hadoop:children]
namenode
resourcemanager
workers
zkfc
journalnode


[all:vars]
# BLOCK OF NEW VARIABLES ADDED
#ansible_python_interpreter=/usr/bin/python3
domain_name=agceci.onmicrosoft.com
realm_name={{ domain_name |upper }}
domain_dn=DC=agceci,DC=onmicrosoft,DC=com
domain_workgroup=AGCECI
custom_ou_name=MyCustomOU
domain_admin_username=accumulo
domain_admin_full_username='{{ domain_admin_username }}@{{ realm_name }}'
ldap_host_ip_address=10.0.0.5
ldap_hostname=ldaps.agceci.onmicrosoft.com
ldap_uri=ldaps://{{ ldap_hostname }}
ldap_binddn='CN={{ domain_admin_username }},OU=AADDC Users,{{ domain_dn }}'
encoded_user_init_password=:~~~~~~~~~~~~~
decoded_user_init_password=XXXXXX
# variables copied from hosts file generated by flue-muchos setup command
install_dir=/opt/muchos/install
accumulo_version=2.1.0-SNAPSHOT
zookeeper_version = 3.5.8
accumulo_home='{{ install_dir }}/accumulo-{{ accumulo_version }}'
zookeeper_basename = {% if zookeeper_version is version('3.5', '>=') or zookeeper_version in 'SNAPSHOT' %}apache-zookeeper-{{ zookeeper_version }}-bin{% else %}zookeeper-{{ zookeeper_version }}{% endif %}
zookeeper_home = {{ install_dir }}/{{ zookeeper_basename }}
hadoop_version = 3.2.1
hadoop_home = '{{ install_dir }}/hadoop-{{ hadoop_version }}'
hadoop_major_version = {{ hadoop_version.split('.')[0] }}
```

Use `lib/genEncodedPasswd.py` to geneate your own initial encoded password for users and the corresponding SPN. Original password and encoded one would be used to create users and retrieve keytab files for spn mapped to those users.

Run the following sequence of ansible playbooks.

```
ansible-playbook -i hosts ansible/ldap_config.yml
ansible-playbook -i hosts ansible/ldap_addou.yml
ansible-playbook -i hosts ansible/join_domain.yml
ansible-playbook -i hosts ansible/ldap_adduser.yml
ansible-playbook -i hosts ansible/retreive_keytab_file.yml
# stop-dfs.sh on the first name node
# make sure to enable tls configuration before running the following. It turns out this is must in order to bring up Hadoop datanodes with Kerberos security.
ansible-playbook -i hosts ansible/hadoop-config-kerberos.yml
# start-dfs.sh and address any failures
# verify that hadoop is enabled with kerberos with 'hdfs dfs -put /etc/hosts /' and 'hdfs dfs -ls /'. 
ansible-playbook -i hosts ansible/accumulo-config-kerberos.yml 
```

# Notes

## Question: Got error when joining domain with 'Couldn't find a configured realm' error.

Answer: Your VNET, which is peered with AAD DS domain services VNET, might not be configured with custom DNS servers. Refer to this document (https://docs.microsoft.com/en-us/azure/active-directory-domain-services/tutorial-configure-networking#configure-dns-servers-in-the-peered-virtual-network) to add custom DNS servers to your VNET and reboot your VMSS. Reboot is required to take effects.


## Question: In HDFS configuration, _HOST in HTTP/_HOST@REALM_NAME are not replaced with FQDN and failed to authenticate with Kerberos.

Answer: This was observed when your namenode host names in core-site.xml and hdfs-site.xml are not FQDN names. Modify hostname(s) in hdfs-site.xml and core-site.xml with the FQDN, as described in the preparation section. You may update this in the first head node and copy to all other nodes.

## Question: My namenodes are not up when invoking start-dfs.sh. How do I recover my namenodes?

Answer: using `hadoop namenode -recover` command, you may recover namenodes. You need to first run `kinit` command to get authenticate.

## Question: Does hadoop kerberos configuration automatically refresh kerberos token at the regular interval?
Answer: My obervation is No. From the discussion with my collegue, my recommendation is to set a daily cron job to run `kdestroy && kinit` command.

## Question: I enabled Kerbero configuration on Accumulo and see "code:BAD_CREDENTIALS" error on accumulo tserver. Tserver does not accept the credential from Accumulo master.
Answer: Check if all host name references of accumulo.properties and accumulo-client.propertis use FQDN host names. For example, make sure `instance.zookeepers` and `instance.zookeeper.host` has FQDN host names.