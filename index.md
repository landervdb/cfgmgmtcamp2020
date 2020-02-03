<!-- _class: title -->

<!-- footer: @landervdb -->

<div class="front">
    <div class="top">
    <div class="logo"></div>
    <div class="author">Lander Van den Bulcke<br/>@landervdb</div>
    </div>
    <div class="title">Achieving fully hands-off deployment of an Icinga 2 cluster using Puppet<br/><br/></div>
    <div class="location"><span>CfgMgmtCamp</span>February 3rd, 2020</div>
</div>

---

# Starting point

- Single Icinga 1.x server
- Puppet 3
- Exported resources all the things

---

# Starting point

- 500+ hosts
- 10 000+ services
- 20+ environments

---
<!-- _class: blue -->

## Things are sloooow

---

# Let's upgrade

- This Icinga 2 thing looks pretty cool
  - Native clustering
  - Automatic loadbalancing of checks
  - Master-satellite topology
  - Apply rules (yay!)
- Very nice upstream puppet modules provided
  - No support for Puppet 3 though
  - Aint that a good reason to upgrade...

---

# Master setup

- Lets try and get our masters set up
- Icinga wants to know about all the endpoints it needs to connect to in its configuration
- Could put it all in hiera?
- That's not really automagic though...

---

# PuppetDB to the rescue!

```puppet
class profile_icinga2 (
  Boolean          $server      = false, 
  Optional[String] $parent_zone = undef,
  String           $zone        = $::fqdn
) {
  if $server {
    if $zone == 'master' {
      include ::profile_icinga2::master
    } else {
      include ::profile_icinga2::satellite
    }
  }
  else {
    include ::profile_icinga2::agent
  }
}
```

---

# PuppetDB to the rescue!

```puppet
# Get all master endpoints
$this_endpoint = {
  $::fqdn => {
    'host' => $::ipaddress,
  }
}

$master_endpoints_results = puppetdb_query("facts[certname, value] { certname in resources[certname] 
    { type = \"Class\" and title = \"Profile_icinga2::Master\" and !(certname = \"${::fqdn}\") } 
    and name = \"ipaddress\" }")

$master_endpoints = $this_endpoint + $master_endpoints_results.reduce({}) | Hash $memo, Hash $endpoint | {
  $memo + {
    $endpoint['certname'] => {
      'host' => $endpoint['value'],
    },
  }
}
```

---

# Likewise for the satellites

```puppet
$sat_endpoints_results = puppetdb_query('facts[certname, value] { certname in resources[certname] 
    { type = "Class" and title = "Profile_icinga2::Satellite" } and name = "ipaddress" }')

$sat_endpoints = $sat_endpoints_results.reduce({}) | Hash $memo, Hash $endpoint | {
  $memo + {
    $endpoint['certname'] => {
      'host' => $endpoint['value'],
    },
  }
}

$endpoints = $master_endpoints + $sat_endpoints
```

---


# Icinga also wants to know about the zones

```puppet
$master_zone = {
  'master' => {
    'endpoints' => keys($master_endpoints),
  },
}

$zones_results = puppetdb_query('resources { type = "Class" and title = "Profile_icinga2::Satellite" }')
```

---

# Icinga also wants to know about the zones

```puppet
$zones = $master_zone + $zones_results.reduce({}) | Hash $memo, Hash $current | {
  if has_key($memo, $current['parameters']['zone']) {
    $memo + {
      $current['parameters']['zone'] => {
        'endpoints' => concat($memo[$current['parameters']['zone']]['endpoints'], $current['certname']),
        'parent'    => 'master',
      },
    }
  } else {
    $memo + {
      $current['parameters']['zone'] => {
        'endpoints' => [ $current['certname'] ],
        'parent'    => 'master',
      },
    }
  }
}
```

---

# Just need to apply it now

```puppet
class { 'icinga2::feature::api':
  accept_commands => true,
  accept_config   => $_accept_config,
  endpoints       => $endpoints,
  pki             => 'puppet',
  zones           => $zones,
}
 ```

---

# What about the satellite configuration?

```puppet
# Get other endpoints in zone
$zone_endpoints_results = puppetdb_query("resources[certname] { type = \"Class\" and 
    title = \"Profile_icinga2\" and parameters.zone = \"${zone}\" and 
    parameters.server = true and !(certname = \"${::fqdn}\") } ")

$zone_endpoints = concat($zone_endpoints_results.map | $result | { $result['certname'] }, $::fqdn)

# Get master endpoints
$master_endpoints_results = puppetdb_query('resources[certname] { type = "Class" and 
    title = "Profile_icinga2" and parameters.zone = "master" and parameters.server = true}')

$master_endpoints = $master_endpoints_results.map | $result | { $result['certname'] }
```


--- 

# What about the satellite configuration?

```puppet
$zones = {
  $zone    => {
    'endpoints' => $zone_endpoints,
    'parent'    => 'master',
  },
  'master' => {
    'endpoints' => $master_endpoints,
  }
}
```

--- 

# What about the satellite configuration?

```puppet
$zone_endpoints_hash = $zone_endpoints.reduce({}) | Hash $memo, String $endpoint | {
  $memo + {
    $endpoint => {},
  }
}

$master_endpoints_hash = $master_endpoints.reduce({}) | Hash $memo, String $endpoint | {
  $ip = puppetdb_query("facts[value] { certname = \"${endpoint}\" and name = \"ipaddress\" }")[0]['value']
  $memo + {
    $endpoint => {
      'host' => $ip,
    }
  }
}

$endpoints = $zone_endpoints_hash + $master_endpoints_hash
```

--- 

# And ofcourse the agents also need configuration

