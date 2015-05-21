# -*- mode: ruby -*-
# vi: set ft=ruby :

# <<< Configuration >>>
# processor arch bits (32/64)
config_arch=32
# virtual machine memory
vm_memory=2048
# virtual machine cpu 
vm_cpus=2
# openstack version
os_version='stable/juno'
# components to install
os_components=[
    'neutron',
#    'swift',
#    'security_groups',
#    'tempest',
    'trove',
#    'ceilometer',
#    'designate',
#    'designate_agent',
#    'designate_sink',
#    'neutron_lbaas',
#    'neutron_vpn',
#    'neutron_fwaas',
#    'ceph',
]
# vnc keymap
vnc_keymap='en-us'
# trove databse
#trove_db='mysql-5.6'
trove_db='postgresql-9.3'
# networks for vm
# old=devstack-vm original, 3 network cards
# one=only one network card + forwarding port
vm_network='one'
# <<< Configuration>>>

#using proxy?
#http_proxy=http://user:pass@hote:port
#https_proxy=http://user:pass@hote:port
#no_proxy=127.0.0.1,localhost,192.168.27.100
#vagrant plugin install vagrant-proxyconf

Vagrant.configure("2") do |config|


 if Vagrant.has_plugin?("vagrant-proxyconf")
    config.proxy.http    = ENV['http_proxy'] if  ENV['http_proxy']!=nil
    config.proxy.https    = ENV['https_proxy'] if  ENV['https_proxy']!=nil
    config.proxy.no_proxy = ENV['no_proxy'] if ENV['no_proxy']!=nil
  end
    
    # if you have a vagrant-cachier plugin, use it for apt caches
    if Vagrant.has_plugin?("vagrant-cachier")
      config.cache.scope = :box
    end

    if config_arch==32
        cirros_arch='i386'
    else
        cirros_arch='x86_64'
    end

    components_params=''
    os_components.each do |val|
     components_params+=" -e #{val}=True"
    end
    
    trove_db_name=trove_db.split('-')[0]
    trove_db_version=trove_db.split('-')[1]
    
    config.vm.box = "ubuntu/trusty#{config_arch}"

    # Ubuntu has a "bug" (https://github.com/mitchellh/vagrant/issues/1673)
    config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"  

    config.vm.hostname = 'devstack'
    
    if vm_network == 'one' then
      public_ip='127.0.0.1'
      config.vm.network "forwarded_port", guest: 80, host: 8080
      config.vm.network "forwarded_port", guest: 6080, host: 6080
      config.vm.network "forwarded_port", guest: 6083, host: 6083
    else
     public_ip='192.168.27.100'
     # eth1, this will be the endpoint
     config.vm.network :private_network, ip: "192.168.27.100"
     # eth2, this will be the OpenStack "public" network, use DevStack default
     config.vm.network :private_network, ip: "172.24.4.225", :netmask => "255.255.255.224", :auto_config => false
    
     config.vm.provider :virtualbox do |vb|
           # eth2 must be in promiscuous mode for floating IPs to be accessible
           vb.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"]
     end
    end

    config.vm.provider :virtualbox do |vb|
        vb.memory = vm_memory
        vb.cpus = vm_cpus
    end
   
    config.vm.provision :shell, inline: <<-SCRIPT
#create swap
	if [ ! -f /file.swap ]; then
	fallocate -l 3G /file.swap
	chmod 600 /file.swap
	mkswap /file.swap
	swapon /file.swap
	echo '/file.swap    none    swap    sw    0   0'>>/etc/fstab
    fi
	
     if [ ! -f /vagrant/devstack.yaml ]; then
        echo 'error synchronized folder /vagrant missing files!' >&2
        exit 255
     fi
     DEBIAN_FRONTEND=noninteractive apt-get -qqy install software-properties-common
     DEBIAN_FRONTEND=noninteractive apt-add-repository ppa:ansible/ansible
     DEBIAN_FRONTEND=noninteractive apt-get -qqy update
     DEBIAN_FRONTEND=noninteractive apt-get -qqy install ansible
     ansible-playbook /vagrant/devstack.yaml -v -e 'trove_db_name=#{trove_db_name}' -e 'trove_db_version=#{trove_db_version}' \
     -e 'public_ip=#{public_ip}' -e 'vnc_keymap=#{vnc_keymap}' \
     -e 'version=#{os_version}' -e 'cirros_arch=#{cirros_arch}' #{components_params}
     [ "$?" -ne '0' ] && exit 255
     chown -R vagrant /opt/stack/.
     if [ $(sudo -u vagrant env HOME=/home/vagrant screen -ls stack | wc -l) -gt '2' ]; then
          echo 'unstack current openstack'
          sudo -u vagrant env HOME=/home/vagrant bash -c 'cd /opt/stack/devstack; ./unstack.sh'
     fi
    SCRIPT
    if os_components.include?('trove')
     if config_arch == 32
      dib_arch='i386'
     else
      dib_arch='amd64'
     end
     if vm_memory < 4096
      dib_notmpfs_opts='--no-tmpfs'
     else
      dib_notmpfs_opts=''
     end
     config.vm.provision :shell, inline: <<-SCRIPT

       export ELEMENTS_PATH=/opt/stack/tripleo-image-elements/elements:/opt/stack/trove-guest-image-elements/elements
       export DIB_CLOUD_INIT_DATASOURCES="ConfigDrive"
       export DISTRO=ubuntu
       export DATASTORE='#{trove_db_name}'
       export DATASTORE_VERSION='#{trove_db_version}'
       [ ! -d /opt/stack/images ] && mkdir -p /opt/stack/images
       QEMU_IMG_OPTIONS=$(! $(qemu-img | grep -q 'version 1') && \
            echo "--qemu-img-options compat=0.10")
	if [ ! -f /opt/stack/images/${DISTRO}-${DATASTORE}-${DATASTORE_VERSION}-guest-image.qcow2 ]; then 
       /opt/stack/diskimage-builder/bin/disk-image-create #{dib_notmpfs_opts} -a #{dib_arch} \
           -o /opt/stack/images/${DISTRO}-${DATASTORE}-${DATASTORE_VERSION}-guest-image \
           -x ${QEMU_IMG_OPTIONS} ${DISTRO} ${EXTRA_ELEMENTS} \
            vm heat-cfntools cloud-init-datasources \
            ${DISTRO} ${DISTRO}-${DATASTORE}-guest-image
	fi
     SCRIPT
    end
    config.vm.provision :shell, inline: <<-SCRIPT
     sudo -u vagrant env HOME=/home/vagrant bash -c 'cd /opt/stack/devstack; ./stack.sh'
     [ "$?" -ne '0' ] && exit $?
     ip a show eth2 && ovs-vsctl add-port br-ex eth2-he || :
     ansible-playbook /vagrant/horizon-workaround.yaml -v
     echo 'OFFLINE=True'>>/opt/stack/devstack/local.conf
    SCRIPT
    
end