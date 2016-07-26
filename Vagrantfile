# -*- mode: ruby -*-
# vi: set ft=ruby :

# Please do not edit this file directly as it lives in the submodule. You can
# customize your Vagrant box by editing the following files :
#
#   * virtualization/parameters.yml for project related parameters
#   * virtualization/playbook.yml for provisioning option
#   * Vagrantfile should you need anything else
#
# If those options are not enough for you, please get in touch so we can
# devise something reusable for everyone !

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

# Cross-platform way of finding an executable in the $PATH.
#   which('ruby') #=> /usr/bin/ruby
def which(cmd)
  exts = ENV['PATHEXT'] ? ENV['PATHEXT'].split(';') : ['']
  ENV['PATH'].split(File::PATH_SEPARATOR).each do |path|
    exts.each { |ext|
      exe = File.join(path, "#{cmd}#{ext}")
      return exe if File.executable?(exe) && !File.directory?(exe)
    }
  end
  return nil
end

# get git config on the host so that we can have it even when running
# ansible_local as provisioner.
git_config = `git config --list --global`
git_config.gsub! "$", "\\$" # escape $ so they don't get interpreted later

custom_config = CustomConfig.new

ansible_provisioner = custom_config.get('ansible_local', true) ? 'ansible_local' : 'ansible'

if ansible_provisioner == 'ansible_local'
    Vagrant.require_version ">= 1.8.4"
else
    unless which 'ansible'
        raise "[PROVISIONER ERROR] cannot find ansible on the host. Try using ansible_local."
    end
    Vagrant.require_version ">= 1.6.2"
end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
    config.vm.box = custom_config.get('box_name', 'drifter/jessie64-base')
    box_url = custom_config.get('box_url', 'https://vagrantbox-public.liip.ch/drifter-jessie64-base.json')
    if box_url != ""
       config.vm.box_url = box_url
    end

    config.vm.hostname = custom_config.get('hostname')

    custom_config.get('forwarded_ports', {"80" => "8080", "3000" => "3000"}).each do |source, dest|
        config.vm.network "forwarded_port", guest: source, host: dest, auto_correct: true
    end

    config.ssh.forward_agent = true
    config.ssh.forward_x11 = true

    if Vagrant.has_plugin?("vagrant-hostmanager")
        config.hostmanager.enabled = true
        config.hostmanager.manage_host = true
        config.hostmanager.ignore_private_ip = false
        config.hostmanager.include_offline = true
        config.hostmanager.aliases = custom_config.get('hostnames', [])
    end

    if Vagrant.has_plugin?("vagrant-cachier")
        # cache the packages for all identical boxes
        config.cache.scope = :box
    end

    config.vm.provider "virtualbox" do |v, override|
        override.vm.network :private_network, ip: custom_config.get('box_ip', "10.10.10.10")

        if Vagrant::Util::Platform.windows? && !Vagrant.has_plugin?("vagrant-winnfsd")
            puts "*************************************************************************"
            puts "Please install the plugin vagrant-winnfsd to have NFS Support on Windows!"
            puts "vagrant plugin install vagrant-winnfsd"
            puts "*************************************************************************"
            exit 1
        end
        override.vm.synced_folder ".", "/vagrant", type: "nfs", mount_options: ['noatime', 'noacl', 'proto=udp', 'vers=3', 'async', 'actimeo=1']

        v.linked_clone = true if Gem::Version.new(Vagrant::VERSION) >= Gem::Version.new('1.8')
        v.memory = 4096
        v.cpus = 2
        v.customize ["modifyvm", :id, "--natdnsproxy1", "off"]
        v.customize ["modifyvm", :id, "--natdnshostresolver1", "off"]

        if Vagrant.has_plugin?("vagrant-cachier")
            # use the same nfs config than above for cache performance
             override.cache.synced_folder_opts = {type: "nfs", mount_options: ['noatime', 'noacl', 'proto=udp', 'vers=3', 'async', 'actimeo=1']}
        end
    end

    config.vm.provider "lxc" do |lxc, override|
        if Vagrant.has_plugin?("vagrant-hostmanager")
            override.hostmanager.ignore_private_ip = true
        end
    end

    # Set some env variables, so they can be used within the vagrant box as well
    # Important for example if you want to provision different things on the CI
    config.vm.provision "shell", inline: "echo export CI_SERVER=#{ENV['CI_SERVER']} > /etc/profile.d/drifter_vars.sh;
                                          echo export GITLAB_CI=#{ENV['GITLAB_CI']} >> /etc/profile.d/drifter_vars.sh;
                                          chmod a+x /etc/profile.d/drifter_vars.sh"

    # Sync the git configuration over
    sync_git = <<SCRIPT
echo -e "#{git_config}" > /tmp/gitconfig-host

while read l; do
    KEY=$(echo $l | cut -d"=" -f1)
    VALUE=$(echo $l | cut -d"=" -f2-)
    if [ -n "$KEY" -a -n "$VALUE" ]; then
        sudo -u vagrant git config --file /home/vagrant/.gitconfig-host --replace-all $KEY "$VALUE"
    fi
done < /tmp/gitconfig-host

rm -f /tmp/gitconfig-host
SCRIPT
    config.vm.provision "Sync GIT config", type: "shell", inline: sync_git, run: "always"

    config.vm.provision ansible_provisioner do |ansible|
        ansible.extra_vars = custom_config.get('extra_vars', {})
        ansible.playbook = custom_config.get('playbook', 'virtualization/playbook.yml')

        # force handlers to run even on playbook errors
        ansible.raw_arguments = ['--force-handlers']

        # Update verbosity as needed, multiples 'v' means more verbose
        # ansible.verbose = 'v'

        if ansible_provisioner == 'ansible'
            ansible.host_key_checking = false
            # We can't use controlmaster because it makes SSH use the same
            # connection over and over again. This clashes with the base role which
            # changes sshd configuration. If controlmaster is used, the new sshd
            # configuration is not taken into account
            ansible.raw_ssh_args = ['-o ControlMaster=no']
        end
    end

    if ENV['GITLAB_CI'] && ENV['DO_GLOBAL_PROJECTS_CACHE']
        config.vm.synced_folder "/home/gitlab-runner/projects_cache/", "/home/vagrant/.projects_cache"
    end
end
