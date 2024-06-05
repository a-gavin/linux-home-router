# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'fileutils'

VAGRANTFILE_API_VERSION = "2"

cfg_dir = ENV["CFG_DIR"] || ""
vm_os = ENV["VM_OS"] || "f40"
base_pkg_cache_path = ENV["PKG_CACHE"] || ""

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
    is_fedora = true

    case vm_os
    when "f40"
        config.vm.box = "fedora/#{vm_os[1..-1]}-cloud-base"
    when "u22"
        # TODO: Select Ubuntu release based on version number
        is_fedora = false
        config.vm.box = "ubuntu/jammy64"
    else
        puts "ERROR: Unknown operating system \'#{vm_os}\'. Supported are \'f40\' and \'u22\'"
        puts "       For example, \'VM_OS=f40 CFG_DIR=example_cfgs vagrant up\'"
        abort
    end

    ### VM NAMING ###
    # Otherwise libvirtd-daemon name will be prefixed w/ directory name
    # See: https://github.com/vagrant-libvirt/vagrant-libvirt/issues/289#issuecomment-68229438
    config.vm.provider :libvirt do |libvirt|
        libvirt.default_prefix = ""
        #libvirt.host = "#{vm_fedora_version}" # DANGER: I have had CA cert issues after setting this
    end
    config.vm.define "#{vm_os}" # Actual VM name

    ### VM HARDWARE/DEVICES ###
    config.vm.provider :libvirt do |libvirt|
        libvirt.cpus = 2
        libvirt.memory = 2048
    end

    ### CACHE VM PACKAGES ###
    # Doing this will speed up repeated package updates/installations
    if base_pkg_cache_path != ""
        if base_pkg_cache_path[-1] !=  "/"
            base_pkg_cache_path += "/"
        end
        pkg_cache_dir = base_pkg_cache_path + "#{vm_os}"

        # Make local pkg cache dir if doesn't exist yet
        unless File.directory?(pkg_cache_dir)
            FileUtils.mkdir_p(pkg_cache_dir)
        end

        if is_fedora
            vm_cache_dir = "/var/cache/dnf"
        else
            vm_cache_dir = "/var/cache/apt"
        end

        # If using NFS mount for DNF cache, updating DNF will
        # fail because of NFS permissions issues (hand waving a bit here).
        # This is the closest I could find to the specific issue:
        # https://bugzilla.redhat.com/show_bug.cgi?id=2018678
        #config.vm.synced_folder pkg_cache_dir, vm_cache_dir, type: "nfs"
        #config.vm.synced_folder pkg_cache_dir, vm_cache_dir,
        #    type: "nfs", nfs_version: 4, nfs_udp: false
    end

    ### PROVISION VM ###
    #config.vm.provision "ansible" do |ansible|
    #    if cfg_dir.empty?
    #        puts "ERROR: Required config file directory not specified."
    #        puts "       For example, \'CFG_DIR=wired_no_vlan vagrant up\'"
    #        abort
    #    end

    #    ansible.playbook = "playbook.yml"
    #    ansible.extra_vars = {
    #        "cfg_dir": cfg_dir
    #    }
    #    ansible.verbose = "vvv"
    #end
end