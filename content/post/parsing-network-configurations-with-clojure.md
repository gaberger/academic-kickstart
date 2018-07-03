+++
title = "Parsing Network Configurations With Clojure"
draft = false
tags = ["netops", "clojure", "automation"]
categories = ["network automation"]

[header]
image = ""
caption = ""
+++

### Networking Configuration as a Domain Specific Language


Network configurations are a language and we can use a host of tools to allow us to create a model of this configuration language by converting it into an AST (Abstract Syntax Tree) then finally into a data structure that can be consumed by a programming language like JSON.

### Network Configuration Parser (nparser)
`nparser` is a utility that reads in network configuration files and transforms them to a structured representation that can be used for analysis, modification, reporting, etc.. 
 
 Take a look at an example [configuration](https://raw.githubusercontent.com/gaberger/nparser/master/configs/frr/adv-bgp/frr.conf), a subset appears below

```
interface GigEthernet0/0/0
 description "Faces Leaf switch 1"
 ip address 10.1.1.1/24
!
interface GigEthernet0/0/1
 description "Faces Spine switch 1"
 ip address 10.1.2.1/24
!
interface None0
 description "For blackholing traffic"
!
interface loop0
 ip address 10.10.1.1/32
```

This configuration is a standard `integrated` *free range routing* configuration file, this is possible using the command `service integrated-vtysh-config`

### CLI as a configuration language

Most networking professionals are very familiar with the so called `CLI` syntax which has become the defacto standard thanks to Cisco Systems original router operating system IOS. 

The CLI configuration language consists of a set of *stanzas* typically opened by an entity such as *interface* and then followed by a number of attributes.

> stanza - a group of lines forming the basic recurring metrical unit in a poem; a verse.

We can describe this using a language called _Extented Backus-Naur Form_ <sup id="a1">[1](#f1)</sup>


```EBNF
(* interface section *)
interfaces = interface*
interface = <'interface'> name  description?  ip_address?
description = <'description'> (<doublequote> word+ <doublequote>)
interface-name = <'interface'> word
name = word
ip_address = <'ip address'> prefix

(* Primitives *)
<prefix> = cidr
<cidr> = (address '/' number)
<address> = #"\d+\.\d+\.\d+\.\d+"
<word> = #'[a-zA-Z0-9()\\.,-^?_|]+'
<doublequote> = #"\""
```


###### nparser process

![](https://vectr.com/firstclassfunc/c62PTRNb3.svg?width=456.11&height=56.57&select=a26dGl23C5,b5XL574L1,aCZ1VAT9M,bhOm4elW,b1jXBRq7Zy,a633NyvWK,h5AeWnZIGW,beIb7EQF0&source=selection)


## Lets take a closer look at the application
### nparser

`nparser` is a native linux binary that is executed from the command line. Executing it with no options will print the help menu.

```bash
:>./target/nparser
** ERROR: **
No sub-command specified


NAME:
 nparser - A command-line configuration generator

USAGE:
 nparser [global-options] command [command options] [arguments...]

VERSION:
 0.1.3-alpha

COMMANDS:
   to-json              Generate JSON from a config
   to-config            Generate config from an input file

GLOBAL OPTIONS:
   -?, --help
```

There are _submenus_ as well to guide the user

```bash
:>./nparser to-json
** ERROR: **
Option error: Missing option: file
Missing option: grammar


NAME:
 nparser to-json - Generate JSON from a config

USAGE:
 nparser to-json [command options] [arguments...]

OPTIONS:
       --file S*     Config input file
       --grammar S*  Grammar file
   -?, --help
```

_nparser_ to-json takes two arguments:  _file_ as the configuration and _grammar_ as the EBNF content describing the configuration.

Exeuting the following:

```
./nparser to-json --file ./configs/frr/adv-bgp/frr.conf --grammar ./parsers/frr/v2/frr.ebnf
```

Outputs the configuration into JSON having gone all three transformation steps

```json
{"<device>":{"hostname":"foobar","service":"integrated-vtysh-config","<interfaces>":[{"interface":{"<name>":"GigEthernet0/0/0","description":"\"Faces Leaf switch 1\"","ip_address":"10.1.1.1/24"}},{"interface":{"<name>":"GigEthernet0/0/1","description":"\"Faces Spine switch 1\"","ip_address":"10.1.2.1/24"}},{"interface":{"<name>":"None0","description":"\"For blackholing traffic\""}},{"interface":{"<name>":"loop0","ip_address":"10.10.1.1/32"}}],"router-id":"10.10.1.1","router_bgp":[{"<asn>":65000},{"<bgplist>":[{"bgp":{"+always-compare-med":true}},{"bgp":{"confederation":{"identifier":100}}},{"bgp":{"confederation":{"peers":["65527","65528","65529","65530"]}}},{"bgp":{"+deterministic-med":true}},{"bgp":{"bestpath":{"+as-path_confed":true}}},{"bgp":{"bestpath":{"+compare-routerid":true}}}]},{"<neighbors>":[{"neighbor":{"LEAF":"peer-group"}},{"neighbor":{"RR":"peer-group"}},{"neighbor":{"TEST":"peer-group"}},{"neighbor":{"UNDEFINED":"peer-group"}},{"neighbor":{"10.1.1.2":{"remote-as":64001}}},{"neighbor":{"10.1.1.2":{"peer-group":"LEAF"}}},{"neighbor":{"10.1.2.2":{"remote-as":73003}}},{"neighbor":{"10.1.2.2":{"peer-group":"UNDEFINED"}}},{"neighbor":{"10.1.2.2":{"update-source":"10.1.2.1"}}}]},{"<afiu>":[{"address-family":"ipv4 unicast"},{"<afneighbors>":[{"neighbor":{"LEAF":"addpath-tx-all-paths"}},{"neighbor":{"LEAF":"soft-reconfiguration inbound"}},{"neighbor":{"RR":"soft-reconfiguration inbound"}}]},{"<exit-address-family>":"exit-address-family"},{"address-family":"ipv6 unicast"},{"<afneighbors>":[{"neighbor":{"LEAF":"soft-reconfiguration inbound"}}]},{"<afneighbors>":[{"neighbor":{"TEST":"soft-reconfiguration inbound"}}]},{"<exit-address-family>":"exit-address-family"}]}],"vnc":[{"<vncdefaults>":"defaults"},{"response-lifetime":3600},{"<exitvnc>":"exit-vnc"}],"line":"vty"}}
```

How about we leverage `jq` and filter on just the _interfaces_ section

```
./nparser to-json --file ./configs/frr/adv-bgp/frr.conf --grammar ./parsers/frr/v2/frr.ebnf| jq '."<device>"."<interfaces>"' 
```

```json
[
  {
    "interface": {
      "<name>": "GigEthernet0/0/0",
      "description": "\"Faces Leaf switch 1\"",
      "ip_address": "10.1.1.1/24"
    }
  },
  {
    "interface": {
      "<name>": "GigEthernet0/0/1",
      "description": "\"Faces Spine switch 1\"",
      "ip_address": "10.1.2.1/24"
    }
  },
  {
    "interface": {
      "<name>": "None0",
      "description": "\"For blackholing traffic\""
    }
  },
  {
    "interface": {
      "<name>": "loop0",
      "ip_address": "10.10.1.1/32"
    }
  }
]
```

How about just the first interface?

```
nparser to-json --file ./configs/frr/adv-bgp/frr.conf --grammar ./parsers/frr/v2/frr.ebnf| jq '."<device>"."<interfaces>"[0]'
```

```json
{
  "interface": {
    "<name>": "GigEthernet0/0/0",
    "description": "\"Faces Leaf switch 1\"",
    "ip_address": "10.1.1.1/24"
  }
}
```


Ok lets say we are adding a new interface, we can use `jq` transform for that. Note: you can do this with any language that can manipulate JSON data.

Next we are going to add a new interface called *GigEthernet0/1/0*


```
./nparser to-json --file ./configs/frr/adv-bgp/frr.conf --grammar ./parsers/frr/v2/frr.ebnf| jq '."<device>"."<interfaces>" += [{interface: {"<name>": "GigEthernet0/1/0", ip_address: "10.2.1.1/24", description: "\"Faces Leaf switch 1\""}}]'
```


```
{
  "<device>": {
    "hostname": "foobar",
    "service": "integrated-vtysh-config",
    "<interfaces>": [
      {
        "interface": {
          "<name>": "GigEthernet0/0/0",
          "description": "\"Faces Leaf switch 1\"",
          "ip_address": "10.1.1.1/24"
        }
      },
      {
        "interface": {
          "<name>": "GigEthernet0/0/1",
          "description": "\"Faces Spine switch 1\"",
          "ip_address": "10.1.2.1/24"
        }
      },
      {
        "interface": {
          "<name>": "None0",
          "description": "\"For blackholing traffic\""
        }
      },
      {
        "interface": {
          "<name>": "loop0",
          "ip_address": "10.10.1.1/32"
        }
      },
      {
        "interface": {
          "<name>": "GigEthernet0/1/0",
          "ip_address": "10.2.1.1/24",
          "description": "\"Faces Leaf switch 1\""
        }
      }
    ],
    "router-id": "10.10.1.1",
    "router_bgp": [
      {
        "<asn>": 65000
      },
      {
        "<bgplist>": [
          {
            "bgp": {
              "+always-compare-med": true
            }
          },
          {
            "bgp": {
              "confederation": {
                "identifier": 100
              }
            }
          },
          {
            "bgp": {
              "confederation": {
                "peers": [
                  "65527",
                  "65528",
                  "65529",
                  "65530"
                ]
              }
            }
          },
          {
            "bgp": {
              "+deterministic-med": true
            }
          },
          {
            "bgp": {
              "bestpath": {
                "+as-path_confed": true
              }
            }
          },
          {
            "bgp": {
              "bestpath": {
                "+compare-routerid": true
              }
            }
          }
        ]
      },
      {
        "<neighbors>": [
          {
            "neighbor": {
              "LEAF": "peer-group"
            }
          },
          {
            "neighbor": {
              "RR": "peer-group"
            }
          },
          {
            "neighbor": {
              "TEST": "peer-group"
            }
          },
          {
            "neighbor": {
              "UNDEFINED": "peer-group"
            }
          },
          {
            "neighbor": {
              "10.1.1.2": {
                "remote-as": 64001
              }
            }
          },
          {
            "neighbor": {
              "10.1.1.2": {
                "peer-group": "LEAF"
              }
            }
          },
          {
            "neighbor": {
              "10.1.2.2": {
                "remote-as": 73003
              }
            }
          },
          {
            "neighbor": {
              "10.1.2.2": {
                "peer-group": "UNDEFINED"
              }
            }
          },
          {
            "neighbor": {
              "10.1.2.2": {
                "update-source": "10.1.2.1"
              }
            }
          }
        ]
      },
      {
        "<afiu>": [
          {
            "address-family": "ipv4 unicast"
          },
          {
            "<afneighbors>": [
              {
                "neighbor": {
                  "LEAF": "addpath-tx-all-paths"
                }
              },
              {
                "neighbor": {
                  "LEAF": "soft-reconfiguration inbound"
                }
              },
              {
                "neighbor": {
                  "RR": "soft-reconfiguration inbound"
                }
              }
            ]
          },
          {
            "<exit-address-family>": "exit-address-family"
          },
          {
            "address-family": "ipv6 unicast"
          },
          {
            "<afneighbors>": [
              {
                "neighbor": {
                  "LEAF": "soft-reconfiguration inbound"
                }
              }
            ]
          },
          {
            "<afneighbors>": [
              {
                "neighbor": {
                  "TEST": "soft-reconfiguration inbound"
                }
              }
            ]
          },
          {
            "<exit-address-family>": "exit-address-family"
          }
        ]
      }
    ],
    "vnc": [
      {
        "<vncdefaults>": "defaults"
      },
      {
        "response-lifetime": 3600
      },
      {
        "<exitvnc>": "exit-vnc"
      }
    ],
    "line": "vty"
  }
}
```

Lets filter that again so we can see just the interfaces.

```
./target/nparser to-json --file ./configs/frr/adv-bgp/frr.conf --grammar ./parsers/frr/v2/frr.ebnf| jq '."<device>"."<interfaces>" += [{interface: {"<name>": "GigEthernet0/1/0", ip_address: "10.2.1.1/24", description: "\"Faces Leaf switch 1\""}}]'| jq '."<device>"."<interfaces>"'
```

```
[
  {
    "interface": {
      "<name>": "GigEthernet0/0/0",
      "description": "\"Faces Leaf switch 1\"",
      "ip_address": "10.1.1.1/24"
    }
  },
  {
    "interface": {
      "<name>": "GigEthernet0/0/1",
      "description": "\"Faces Spine switch 1\"",
      "ip_address": "10.1.2.1/24"
    }
  },
  {
    "interface": {
      "<name>": "None0",
      "description": "\"For blackholing traffic\""
    }
  },
  {
    "interface": {
      "<name>": "loop0",
      "ip_address": "10.10.1.1/32"
    }
  },
  {
    "interface": {
      "<name>": "GigEthernet0/1/0",
      "ip_address": "10.2.1.1/24",
      "description": "\"Faces Leaf switch 1\""
    }
  }
]
```

Ok great!, Now we can leverage the _generation_ part of `nparser`. The structure that is produced by `nparser` has a number of syntactic *decorators* which allow the generator to understand how to produce a valid configuration syntax. We won't go through those details here, but needless to say the generator has no other knowledge of the configuration semantics and can be used to generate any number of configuration formats.


Ok so if we want to generate the configuration we can use the *to-config* option.

```
echo $(./target/nparser to-json --file ./configs/frr/adv-bgp/frr.conf --grammar ./parsers/frr/v2/frr.ebnf | jq '."<device>"."<interfaces>" += [{interface: {"<name>": "GigEthernet0/1/0", ip_address: "10.2.1.1/24", description: "\"Faces Leaf switch 1\""}}]') | jq '."<device>"."<interfaces>"' | ./target/nparser to-config
```

Generated configuration

```
interface GigEthernet0/0/0
description "Faces Leaf switch 1"
ip address 10.1.1.1/24
interface GigEthernet0/0/1
description "Faces Spine switch 1"
ip address 10.1.2.1/24
interface None0
description "For blackholing traffic"
interface loop0
ip address 10.10.1.1/32
interface GigEthernet0/1/0
ip address 10.2.1.1/24
description "Faces Leaf switch 1"
```

That's a long command line so lets unpack it.

```
echo $(./target/nparser to-json --file ./configs/frr/adv-bgp/frr.conf --grammar ./parsers/frr/v2/frr.ebnf | jq '."<device>"."<interfaces>" += [{interface: {"<name>": "GigEthernet0/1/0", ip_address: "10.2.1.1/24", description: "\"Faces Leaf switch 1\""}}]')
```

Above is exactly what we used to modify the JSON structure except we are wrapping it so it can be piped into `nparser` using STDIN. Yes, this is a _secret_ option available to make it easier to use in CI pipelines. 

### Sidebar

Try this.
```
echo "foobar" | ./target/nparser to-config --help
```
```
NAME:
 nparser to-config - Generate config from an input file

USAGE:
 nparser to-config [command options] [arguments...]

OPTIONS:
       --stdin S*  JSON input file
   -?, --help
```

See the hidden `--stdin` option shows up when there is something in the input buffer.

### End Sidebar

so the next two commands are 

```jq '."<device>"."<interfaces>"' | ./target/nparser to-config```


This should be obvious by now, the first `jq` call filters out the list of interfaces but the second uses the new option `to-config` and takes the input buffer from the pipe to generate just the interface subsection.

Lets remove the filter to generate the entire configuration


```
echo $(./target/nparser to-json --file ./configs/frr/adv-bgp/frr.conf --grammar ./parsers/frr/v2/frr.ebnf | jq '."<device>"."<interfaces>" += [{interface: {"<name>": "GigEthernet0/1/0", ip_address: "10.2.1.1/24", description: "\"Faces Leaf switch 1\""}}]') | ./target/nparser to-config
```

```
hostname foobar
service integrated-vtysh-config
interface GigEthernet0/0/0
description "Faces Leaf switch 1"
ip address 10.1.1.1/24
interface GigEthernet0/0/1
description "Faces Spine switch 1"
ip address 10.1.2.1/24
interface None0
description "For blackholing traffic"
interface loop0
ip address 10.10.1.1/32
interface GigEthernet0/1/0
ip address 10.2.1.1/24
description "Faces Leaf switch 1"
router-id 10.10.1.1
router bgp 65000
bgp always-compare-med
bgp confederation identifier 100
bgp confederation peers 65527 65529 65530 65528
bgp deterministic-med
bgp bestpath as-path confed
bgp bestpath compare-routerid
neighbor LEAF peer-group
neighbor RR peer-group
neighbor TEST peer-group
neighbor UNDEFINED peer-group
neighbor 10.1.1.2 remote-as 64001
neighbor 10.1.1.2 peer-group LEAF
neighbor 10.1.2.2 remote-as 73003
neighbor 10.1.2.2 peer-group UNDEFINED
neighbor 10.1.2.2 update-source 10.1.2.1
address-family ipv4 unicast
neighbor LEAF addpath-tx-all-paths
neighbor LEAF soft-reconfiguration inbound
neighbor RR soft-reconfiguration inbound
exit-address-family
address-family ipv6 unicast
neighbor LEAF soft-reconfiguration inbound
neighbor TEST soft-reconfiguration inbound
exit-address-family
vnc defaults
response-lifetime 3600
exit-vnc
line vty
```

### Summary

The configuration syntax is in and of itself a very valuable language for describing the runtime configuration of a device. Operators, engineers and architects are extremely familiar with this language and often leverage it for designing and troubleshooting their environments. By formally defining a grammar for the configuration we can transform and manipulate it with a vast amount of tools and programming languages. This technique is used in various projects and processes and I hope to demonstrate how leveraging Clojure makes this easier and more consumable for network operators to leverage.


### Caveat

The grammar used in this demo is just a subset of a very extensive set of instructions used by `frr`. The techniques used by `nparser` are easily extendable to various use-cases and configuration dialects.


### Wrapup

In a follow up post, I will go into the details of `nparser` including:

1. Using dicClojure as a systems language for automation
2. Leveraging functional programming to write concise code
3. Leveraging Clojure powerful specification language to build validation
4. Leverage Clojure spec generators for doing property testing
5. Leveraging GraalVM native-image compiler for building native binaries



#### References


1. <b id="f1"></b> [grammars-bnf-ebnf](http://matt.might.net/articles/grammars-bnf-ebnf/). [↩](#a1)




