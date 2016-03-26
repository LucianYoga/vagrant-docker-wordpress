# -*- mode: ruby -*-
# vi: set ft=ruby :

vagrant_dir = File.join(File.expand_path(File.dirname(__FILE__)), 'vagrant')

unless Vagrant.has_plugin?("vagrant-docker-compose")
  system("vagrant plugin install vagrant-docker-compose")
  puts "Dependencies installed, please try the command again."
  exit
end

# @param swap_size_mb [Integer] swap size in megabytes
# @param swap_file [String] full path for swap file, default is /swapfile1
# @return [String] the script text for shell inline provisioning
def create_swap(swap_size_mb, swap_file = "/swapfile1")
  <<-EOS
    if [ ! -f #{swap_file} ]; then
      echo "Creating #{swap_size_mb}mb swap file=#{swap_file}. This could take a while..."
      dd if=/dev/zero of=#{swap_file} bs=1024 count=#{swap_size_mb * 1024}
      mkswap #{swap_file}
      chmod 0600 #{swap_file}
      swapon #{swap_file}
      if ! grep -Fxq "#{swap_file} swap swap defaults 0 0" /etc/fstab
      then
        echo "#{swap_file} swap swap defaults 0 0" >> /etc/fstab
      fi
    fi
  EOS
end

Vagrant.configure("2") do |config|
  # Store the current version of Vagrant for use in conditionals when dealing
  # with possible backward compatible issues.
  vagrant_version = Vagrant::VERSION.sub(/^v/, '')

  config.vm.box = "hashicorp/boot2docker"
  config.vm.define "docker-wordpress"

  #boot2docker ssh
  config.ssh.insert_key = true
  config.ssh.username = "docker"
  config.ssh.password = "tcuser"
  config.ssh.guest_port = 22
  config.ssh.port = 2200
  config.ssh.host = '127.0.0.1'

  # Configurations from 1.0.x can be placed in Vagrant 1.1.x specs like the following.
  config.vm.provider :virtualbox do |v|
    # Set the box name in VirtualBox to match the working directory.
    v.name = "Docker-Wordpress"
  end

  # Local Machine Hosts
  #
  # If the Vagrant plugin hostsupdater (https://github.com/cogitatio/vagrant-hostsupdater) is
  # installed, the following will automatically configure your local machine's hosts file to
  # be aware of the domains specified below. Watch the provisioning script as you may need to
  # enter a password for Vagrant to access your hosts file.
  #
  # By default, we'll include the domains set up by VVV through the vvv-hosts file
  # located in the www/ directory.
  #
  # Other domains can be automatically added by including a vvv-hosts file containing
  # individual domains separated by whitespace in subdirectories of www/.
  if defined?(VagrantPlugins::HostsUpdater)
    # Recursively fetch the paths to all vvv-hosts files under the www/ directory.
    paths = Dir[File.join(vagrant_dir, 'www', '**', 'vvv-hosts')]

    # Parse the found vvv-hosts files for host names.
    hosts = paths.map do |path|
      # Read line from file and remove line breaks
      lines = File.readlines(path).map(&:chomp)
      # Filter out comments starting with "#"
      lines.grep(/\A[^#]/)
    end.flatten.uniq # Remove duplicate entries

    # Pass the found host names to the hostsupdater plugin so it can perform magic.
    config.hostsupdater.aliases = hosts
    config.hostsupdater.remove_on_suspend = true
  end

  # Private Network (default)
  #
  # A private network is created by default. This is the IP address through which your
  # host machine will communicate to the guest. In this default configuration, the virtual
  # machine will have an IP address of 192.168.50.4 and a virtual network adapter will be
  # created on your host machine with the IP of 192.168.50.1 as a gateway.
  #
  # Access to the guest machine is only available to your local host. To provide access to
  # other devices, a public network should be configured or port forwarding enabled.
  #
  # Note: If your existing network is using the 192.168.50.x subnet, this default IP address
  # should be changed. If more than one VM is running through VirtualBox, including other
  # Vagrant machines, different subnets should be used for each.
  #
  config.vm.network :private_network, id: "vvv_primary", ip: "192.168.50.10"
  config.vm.network(:forwarded_port, guest: 8080, host: 8080)
  #config.vm.network(:forwarded_port, guest: 3333, host: 3333)

  # /srv/www/
  # /vagrant/
  #
  # If a www directory exists in the same directory as your Vagrantfile, a mapped directory
  # inside the VM will be created that acts as the default location for nginx sites. Put all
  # of your project files here that you want to access through the web server
  if vagrant_version >= "1.3.0"
    config.vm.synced_folder File.join(vagrant_dir, "www/"), "/srv/www/", :owner => "www-data", :mount_options => [ "dmode=775", "fmode=664" ]
    config.vm.synced_folder File.expand_path(File.dirname(__FILE__)), "/vagrant/", :owner => "www-data", :mount_options => [ "dmode=775", "fmode=664" ]
  else
    config.vm.synced_folder File.join(vagrant_dir, "www/"), "/srv/www/", :owner => "www-data", :extra => 'dmode=775,fmode=664'
    config.vm.synced_folder File.expand_path(File.dirname(__FILE__)), "/vagrant/", :owner => "www-data", :extra => 'dmode=775,fmode=664'
  end

  config.vm.provision "fix-no-tty", type: "shell" do |s|
    s.privileged = false
    s.inline = "sudo sed -i '/tty/!s/mesg n/tty -s \\&\\& mesg n/' /root/.profile"
  end

  # The Parallels Provider does not understand "dmode"/"fmode" in the "mount_options" as
  # those are specific to Virtualbox. The folder is therefore overridden with one that
  # uses corresponding Parallels mount options.
  config.vm.provider :parallels do |v, override|
    override.vm.synced_folder File.join(vagrant_dir, "www/"), "/srv/www/", :owner => "www-data", :mount_options => []
    override.vm.synced_folder File.expand_path(File.dirname(__FILE__)), "/vagrant/", :owner => "www-data", :mount_options => []
  end

  # The Hyper-V Provider does not understand "dmode"/"fmode" in the "mount_options" as
  # those are specific to Virtualbox. Furthermore, the normal shared folders need to be
  # replaced with SMB shares. Here we switch all the shared folders to us SMB and then
  # override the www folder with options that make it Hyper-V compatible.
  config.vm.provider :hyperv do |v, override|
    override.vm.synced_folder File.join(vagrant_dir, "www/"), "/srv/www/", :owner => "www-data", :mount_options => ["dir_mode=0775","file_mode=0774","forceuid","noperm","nobrl","mfsymlinks"]
    override.vm.synced_folder File.expand_path(File.dirname(__FILE__)), "/vagrant/", :owner => "www-data", :mount_options => ["dir_mode=0775","file_mode=0664","forceuid","noperm","nobrl","mfsymlinks"]
    # Change all the folder to use SMB instead of Virtual Box shares
    override.vm.synced_folders.each do |id, options|
      if ! options[:type]
        options[:type] = "smb"
      end
    end
  end
  
  config.vm.provision :docker
  config.vm.provision :docker_compose, yml: ["/vagrant/docker-compose.yml"], rebuild: true, project_name: "myproject", run: "always"
end
