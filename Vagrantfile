# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'
require 'mkmf'
require 'fileutils'
require 'socket'

# Prevent logs from mkmf
module MakeMakefile::Logging
  @logfile = File::NULL
end

DIR = File.dirname(__FILE__)

# Create .vagrant folder
unless File.exist?(File.join(DIR,'.vagrant'))
  FileUtils.mkdir_p( File.join(DIR,'.vagrant') )
end

# Create config file
config_file = File.join(DIR, 'config.yml')
sample_file = File.join(DIR, 'config-sample.yml')

unless File.exists?(config_file)
  # Use sample instead
  FileUtils.copy sample_file, config_file
  puts '==> default: config.yml was not found. Copying defaults from sample configs..'
end

site_config = YAML.load_file(config_file)

# Create private ip address in file
private_ip_file = File.join(DIR,'.vagrant','private.ip')

unless File.exists?(private_ip_file)
  private_ip = "192.168.#{rand(255)}.#{rand(2..255)}"
  File.write(private_ip_file, private_ip)
else
  private_ip = File.open(private_ip_file, 'rb') { |file| file.read }
end

# Multiple public_network mappings need at least 1.7.4
# Custom triggers stop working in 2.1.0 so that is max for now
Vagrant.require_version '>= 1.7.4', '< 2.1.0'

Vagrant.configure('2') do |config|

  # Use host-machine ssh-key so we can log into production
  config.ssh.forward_agent = true

  # Minimum box version requirement for this Vagrantfile revision
  config.vm.box_version = "<= 1.0.0"

  # Use precompiled box
  config.vm.box = 'seravo/wordpress-beta'

  # Use the name of the box as the hostname
  config.vm.hostname = site_config['name']

  # Only use avahi if config has this
  # development:
  #   avahi: true
  if site_config['development']['avahi'] && has_internet? and is_osx?
    # The box uses avahi-daemon to make itself available to local network
    config.vm.network "public_network", bridge: [
      "en0: Wi-Fi (AirPort)",
      "en1: Wi-Fi (AirPort)",
      "wlan0"
    ]
  end

  # Use random ip address for box
  # This is needed for updating the /etc/hosts file
  config.vm.network :private_network, ip: private_ip

  config.vm.define "#{site_config['name']}-box"

  # Disable default vagrant share
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # Sync the folders
  # We have tried using NFS but it's super slow compared to synced_folder
  config.vm.synced_folder DIR, '/data/wordpress/', owner: 'vagrant', group: 'vagrant', mount_options: ['dmode=775', 'fmode=775']

  # Add SSH Public Key from developer home folder into vagrant
  if File.exists? File.join(Dir.home, ".ssh", "id_rsa.pub")
    id_rsa_ssh_key_pub = File.read(File.join(Dir.home, ".ssh", "id_rsa.pub"))
    config.vm.provision :shell, :inline => "echo '#{id_rsa_ssh_key_pub }' >> /home/vagrant/.ssh/authorized_keys && chmod 600 /home/vagrant/.ssh/authorized_keys"
  end

  # Some useful triggers for better workflow
  if Vagrant.has_plugin? 'vagrant-triggers'
    config.trigger.after :up do

      #Run all system commands inside project root
      Dir.chdir(DIR)

      # vagrant shell provisioner does nto support interactive mode..
      # run_remote('wp-development-up')

      # Use system call as that is the only way to get an interactive session
      # where the developer can respond to the wp-development-up prompts
      # @TODO: try running this always, not just on trigger?
      system("vagrant ssh --tty --command 'wp-development-up'")
      #system("vagrant ssh --tty --command 'sc shell wp-development-up'") # circumvent wp-wrapper

      # Sync plugin files from production is so configured to do
      if site_config['production'] != nil && site_config['production']['ssh_port'] != nil and site_config['development']['pull_production_plugins'] == 'always'
        run_remote("wp-pull-production-plugins")
      end

      # Sync theme files from production is so configured to do
      if site_config['production'] != nil && site_config['production']['ssh_port'] != nil and site_config['development']['pull_production_themes'] == 'always'
        run_remote("wp-pull-production-themes")
      end

      # Database imports
      if site_config['production'] != nil && site_config['production']['ssh_port'] != nil && site_config['development']['pull_production_db'] != 'never' && (site_config['development']['pull_production_db'] == 'always' or confirm("Pull database from production?", false))
        # Seravo customers are asked if they want to pull the production database here
        run_remote("wp-pull-production-db")
      elsif File.exists?(File.join(DIR,'.vagrant','shutdown-dump.sql'))
        # Return the state where we last left if WordPress isn't currently installed
        # First part in the command prevents overriding existing database
        run_remote("wp core is-installed --quiet &>/dev/null || wp-vagrant-import-db")
      elsif File.exists?(File.join(DIR,'vagrant-base.sql'))
        run_remote("wp db import /data/wordpress/vagrant-base.sql")
      else
        # If nothing else was specified bootstap just installed basic WordPress
        # @TODO run this only if installation ran, not on every Vagrant up
        notice "Installed default WordPress with user:vagrant password:vagrant"
      end

      # Don't activate git hooks, just notify them
      if File.exists?( File.join(DIR,'.git', 'hooks', 'pre-commit') )
        puts "If you want to use a git pre-commit hook please run 'wp-activate-git-hooks' inside the Vagrant box."
      end

      case RbConfig::CONFIG['host_os']
      when /darwin/
        # Do OS X specific things

        # Trust the self-signed cert in keychain
        unless File.exists?(File.join(ssl_cert_path,'trust.lock'))
          if File.exists?(File.join(ssl_cert_path,'development.crt')) and confirm "Trust the generated ssl-certificate in OS-X keychain?"
            system "sudo security add-trusted-cert -d -r trustRoot -k '/Library/Keychains/System.keychain' '#{ssl_cert_path}/development.crt'"
            # Write lock file so we can remove it too
            touch_file File.join(ssl_cert_path,'trust.lock')
          end
        end
      when /linux/
        # Do linux specific things
      end

      # Run 'vagrant up' customizer script if it exists
      if File.exist?(File.join(DIR, 'vagrant-up-customizer.sh'))
        notice 'Found vagrant-up-customizer.sh and running it ...'
        Dir.chdir(DIR)
        system 'sh ./vagrant-up-customizer.sh'
      end

      puts "\n"
      notice "Documentation available at https://seravo.com/docs/"
      notice "Visit your site: https://#{site_config['name']}.local"
    end

  else
    puts 'vagrant-triggers plugin missing, using native triggers..'
    system("vagrant ssh --tty --command 'wp-development-up'")
    exit 1
  end

  config.vm.provider 'virtualbox' do |vb|
    # Give VM access to all cpu cores on the host
    cpus = case RbConfig::CONFIG['host_os']
      when /darwin/ then `sysctl -n hw.physicalcpu`.to_i
      when /linux/ then `nproc`.to_i
      else 2
    end

    # Customize memory in MB
    vb.customize ['modifyvm', :id, '--memory', 1536]
    vb.customize ['modifyvm', :id, '--cpus', cpus]

    # Fix for slow external network connections
    vb.customize ['modifyvm', :id, '--natdnshostresolver1', 'on']
    vb.customize ['modifyvm', :id, '--natdnsproxy1', 'on']
  end

