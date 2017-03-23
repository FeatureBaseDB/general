# Pilosa Configuration Proposal

**Discussion**
* https://github.com/pilosa/general/issues/13

**Authors**

* Yuce Tekol <yuce@pilosa.com>
* Cody Soyland <cody@pilosa.com>
* Matt Jaffee <jaffee@pilosa.com>

**Change log**

* 2017-02-27: Updated default data directory
* 2017-02-09: Updated *Address* to include *Scheme*
* 2017-02-03: Second draft
* 2017-02-01: First draft

## Abstract

This document contains the proposals for command line, configuration file and environment variable naming and their priorities.

## Overview

Configuration is one of the most important parts of software. If a consistent naming convention is not used, it may become hard to configure software. If the software doesn't support the same configuration as in the documentation, the users of the software may get frustrated. Because of those reasons, it is beneficial to have a single reference of configuration options, which Pilosa developers use during development and which can be used to write/update the documentation.

Most common configuration comes from the command line, a configuration file and environment variables. What set of options should be supported by these configuration sources? If the same option is used in two different sources, which should have the upper hand? These questions should be clearly answered to decrease the number of surprises during operation of the software.

In summary, the aims of this document are:

* To be an up-to-date reference of configuration supported by Pilosa. Ideally this document should contain information about all public Pilosa configuration at all times.
* To lay the foundation for for common terms used during configuration and define them.
* To propose uniform names for configuration options in order to decrease developer surprises.
* To clearly show the priorities between different configuration sources in order to have less surprises during the operation of Pilosa.

In **Table 1** below, current configuration options and defaults are presented:

Description                       | Configuration File | Command Line Flag | Default
--- | --- | --- | ---
Configuration file           | *(N/A)* | -config | -
Data directory                    | data-dir | - | ~./pilosa
Host                              | host | - | 127.0.0.1:15000
Number of replicas in the cluster | [cluster]/replicas | - | -
Cluster hosts                     | [[cluster.node]]/host | - | -
Cluster polling interval          | [cluster]/polling-interval | - | 60 seconds
Cluster messenger type            | [cluster]/messenger-type | - | -
Cluster gossip seed address       | [cluster.gossip]/seed | - | -
Cluster gossip port               | [cluster.gossip]/port | - | -
Plugin path                       | [plugins.path] | - | -
Anti entropy interval             | [anti-entropy]/interval | - | 600 seconds
CPU profile                       | - | -cpuprofile | -
CPU profile duration              | - | -cputime | 30 seconds

## Proposal

In order to have uniform meaning and representation we use the following terms in our proposal:

* **Directory**: The configuration requires a directory. Aliases: dir, DIR
* **Path**: The configuration requires a filename. Alias: PATH
* **Scheme**: The protocol of the address.
* **Host**: The domain name, hostname or IP address and the port. Alias: HOST
* **Port**: Numerical port of the service: port. Alias: PORT
* **Address**: Complete address of a service. Aliases: bind, BIND

    - `SCHEME://HOST:PORT` form: Specify scheme, host and port
    - `HOST:PORT` form: Specify host and port, use the default scheme
    - `HOST` form: Specify the host and use the default port and scheme
    - `:PORT` form: Specify the port and use the default host and scheme
    - `SCHEME://HOST` form: Specify scheme, host and use the default port
    - `SCHEME://:PORT` form: Specify scheme and port and use the default host

We renamed Host which used to mean an address in the current configuration and use HTTP Address instead. In **Table 2** below, we summarized the proposed configuration, showing added or changed configuration in bold.

Description | Required | Type | Default | Notes
--- | --- | --- | --- | ---
Configuration path | N | string | - | Must exist
Data directory | Y | string | `$HOME/.pilosa` |
**HTTP address** | Y | string | 127.0.0.1:10101 |
**Other protocol address** | N | string | - | Reserved
Number of replicas in the cluster | N | int | 1 |
**Cluster node addresses** | N | list of addresses | empty list |
Cluster polling interval | N | int | 60 seconds |
Cluster messenger type | N | string | - |
**Cluster gossip seed address** | N | string | 127.0.0.1:25000 |
Plugin directory | N | string | - | Must exist |
**Enabled plugins** | N | list of strings | all plugin names if Plugins directory is specified otherwise empty list | 
**Plugin configuration** | N | sections | - | Each plugin may have a separate configuration section |
Anti entropy interval | N | int | 600 seconds |
CPU profile | N | string | - |
CPU profile duration | N | int | 30 seconds |
**Log path** | Y | string | stdout |

### Priority of Configuration Sources

The configuration maybe specified in the command line, in an environment variable, in the configuration file or the default for that configuration is used. In order to be able to specify the configuration, the priority between these sources should be defined. Below is our proposed priority of sources, where higher level sources override the lower ones:

* Command line
* Environment variable
* Configuration file
* Default

### Command Line

