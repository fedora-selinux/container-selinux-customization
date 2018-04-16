# Container SELinux Customization

This repository contains a proof of concept demonstrating block inheritance via CIL namespaces, which allows better container separation.

### Prerequisites

Container engine (e.g: podman) and sesearch tool for searching SELinux rules need to be installed on your system.

    # dnf install podman setools-console
    # dnf remove -y container-selinux

### Installing

    $ cd container-selinux-customization

    # make

    # semodule -i container_runtime.pp
    # semodule -i container.cil
    # semodule -i net_container.cil
    # semodule -i home_container.cil

Make sure that SELinux is in Enforcing mode

    # setenforce 1
    # getenforce
    Enforcing

## Examples

### Home directory access


The following command will run the latest system fedora container in container.process domain (alias for container_t), which is general type for containers shipped in container-selinux package.

    # podman run -v /home:/home --security-opt label=type:container.process -i -t fedora bash
    [root@2578c94cdd73 /]# cd /home
    [root@2578c94cdd73 home]# ls
    ls: cannot open directory '.': Permission denied


Note: container.process domain *CANNOT* access home_root_t dirs/objests so sesearch won't find any allow rule related to this access

    # sesearch -A -s container.process -t home_root_t

However, home_container.cil inherits rules from container namespace and adds several new rules to access /home dir

    # sesearch -A -s home_container.process -t home_root_t
    allow home_container.process home_root_t:dir { getattr ioctl map open read setattr write };

    # podman run -v /home:/home --security-opt label=type:home_container.process -i -t fedora bash
    [root@0469dc13f1a8 /]# cd /home
    [root@0469dc13f1a8 home]# ls
    lvrabec

### Network access

The next example will explain the difference between generic container.process label and net_container. net_container inherits rules from container namespace and additionally allows network access.

    # podman run --security-opt label=type:container.process -i -t fedora bash
    [root@ed342fdb6e42 /]# dnf check-update
    Error: Failed to synchronize cache for repo 'updates'

Note: container.process domain *CANNOT* access http_port_t ports so sesearch won't find any allow rule related to this access

    # sesearch -A -s container.process -t http_port_t -c tcp_socket

With net_container.process label we're able to connect to network and check for updates.

    # podman run --security-opt label=type:net_container.process -i -t fedora bash
    [root@ed342fdb6e42 /]# dnf check-update
    Fedora 27 - x86_64 - Updates                                              66 MB/s |  22 MB     00:00
    Fedora 27 - x86_64                                                        61 MB/s |  58 MB     00:00
    Last metadata expiration check: 0:00:07 ago on Sun Apr 15 19:49:27 2018.
    ...


The following SELinux query confirms that net_container.process can access http_port_t

    # sesearch -A -s net_container.process -t http_port_t -c tcp_socket
    allow sandbox_net_domain port_type:tcp_socket { name_bind name_connect recv_msg send_msg };

### Merging 

Let's say, that you would like to merge net_container and home_container namespace to allow network access and also access to homedirs.
There is a way to merge namespaces.

    $ cat my_container.cil
    (block my_container
	    (blockinherit home_container)
	    (blockinherit net_container)
    )

    # semodule -i my_container.cil

Now, namespaces are merged, following sesearch queries confirms it.

    $ sesearch -A -s my_container.process -t home_root_t 
    allow my_container.process home_root_t:dir { getattr ioctl map open read setattr write };

    $ sesearch -A -s my_container.process -t http_port_t -c tcp_socket 
    allow sandbox_net_domain port_type:tcp_socket { name_bind name_connect recv_msg send_msg };