```puppet
# Get all endpoints in parent zone
$parent_endpoints_results = puppetdb_query("facts[certname, value] { certname in resources[certname] 
    { type = \"Class\" and title = \"Profile_icinga2\" and parameters.zone = \"${parent_zone}\" } 
    and name = \"ipaddress\" }")

$parent_endpoints = $parent_endpoints_results.reduce({}) | Hash $memo, Hash $endpoint | {
  $memo + {
    $endpoint['certname'] => {
      'host' => $endpoint['value'],
    },
  }
}
```

---

# And ofcourse the agents also need configuration

```puppet
$endpoints = {
  'NodeName' =>  {},
} + $parent_endpoints

$zones = {
  $::fqdn      => {
    'endpoints' => [ $::fqdn ],
    'parent'    => $parent_zone,
  },
  $parent_zone => {
    'endpoints' => keys($parent_endpoints)
  }
}
```

---

# Adding a host

```puppet
@@::icinga2::object::endpoint { $::fqdn:
  host   => $::ipaddress,
  target => "/etc/icinga2/zones.d/${parent_zone}/${::hostname}.conf",
}

@@::icinga2::object::zone { "agent ${::fqdn}":
  zone_name => $::fqdn,
  endpoints => [$::fqdn],
  parent    => $parent_zone,
  target    => "/etc/icinga2/zones.d/${parent_zone}/${::hostname}.conf",
}

@@::icinga2::object::host { $::fqdn:
  address      => $::ipaddress,
  display_name => $::fqdn,
  import       => [ 'generic-host' ],
  target       => "/etc/icinga2/zones.d/${parent_zone}/${::hostname}.conf",
}
```

---

# Adding a service

- Could apply the same strategy and export services from each node
  - Results in a whole lot of exported resources though
- Alternative: apply rules!
  - Define variables in the host definition
  - Define each service once
  - Apply the service defintion to each host with matching vars

---

# Apply rules: example

```puppet
::icinga2::object::service { 'load':
  import           => ['generic-service'],
  display_name     => 'Load',
  apply            => true,
  check_command    => 'load',
  check_period     => 'host.vars.load.check_period',
  command_endpoint => 'host.name',
  vars             => {
    load_wload1           => 'host.vars.load.load1_warning',
    load_wload5           => 'host.vars.load.load5_warning',
    load_wload15          => 'host.vars.load.load15_warning',
    load_cload1           => 'host.vars.load.load1_critical',
    load_cload5           => 'host.vars.load.load5_critical',
    load_cload15          => 'host.vars.load.load15_critical',
    notifications_enabled => 'host.vars.load.notifications_enabled',
  },
  assign           => ['host.vars.load'],
  target           => '/etc/icinga2/zones.d/global-templates/services.conf',
}
```

---

# Houston, we have a problem

- We need to specify the vars in the host definition
  - Wich means in a single hash, in a single place
  - We kinda want to define or services together with the things thay are checking though
- Solution: (beware, very dirty)
  - Write the vars to a multi-document yaml file using puppetlabs/concat
  - Use a custom fact to read in the yaml file and merge the hashes

---

# Dirty hacking...

```puppet
define profile_icinga2::varsfragment (
  String $custom_name = $title,
  Hash   $vars        = undef,
) {
  ensure_resource('concat::fragment', "icinga2_vars_${custom_name}", {
    target  => '/etc/icinga2_vars.yaml',
    content => to_yaml($vars)
  })
}
```

---

# Dirty hacking...

```ruby
require 'yaml'

Facter.add(:icinga2_vars) do
  setcode do
    if File.exist? '/etc/icinga2_vars.yaml'
      vars = {}
      YAML.load_stream(File.open('/etc/icinga2_vars.yaml')) do |document|
        vars = deep_merge(vars, document)
      end
      vars
    end
  end
end
```

---

# Dirty hacking

```puppet
@@::icinga2::object::host { $::fqdn:
  address      => $::ipaddress,
  display_name => $::fqdn,
  vars         => $::icinga2_vars
  import       => [ 'generic-host' ],
  target       => "/etc/icinga2/zones.d/${parent_zone}/${::hostname}.conf",
}
```

---

# Pro/con

- Pro:
  - No need to export _any_ services!
- Con:
  - Current approach to merging the var hashes is pretty messy
  - Need 2 runs for the fact to update

---

# Mitigating the cons

- Working on a small puppet module: hashcat
- Basically concat but for hashes:
  - No need to write files to disk
  - No need to wait for a fact in the next run; just use custom function
- Not finished, still buggy

---
<!-- _class: blue -->

## Results?

---

# Adding a new master node

Hiera:
```yaml
profile_icinga2::server: true
profile_icinga2::zone: 'master'
```

---

# Adding a new satellite node

Hiera:
```yaml
profile_icinga2::server: true
profile_icinga2::zone: 'whatever'
profile_icinga2::parent_zone: 'master'
```

---

# Adding a new agent node

Hiera:
```yaml
profile_icinga2::parent_zone: 'whatever'
```

---

# Results

- Specify the parent_zone in a common hiera file
  - No config needed for an agent, it will just join the cluster
- Provisioning a new master/satellite
  - Will automatically join the cluster
  - Will automatically reconfigure other servers in current and parent zone
  - Automatic loadbalancing and failover with servers in the same zone

---
<!-- _class: contact -->

<img src="https://roidelapluie.github.io/marpit-inuits/inuits/logo.png"/>
<div class="left-right">
<div class="right">
    Lander Van den Bulcke<br/>
    @landervdb<br/>
    landervdb@inuits.eu
</div>
<div class="left">
    Essensteenweg 31<br/>
    2930 Brasschaat<br/>
    Belgium<br/>
    <br/>
    <br/>
    <strong>Contact:</strong><br/>
    info@inuits.eu<br/>
    +32-3-8082105
</div>
</div>

<!-- *footer: -->