One of the debates in the computing world is the number of dashes before a flag. BSD style utilities does not use any dashes and allows only single letter flags. GNU convention is to use double dashes (`--`) before the standard form of a flag and a single dash (`-`) before the alternative (short) form. Some software uses minus (`-`) to denote removal of a feature and plus (`+`) to denote addition. Java, Go and Erlang uses single dash before flags with their command line tools. In this proposal we opted for the GNU style flags, based solely on the observation that the quantity of modern software using that convention vastly outweighs the single dash style, and developers using modern UNIX and UNIX-like OSs would have a certain taste for it. It should be noted that Go’s standard flag parser doesn’t differentiate between single and double dashes.

In the light of the discussion above, always use lowercase flags with double dashes (`--`) to denote the long form and a single dash (`-`) to denote the alternative form. Only the most used flags should have an alternative form. Use dash (`-`) character as the word delimiter. Command line options should be preferably as short as possible without losing their meaning. **Table 3** is below:

Description | Standard Form | Alternative Form | Notes
--- | --- | --- | ---
Configuration path | `--config` | `-c` |
Data directory | `--data-dir` | `-d` |
HTTP address | `--bind` | `-b`|
Other protocol address | `--bind-PROTOCOL` | - | Reserved
Number of replicas in the cluster | `--cluster.replicas` | - |
Cluster node addresses | `--cluster.hosts` | - | Use space as a delimiter in the form: `HOST1:PORT1  HOST2:PORT2`
Cluster polling interval | `--poll-interval` | - |
Cluster messenger type | `--messenger` | - |
Cluster gossip seed address | `--gossip` | - |
Plugin directory | `--plugin-dir` | - |
Enabled plugins | `--plugins` | - | Use space as a delimiter in the form: `PLUGIN1 PLUGIN2`
Plugin configuration | `--plugin-PLUGIN_NAME` | - | Reserved
Anti entropy interval | `--anti-entropy.interval` | - |
CPU profile | `--profile.cpu` | - |
CPU profile duration | `--profile.cpu-time` | - |
Log path | `--log` | - |

### Configuration File

The configuration file is in the [TOML format](https://github.com/toml-lang/toml). Use lowercase section and key names. Use dash (`-`) character as the word delimiter. **Table 4** is below:

Description | Section/Key | Notes
--- | --- | ---
Configuration path | *(N/A)* |
Data directory | data-dir |
HTTP address | bind |
Other protocol address | bind-PROTOCOL |
Number of replicas in the cluster | [cluster]/replicas |
Cluster node addresses | [cluster]/hosts | Array of addresses
Cluster polling interval | [cluster]/poll-interval |
Cluster messenger type | [cluster]/messenger-type |
Cluster gossip seed address | [cluster]/gossip-seed |
Plugin directory | plugin-dir |
Enabled plugins | enabled-plugins | Array of plugin names
Plugin configuration | [plugin.PLUGIN_NAME] | Section
Anti entropy interval | [anti-entropy].interval |
CPU profile | [profile]/cpu |
CPU profile duration | [profile]/cpu-time |
Log path | log-path |

### Environment Variables

Most prominent deployment and orchestration tools such as, Puppet, Chef and Ansible also Docker support environment variables to pass configuration to a program. Moreover, environment variables are the preferred way of passing configuration for some application structuring conventions, like [Twelve-Factor App ](https://12factor.net/config)

All environment variables are uppercase with underscore (`_`) used as the word delimiter. Some deployment tools (such as Puppet) seems to unable to set environment variables per process (only for the system). In order to avoid inadvertent configuration,  `PILOSA_` prefix must be used. **Table 5** is below:

Description | Variable Name | Notes
--- | --- | --- |
Configuration path | PILOSA_CONFIG_PATH |
Data directory | PILOSA_DATA_DIR |
HTTP address | PILOSA_BIND |
Other protocol address | PILOSA_BIND_protocol | Reserved
Number of replicas in the cluster | PILOSA_CLUSTER.REPLICAS |
Cluster node addresses | PILOSA_CLUSTER.HOSTS |
Cluster polling interval | PILOSA_CLUSTER.POLL_INTERVAL |
Cluster messenger type | PILOSA_CLUSTER_MESSENGER_TYPE |
Cluster gossip seed address | PILOSA_GOSSIP |
Plugin directory | PILOSA_PLUGIN_DIR |
Enabled plugins | PILOSA_PLUGINS |
Plugin configuration | PILOSA_PLUGIN_plugin_name | Reserved
Anti entropy interval | PILOSA_ANTI_ENTROPY.INTERVAL |
CPU profile | PILOSA_PROFILE.CPU |
CPU profile duration | PILOSA_PROFILE.CPU_TIME |
Log path | PILOSA_LOG_PATH |

## Implementation

The `Config` structure in `config.go` should be modified to match **Table 2**. `(m *Main) ParseFlags(args []string)` method in `cmd/pilosa/main.go` should be moved to `config.go` and become a method of `Config`. New command line flags should be added to that method. A new methods which reads configuration from environment variables should be added. Ideally, `Config` should have a method which reads from all configuration sources and applies the priorities mentioned in this document to the fields of `Config`.
