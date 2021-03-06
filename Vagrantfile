%w(vagrant-hostmanager vagrant-auto_network vagrant-nfs_guest).each do |plugin|
 unless Vagrant.has_plugin?(plugin)
   raise 'In order to use this box, you must install plugin: ' + plugin
 end
end

require_relative 'vagrant/inline/config'
require_relative 'vagrant/inline/nfs-plugin'

# Define Vagrantfile configuration options
VagrantApp::Config.option(:varnish, false) # If varnish needs to be enabled
  .option(:profiler, false) # Is profiler needs to be installed
  .option(:developer, false) # Is developer mode should be enabled
  .option(:magento2, false) # Is it Magento 2.0
  .option(:install, false) # Install Magento? (for now only 2.0)
  .option(:shell, false) # Shell script?
  .option(:php7, false) # Is it PHP7?
  .option(:name, '') # Name
  .option(:hostname, '') # Hostname
  .option(:domains, []) # Domain list
  .option(:cpu, 1) # Number of dedicated CPU
  .option(:memory, 1024) # Number of dedicated memory in MB
  .option(:user, 'app') # User name for share
  .option(:group, 'app') # Group name for share
  .option(:uid, Process.euid) # User ID for mapping
  .option(:gid, Process.egid) # Group ID for mapping
  .option(:directory, 'server') # Directory to be used as mount on host machine
  .option(:network, '33.33.33.0/24') # Directory to be used as mount on host machine

Vagrant.configure("2") do |config|

  # Prepare configuration and setup shell scripts for it
  current_file = Pathname.new(__FILE__)
  box_config = VagrantApp::Config.new

  # Base hypernode provisioner
  box_config.shell_add('hypernode.sh')
    .shell_add('composer.sh') # Composer installer
    .shell_add('nfs.sh') # NFS server modifications to have proper permissions
    .shell_add('developer.sh', :developer) # Developer mode setting, depends on :developer configuration flag
    .shell_add('profiler.sh', :profiler) # Profiler installer, depends on :profiler configuration flag
    .shell_add('disable-varnish.sh', :varnish, true) # Varnish disabler, depends on :varnish inverted flag
    .shell_add('magento2.sh', :magento2) # M2 Nginx Config Flag, depends on :magento2 flag
    .shell_add('magento2-install.sh', [:magento2, :install]) # M2 Installer, depends on :magento2 and :install
    .shell_add('magento2-developer.sh', [:magento2, :install, :developer]) # M2 Developer options, depends on :magento2, :install, :developer
    .shell_add('shell.sh', :shell) # Fish shell installer, depends on :shell flag
    .shell_add('hello.sh') # Final message with connection instructions

  # Loads config.rb from the same directory where Vagrantfile is in
  box_config.load(File.join(current_file.dirname, 'config.rb.dst'))
  box_config.load(File.join(current_file.dirname, 'config.rb'))

  AutoNetwork.default_pool = box_config.get(:network)

  if box_config.flag?(:php7)
    config.vm.box = 'hypernode_php7'
    config.vm.box_url = 'http://vagrant.hypernode.com/customer/php7/catalog.json'
  else
    config.vm.box = 'hypernode_php5'
    config.vm.box_url = 'http://vagrant.hypernode.com/customer/php5/catalog.json'
  end

  config.ssh.forward_agent = true

  config.vm.provider :virtualbox do |v, o|
    v.memory = box_config.get(:memory)
    v.cpus =  box_config.get(:cpu)
    v.customize [
      "modifyvm", :id, 
      "--paravirtprovider", "kvm" # for linux guest
    ]
  end

  config.vm.provider :lxc do |lxc|
    lxc.customize 'cgroup.memory.limit_in_bytes', box_config.get(:memory).to_s + 'M'
  end

  # Disable default /vagrant mount as we use custom user for box
  config.vm.synced_folder '.', '/vagrant/', disabled: true

  config.vm.synced_folder box_config.get(:directory), '/data/web', type: 'nfs_guest', create: true,
    linux__nfs_options: %w(rw no_subtree_check all_squash insecure async),
    map_uid: box_config.get(:uid).to_s,
    map_gid: box_config.get(:gid).to_s,
    owner: box_config.get(:user),
    group: box_config.get(:group)

  box_config.shell_list.each do |file|
    config.vm.provision 'shell', path: 'vagrant/provisioning/' + file, env: {
        VAGRANT_UID: box_config.get(:uid).to_s,
        VAGRANT_GID: box_config.get(:gid).to_s,
        VAGRANT_USER: box_config.get(:user),
        VAGRANT_GROUP: box_config.get(:group),
        VAGRANT_HOSTNAME: box_config.get(:hostname),
        VAGRANT_FPM_SERVICE: box_config.flag?(:php7) ? 'php7.0-fpm' : 'php5-fpm',
        VAGRNAT_PHP_ETC_DIR: box_config.flag?(:php7) ? '/etc/php/7.0/' : '/etc/php5/',
        VAGRNAT_PHP_PACKAGE_PREFIX: box_config.flag?(:php7) ? 'php7.0' : 'php5'
    }
  end

  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true

  config.vm.define 'hypernode' do |node|
    node.vm.hostname = box_config.get(:hostname)
    node.vm.network :private_network, auto_network: true
    node.hostmanager.aliases = box_config.get(:domains)
  end

end
