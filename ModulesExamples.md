# MODULES USAGE EXAMPLES

Here are some usage examples of the NextGen example42 Puppet modules.

## SETTING MONITORING / FIREWALLING BEHAVIOUR
Here's an example on how you can mix general top scope variables and a module without customizing its configuration.

General Top Scope variables in site.pp (or via an ENC or on Hiera): 

        $puppi = true
        $monitor = true
        $monitor_tool = [ 'puppi' , 'nagios' ]
        $firewall = true
        $firewall_tool = [ 'iptables' ]

        class { 'openssh': }

The above code installs openssh with default options and enables monitoring via Puppi and Nagios, firewalling via the Example42 iptables module (opens port 22 to any source, note that on the same host you must have an "include iptables" to load the default iptables setup) and activates puppi integration (for puppi info and puppi log ).

You may need to customize some parameters for the openssh class, overriding the top scope general settings with:

        class { 'openssh':
          monitor_tool => [ 'puppi' , 'nagios' , 'monit' ], # For this module we want also monit checks
          firewall_src => [ '10.0.0.0/8' ], # Allowed source IPs
          firewall_dst => $ipaddress_eth1,  # Allowed destination IP is taken by eth1 
        }

The above code can be replaced by module specific (with openssh_ prefix) Top Scope or Hiera or ENC  variables:

        $openssh_monitor_tool = [ 'puppi' , 'nagios' , 'monit' ]
        $openssh_firewall_src => [ '10.0.0.0/8' ] 
        $openssh_firewall_dst => $ipaddress_eth1

        class { 'openssh': }


## CUSTOMIZING THE MODULE CONFIGURATION
If you want to customize the configuration of the files provided by the module you have various alternatives.

        class { 'openssh':
          source => [ "puppet:///modules/lab42/openssh/openssh.conf--${hostname}" , "puppet:///modules/lab42/openssh/openssh.conf-${role}" , "puppet:///modules/lab42/openssh/openssh.conf" ],
        } 

If we apply the above code on an host named web01 with a custom $role variable called "webservers", the source of the sshd.config file is searched first in /etc/puppet/modules/lab42/files/openssh.conf--web01, then, if not found, in /etc/puppet/modules/lab42/files/openssh.conf-webservers and finally if there's not a speific configuration file for the node or its role, a default file (which must exist) is retrieved: /etc/puppet/modules/lab42/files/openssh.conf .
Of course the logic of the above array (source paths, variables names and hierarchy) can be freely customized.

You can even decide to follow a similar approach of the whole configuration directory ( /etc/ssh in this case ):

        class { 'openssh':
          source_dir => [ "puppet:///modules/lab42/openssh/configdir--${hostname}/" , "puppet:///modules/lab42/openssh/configdir-${role}/" , "puppet:///modules/lab42/openssh/configdir/" ],
        }

in this case the whole content of the first  directory found among /etc/puppet/modules/lab42/files/configdir--web01/ , /etc/puppet/modules/lab42/files/configdir-webservers and /etc/puppet/modules/lab42/files/configdir/ is copied to the host's /etc/ssh directory.
Note that by default existing files not present on the puppetmaster are kept on the target directory. 
If you want to be sure that the target directory contains only files present on the puppetmaster, add the source_dir_purge option:

        class { 'openssh':
          source_dir => [ "puppet:///modules/lab42/openssh/configdir--${hostname}/" , "puppet:///modules/lab42/openssh/configdir-${role}/" , "puppet:///modules/lab42/openssh/configdir/" ],
          source_dir_purge => true ,  # Be careful with this option!
        }

Alternatively you may prefer to populate the /etc/ssh/sshd.config file with an erb template :

        class { 'openssh':
          template => 'lab42/openssh/openssh.conf.erb', 
        }

this option is alternative to the source one and looks for the erb template in /etc/puppet/modules/lab42/templates/openssh/openssh.conf.erb where you can put all the logic of the variables you want.

You can also specify arbitrary options to use in your configuration file with the option argument, which ideally expects and hash of key value pairs. An example (fits better with a class like the resolver one):

        class { 'resolver': 
          dns_servers => [ '83.138.151.80' , '83.138.151.81' , '8.8.8.8' ],
          search      => 'example42.com' , 
          options     => {
            'rotate'  => '',
            'timeout' => '2',
          },
        }

Note that in the above example the dns_servers and search arguments are specific to the reolver module and they cohexist with the "standard" options module, where an hash is used to pass specific options.
The resolver module has a default template so you need not to provide a custom one (you you can if you want), here's it's content, for reference: 

        # File Managed by Puppet
        <% dns_servers.each do |ns| %>nameserver <%= ns %>
        <% end -%>
        <% search.each do |src| -%>search <%= src %>
        <% end -%>
        <% sortlist.each do |sl| -%>sortlist <%= sl %> <% end -%>
        <% scope.lookupvar("resolver::options").sort_by {|key, value| key}.each do |key, value| -%>
        options <%= key %><% if value != "" -%>:<%= value %><% end %>
        <% end -%>

Finally you may want to add some very custom resources to the main module that do not exist on it (if they exist it's very likely that the module already exposes parameters to handle them). For example you might want to add a custom configuration to the nrpe module:

        class site::example42_nrpe {

          file { "nrpe-custom.cfg":
            path    => "${nrpe::config_dir}/nrpe-custom.cfg",
            mode    => $nrpe::config_file_mode,
            owner   => $nrpe::config_file_owner,
            group   => $nrpe::config_file_group,
            ensure  => present,
            require => File['nrpe.dir'],
            notify  => Service['nrpe'],
            content => template('site/nrpe/nrpe-custom.cfg'),
          }

        }

The above example makes use, for better integration, of variables provided by the same nrpe module, but it's not required, it could contain whatever resources. To automatically include the above custom class that somehow extends the main nrpe module you just have to write something like:

        class { 'nrpe': 
          my_class => 'site::example42_nrpe' ,
        }

And, obviously place your custom class in a file that be autoloaded, such as: /etc/puppet/modules/site/manifests/example42_nrpe.pp 

## EXTREME CUSTOMIZATIONS
Example42 expose as class parameters many variables that generally are used only internally and don't need users' modifications because they are automatically calculated for different operating systems.
There are some very specific cases where you might need to modify them, for example when you make a custom package of an application with custom paths and settings and you want to integrate it as the other modules (taking benefit of the monitoring / firewalling / puppi integrations, for example).

Here's an example where these custom settings are largely used:

        class { 'redis':
          template           => 'lab42/redis/redis.conf.erb-redis_server',
          package            => 'lab42-redis',
          service            => 'redis',
          service_status     => false,
          config_file        => '/opt/redis/conf/redis.conf',
          config_file_owner  => 'redis',
          config_file_group  => 'redis',
          config_dir         => '/opt/redis/conf/',
          pid_file           => '/opt/redis/run/redis.pid',
          log_file           => '/opt/redis/log/redis.log',
          log_dir            => '/opt/redis/log',
        }

the first argument defines what template to use for the main configuration files, all the others are releavant to a custom redis package where various paths and names are totally different from the ones provided by default by the supported distribution.
