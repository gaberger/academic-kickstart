+++
title = "Parsing Network Configurations With Clojure"
date = 2018-06-30T15:40:47-04:00
draft = true
tags = ["netops", "clojure", "automation"]
categories = ["network automation"]

[header]
image = ""
caption = ""
+++

#### Networking Configuration as a Domain Specific Language


Network configurations are a language and we can use a host of tools to allow us to create a model of this configuration language by converting it into an AST (Abstract Syntax Tree) then finally into a data structure that can be consumed by a programming language like JSON.



#### What is Clojure?

[Clojure](http://www.clojure.org) "is a dynamic, general-purpose programming language, combining the approachability and interactive development of a scripting language with an efficient and robust infrastructure for multithreaded programming. Clojure is a compiled language, yet remains completely dynamic – every feature supported by Clojure is supported at runtime. Clojure provides easy access to the Java frameworks, with optional type hints and type inference, to ensure that calls to Java can avoid reflection."


Clojure is a `LISP` and therefore code is written in a Functional Programming style. Lets take a quicker look:

#### Installing Clojure

You can follow the directions here: https://clojure.org/guides/getting_started

### Gettng Started

A simple Clojure application. 




#### Why
Having done development in some form or another for almost 30yrs I have certainly had my share of languages. Most engineers first dabbling into programming have become accustomers to the Python ecosystem, or maybe they evolved from Perl, shell scripting or something like TCL.






Most networking professionals are very familiar with the so called `CLI` syntax which has become the defacto standard thanks to Cisco Systems original router operating system IOS.



The configuration language consists of a set of *stanzas* with entry points required for the following lines to be valid.

> stanza - a group of lines forming the basic recurring metrical unit in a poem; a verse.


```
router bgp 65000
 bgp always-compare-med
 bgp confederation identifier 100
 bgp confederation peers 65527 65528 65529 65530
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
```

```EBNF
(* router *)
<router> = router_bgp

(* router bgp *)
router_bgp = routerbgp bgplist? neighbors afiu
<routerbgp> = <'router bgp'> asn
asn = number

bgplist = (<'bgp'> bgp)*
bgp = medoptions | bestpath | confederation

router-id = <'router-id'> address
<medoptions> = always-compare-med | deterministic-med
always-compare-med = 'always-compare-med'
deterministic-med =  'deterministic-med'

bestpath = (<'bestpath'> bestpathoptions)*
<bestpathoptions> = compare-routerid | as-path_confed
compare-routerid = 'compare-routerid'
as-path_confed = 'as-path confed'
```