end

##
# Custom helpers
##
def notice(text)
  puts "==> trigger: #{text}"
end

##
# Create empty file
##
def touch_file(path)
  File.open(path, "w") {}
end

##
# Get boolean answer for question string
##
def confirm(question,default=true)
  if default
    default = "yes"
  else
    default = "no"
  end

  confirm = nil
  until ["Y","N","YES","NO",""].include?(confirm)
    confirm = ask "#{question} (#{default}): "

    if (confirm.nil? or confirm.empty?)
      confirm = default
    end

    confirm.strip!
    confirm.upcase!
  end
  if confirm.empty? or confirm == "Y" or confirm == "YES"
    return true
  end
  return false
end

##
# This is quite hacky but for my understanding the only way to check if the current box state
##
def vagrant_running?
  system("vagrant status --machine-readable | grep state,running --quiet")
end

##
# On OS X we can use a few more features like zeroconf (discovery of .local addresses in local network)
##
def is_osx?
  RbConfig::CONFIG['host_os'].include? 'darwin'
end

##
# Modified from: https://coderrr.wordpress.com/2008/05/28/get-your-local-ip-address/
# Returns true/false
##
def has_internet?
  orig, Socket.do_not_reverse_lookup = Socket.do_not_reverse_lookup, true  # turn off reverse DNS resolution temporarily
  begin
    UDPSocket.open { |s| s.connect '8.8.8.8', 1 }
    return true
  rescue Errno::ENETUNREACH
    return false # Network is not reachable
  end
ensure
  Socket.do_not_reverse_lookup = orig
end
