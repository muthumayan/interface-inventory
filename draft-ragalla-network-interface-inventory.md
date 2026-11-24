---
title: "Network-wide Interface Inventory"
abbrev: "Network Interface Inventory"
category: std

docname: draft-ragalla-device-interface-inventory-latest
submissiontype: IETF
number:
date: 2026-10-29
consensus: true
v: 3
area: OPS
workgroup: IVY Working Group
keyword:
 - interface inventory
 - YANG
 - schema mount
 - network topology
venue:
  group: IVY
  type: Working Group
  mail: ivy@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/ivy/
  github: TODO/REPO
  latest: https://datatracker.ietf.org/doc/draft-ragalla-device-interface-inventory/

author:
 -
    fullname: TBD
    organization: TBD
    email: TBD

normative:

informative:

...

--- abstract

This document defines a YANG data model for a network-wide inventory of
physical and logical interfaces, organized by network element. The model
provides interface attributes, operational state, and layering
relationships, with correlations to network inventory and topology data.

--- middle

# 1.  Introduction

Network operators and management systems need a consistent inventory
of physical and logical interfaces across network elements. Such an
inventory supports per-device retrieval of interface attributes and
state, as well as correlation with network inventory and topology.

An SDN controller can expose this inventory to northbound clients, such
as higher-level controllers and automation systems, to support
network-wide operations.

RFC 8343 [RFC8343] defines interface data, including operational state
and interface-layering relationships, for a server. The IVY network
inventory model [IVY-INVENTORY] describes network elements and their
components, while RFC 8345 [RFC8345] and the IVY inventory topology
model [IVY-TOPOLOGY] describe topology and its relationship to inventory.
These models provide complementary information.

This document defines a network-wide interface inventory organized by
network element. It reuses RFC 8343 interface data and provides
correlations to network inventory and topology. Sections 5 and 6
describe the data model; Sections 7 and 8 provide operations and
examples. Discovery and reconciliation of the underlying data are
implementation-specific.

# 2.  Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 [RFC2119] [RFC8174] when, and only when, they
appear in all capitals, as shown here.

Tree diagrams used in this document follow the notation defined in
[RFC8340].

The terms "network element" (NE), "component", and "port" are
used as defined in [IVY-INVENTORY].  The terms "network", "node",
"termination point" (TP), and "supporting-node"/"supporting-
termination-point" are used as defined in [RFC8345].

# 3.  Problem Statement

A network management system may collect interface data from multiple
network elements and expose it to clients. RFC 8343 provides an
interface model for a server, including interface state and layering.
The IVY inventory model [IVY-INVENTORY] identifies network elements
and components, while RFC 8345 [RFC8345] and the IVY inventory topology
model [IVY-TOPOLOGY] describe topology and its relationship to inventory.

These models do not themselves define the combined inventory view
proposed here: interface data organized by IVY network-element identity,
with correlations from each interface to its associated inventory
component and topology termination points. A client using the separate
data sets may need to obtain and correlate information from device,
inventory, and topology views, including multiple topology layers.

For example, a controller can learn from the RFC 8345 topology model
which link connects two network devices. If that link is realized as a
link aggregation group (LAG) and the link degrades (for example, due to
loss/errors on one member), troubleshooting typically requires identifying
the member interfaces of the LAG on each device. A controller may already
collect the LAG and member-interface relationships while building its
topology view, but RFC 8345 does not provide a device-scoped interface
inventory with operational state and layering that can be retrieved and
correlated directly to the topology termination points. This document's
model allows the controller to expose that per-device interface inventory
and its correlations northbound, so clients can navigate from a topology
link to the underlying per-device member interfaces and their state.

The proposed model provides a common structure for that view. Its
completeness and accuracy depend on the data available to the server
and on the server's discovery and reconciliation mechanisms.

# 4.  Relationship to Existing Work

RFC 8343 [RFC8343] Section 3.3 ("Interface Layering") already
defines the generic mechanism this document needs: two read-only
leaf-lists, "higher-layer-if" and "lower-layer-if", present on
every "/interfaces/interface" entry, explicitly described as the
YANG equivalent of the IF-MIB "ifStackTable" (RFC 8343 Section 4).
A LAG's members populate the LAG's "lower-layer-if"; each member's
"higher-layer-if" points back at the LAG.  A VLAN sub-interface's
"lower-layer-if" points at its parent physical interface.  This
document does not redefine LAG or VLAN semantics; it relies
entirely on RFC 8343 Section 3.3 for interface stacking.

RFC 8528 [RFC8528] ("YANG Schema Mount") defines a mechanism for
attaching a schema tree at a mount point. Its Appendix A illustrates
device/LNE-scoped mounted data, but the RFC does not prescribe a
device key or require one mount per device. This proposal chooses
that organization as a data-model design. It uses separate parent
and mounted modules. RFC 8528 confines paths in a mounted schema to
the mount jail; therefore the mounted augmentation represents
cross-model inventory/topology identifiers as strings rather than
leafrefs into parent-schema data. A server may use shared-schema
parent-reference metadata for XPath evaluation, but this proposal
does not require it. This document does not change RFC 8345,
RFC 8346, RFC 8944, or the IVY drafts.

# 5.  Data Model Overview

## 5.1.  Design Summary

- "network-interface-inventory/device" is keyed by "ne-ref", a leafref into
  the IVY "network-element" list -- the same device identifier
  already used by inventory, so no new device identity scheme is
  introduced.

- Each "device" list entry is an RFC 8528 mount point labeled
  "interfaces-root", under which the server exposes a mounted
  "ietf-interfaces:interfaces" subtree. The server advertises the
  mounted module set in schema-mount/YANG Library data.

- Every "if:interface" entry in that mounted subtree is augmented
  (Section 6) with:

  - an "inventory-ref" carrying the IVY port component identifier,
    when one is known, within the network element identified by the
    enclosing device's "ne-ref"; and

  - zero or more "topology-ref" entries carrying string
    identifiers for the RFC 8345 network/node/termination-point
    views corresponding to the interface; and

  - "vlan-id" and "ip-address" convenience leaves populated
    from the corresponding RFC 8944/RFC 8346-style topology
    attributes, so a client does not have to dereference
    "topology-ref" just to read a VLAN tag or an IP address.

- "higher-layer-if" and "lower-layer-if" are NOT redefined; they
  are inherited unmodified from RFC 8343. The server populates
  this operational view from device/interface data and available
  LAG/VLAN relationships; the discovery and reconciliation
  mechanism is implementation-specific.

The following illustration shows the hierarchy and example correlations.
Device identifiers are abbreviated, and RouterX represents another
network element. Arrows indicate relationships, not data containment.

```text
network-interface-inventory
|
+-- device [ne-ref = RouterA] [mount point: interfaces-root]
|   +-- interfaces ...
|
+-- device [ne-ref = RouterB] [mount point: interfaces-root]
|   +-- interfaces
|       +-- interface [Bundle-Ether10]
|       |   +-- lower-layer-if --> TenGigE0/1/0/0
|       |   +-- topology-ref -----> network / node / TP
|       |
|       +-- interface [TenGigE0/1/0/0]
|           +-- higher-layer-if -> Bundle-Ether10
|           +-- inventory-ref ---> port component in RouterB
|           +-- topology-ref -----> network / node / TP
|
|   ...
|
+-- device [ne-ref = RouterX] [mount point: interfaces-root]
    +-- interfaces ...
```

## 5.2.  Tree Diagram

Parent schema module: inventory-network-interfaces

```text
  +--ro network-interface-inventory
     +--ro device* [ne-ref]
        +--ro ne-ref             -> /nei:network-inventory/.../ne-id
        +--mp interfaces-root
           (mounted schema set includes ietf-interfaces and
            device-interface-attributes)
```

Mounted schema augmentation module:
device-interface-attributes

```text
  augment /if:interfaces/if:interface:
    +--ro inventory-ref?
    |  +--ro port-ref?      string
    +--ro topology-ref* [network-ref node-ref tp-ref]
    |  +--ro network-ref    string
    |  +--ro node-ref       string
    |  +--ro tp-ref         string
    +--ro vlan-id?                  uint16
    +--ro ip-address*               inet:ip-address
    +--ro mtu?                      uint16
    +--ro duplex-mode?              enumeration
    +--ro direction?                enumeration
    +--ro layer-rate?               string
    +--ro service-state?            enumeration
    +--ro floating?                 boolean
    +--ro notes?                    string
```

## 5.3.  Augmentation Overview

The mounted interface augmentation provides three categories of
information:

- Correlation identifiers to connect per-device interface entries to
  inventory and topology views.

- Convenience leaves derived from topology data, so clients do not need
  to dereference topology termination points to retrieve common values.

- Additional operational attributes and annotations not defined by the
  base interface model.

Normative definitions for the augmentation are in the YANG modules in
Section 6. Examples are in Section 8. Informative comparisons and
mappings are in Appendices E, F, and G.

```text
   +================+==============================+
   | Node(s)        | Purpose                      |
   +================+==============================+
   | inventory-ref  | Correlate to inventory port  |
   | topology-ref   | Correlate to topology TP(s)  |
   | vlan-id        | Convenience VLAN tag         |
   | ip-address     | Convenience IP address       |
   | mtu            | Augmented attribute          |
   | duplex-mode    | Augmented attribute          |
   | direction      | Augmented attribute          |
   | layer-rate     | Augmented attribute          |
   | service-state  | Augmented attribute          |
   | floating       | Derived/no-topology flag     |
   | notes          | Free-text annotation         |
   +================+==============================+
```

# 6.  YANG Modules

The model is split to respect RFC 8528 mount-jail semantics. The
parent module is loaded in the management system's parent schema.
At each mount point, the server advertises a mounted schema set that
includes ietf-interfaces and device-interface-attributes.
The same mounted module set MAY be shared by all devices when their
schemas are identical. The server MUST publish the applicable
mount-point schema through RFC 8528 schema-mount data and the YANG
Library mechanism (RFC 8525).

The mounted augmentation contains no leafref into the parent
inventory or topology trees. Its cross-model correlation values are
strings: RFC 8528 confines mounted paths to the mount jail unless a
server configures shared-schema parent-reference metadata. This
proposal does not require parent-reference metadata.

## 6.1.  Parent module: inventory-network-interfaces

&lt;CODE BEGINS&gt; file "inventory-network-interfaces@2026-10-29.yang"

```yang
module inventory-network-interfaces {
  yang-version 1.1;
  namespace "urn:example:ns:yang:inventory-network-interfaces";
  prefix "inii";

  import ietf-yang-schema-mount { prefix yangmnt; }
  import ietf-network-inventory { prefix nei; }

  organization "TBD";
  contact "TBD";

  description
    "Parent data model for a device-scoped interface inventory facade.
     It identifies a network element and defines an RFC 8528 schema
     mount point where the mounted schema set contains ietf-interfaces
     and device-interface-attributes.

     The module does not define an HTTP API. A server may expose the
     device list and mounted interface data through NETCONF, RESTCONF,
     or another management interface.";

  revision 2026-10-29 {
    description
      "-03 revision. Parent mount wrapper is unchanged; the mounted
       augmentation no longer includes alarm attributes.";
    reference "draft-ragalla-device-interface-inventory-03";
  }

  revision 2026-10-13 {
    description
      "Initial -02 parent/child split. The parent device mount wrapper
       is separate from the mounted interface augmentation.";
    reference "draft-ragalla-device-interface-inventory-02";
  }

  container network-interface-inventory {
    config false;
    description
      "Network-wide interface inventory organized by network element.";

    list device {
      key "ne-ref";
      description
        "One entry for each network element served by this management
         system. The ne-ref leafref is evaluated in the parent schema,
         outside the mounted schema jail. The server advertises the
         mounted schema through RFC 8528 schema-mounts data, using an
         inline or shared-schema entry as appropriate. For this
         read-only inventory view, the schema-mount config value MUST
         be false. The mounted module set MUST include ietf-interfaces
         and device-interface-attributes.";

      leaf ne-ref {
        type leafref {
          path "/nei:network-inventory/nei:network-elements/"
             + "nei:network-element/nei:ne-id";
        }
        description
          "Identifier of the network element owning the mounted
           interface list.";
      }

      yangmnt:mount-point "interfaces-root" {
        description
          "RFC 8528 mount point for the per-device interface model.";
      }
    }
  }
}
```

&lt;CODE ENDS&gt;

## 6.2.  Mounted augmentation: device-interface-attributes

&lt;CODE BEGINS&gt; file "device-interface-attributes@2026-10-29.yang"

```yang
module device-interface-attributes {
  yang-version 1.1;
  namespace "urn:example:ns:yang:device-interface-attributes";
  prefix "edia";

  import ietf-interfaces  { prefix if; }
  import ietf-inet-types  { prefix inet; }

  organization "TBD";
  contact "TBD";

  description
    "Read-only augmentation of a mounted ietf-interfaces schema for
     device-scoped interface inventory. This module is intended to be
     included in the mounted schema set at the interfaces-root mount point
     defined by inventory-network-interfaces.

     RFC 8528 confines a mounted schema to its mount jail. Therefore,
     inventory-ref and topology-ref values are correlation identifiers
     represented as strings; the mounted module does not use leafrefs
     that cross into the parent inventory or topology schema.";

  revision 2026-10-29 {
    description
      "-03 revision. Removes current-alarm-severity and alarm-ref from
       the mounted interface augmentation. All remaining augmentation
       data is operational/read-only; cross-mount correlations use
       string identifiers to respect the RFC 8528 mount-jail boundary.";
    reference "draft-ragalla-device-interface-inventory-03";
  }

  revision 2026-10-13 {
    description
      "Initial -02 mounted-interface augmentation. All augmentation
       data is operational/read-only. Cross-mount correlations use
       string identifiers to respect the RFC 8528 mount-jail boundary.";
    reference "draft-ragalla-device-interface-inventory-02";
  }

  grouping topology-ref {
    description
      "Reference to a termination point in an RFC 8345 topology view.
       The identifiers are strings because the referenced topology
       data may be outside the mounted schema.";

    leaf network-ref {
      type string;
      mandatory true;
      description "RFC 8345 network-id.";
    }
    leaf node-ref {
      type string;
      mandatory true;
      description "node-id within network-ref.";
    }
    leaf tp-ref {
      type string;
      mandatory true;
      description "tp-id within node-ref.";
    }
  }

  grouping inventory-ref {
    description
      "Identifies the inventory port component associated with this
       interface, within the network element identified by the
       enclosing device's ne-ref. The identifier is a string because
       IVY inventory data may be outside the mounted schema.";

    leaf port-ref {
      type string;
      description
        "IVY component-id of the physical port, when known, scoped
         to the network element identified by the enclosing device's
         ne-ref. This leaf identifies a directly associated physical
         port. A link aggregation interface has no port-ref; its
         physical ports can be identified by following its
         lower-layer-if entries to member interfaces and reading
         their port-ref values, when the server reports those
         relationships and correlations. This string does not enforce
         the existence of the referenced inventory component.";
    }
  }

  augment "/if:interfaces/if:interface" {
    description
      "Adds read-only inventory/topology correlations and commonly
       needed interface attributes to the mounted RFC 8343 interface
       list. The base ietf-interfaces module remains unchanged.";

    container inventory-ref {
      config false;
      description
        "Correlation to a physical port component, if one is known.";
      uses inventory-ref;
    }

    list topology-ref {
      config false;
      key "network-ref node-ref tp-ref";
      description
        "Zero or more topology termination points corresponding to
         this interface. An interface represented in multiple topology
         layers has one entry per layer/view.";
      uses topology-ref;
    }

    leaf vlan-id {
      config false;
      type uint16 { range "1..4094"; }
      description "Outer VLAN identifier when applicable.";
      reference
        "RFC 8944; openconfig-vlan single-tagged vlan-id";
    }

    leaf-list ip-address {
      config false;
      type inet:ip-address;
      description
        "IP addresses associated with this interface in the relevant
         L3 interface/topology model.";
      reference
        "RFC 8346; RFC 4293 IP-MIB; openconfig-if-ip";
    }

    leaf mtu {
      config false;
      type uint16;
      description "Maximum transmission unit, in octets.";
      reference
        "RFC 2863 IF-MIB ifMtu; RFC 8343, Sec. 4 (ifMtu not mapped to ietf-interfaces); openconfig-interfaces mtu";
    }

    leaf duplex-mode {
      config false;
      type enumeration {
        enum full-duplex;
        enum half-duplex;
        enum auto;
      }
      description
        "Observed/configured duplex state. The auto value is a
         convenience rollup; OpenConfig models auto-negotiation as a
         separate boolean.";
      reference
        "RFC 3635 EtherLike-MIB; openconfig-if-ethernet";
    }

    leaf direction {
      config false;
      type enumeration {
        enum in;
        enum out;
        enum bidir;
      }
      description "Signal direction where applicable.";
    }

    leaf layer-rate {
      config false;
      type string;
      description
        "Transport layer rate, e.g., ETH, OTU4, ODU4, or IP.";
      reference "openconfig-terminal-device rate-class and trib-protocol";
    }

    leaf service-state {
      config false;
      type enumeration {
        enum planned;
        enum in-test;
        enum in-service;
        enum maintenance;
        enum out-of-service;
      }
      description
        "Service/lifecycle state, distinct from RFC 8343 admin-status
         and oper-status.";
    }

    leaf floating {
      config false;
      type boolean;
      description
        "True when topology-ref is empty. inventory-ref may still be
         present for a known physical component not participating in a
         topology view.";
    }

    leaf notes {
      config false;
      type string;
      description "Optional human-readable annotation.";
    }
  }
}
```

&lt;CODE ENDS&gt;

# 7.  RESTCONF Operations

These examples use the RESTCONF URI form defined by RFC 8040 for a
resource beneath a schema-mount point. They assume the parent and
mounted schemas described in Section 6 are advertised by the server
through schema-mount/YANG Library data. Implementations that do not
expose RESTCONF may provide equivalent data through another API.

## 7.1.  Retrieve All Interfaces on a Device

A client retrieves every interface on a device with a single GET
against the mounted interfaces container. URI path key values MUST
be percent-encoded when they contain reserved characters:

```http
   GET /restconf/data/inventory-network-interfaces:\
       network-interface-inventory/device=RouterB.subdomain1.region3.isp.example.net/\
       ietf-interfaces:interfaces
```

The response representation is the mounted
"ietf-interfaces:interfaces" subtree plus augmentation data.
The requested device key supplies the network-element context for
any "inventory-ref/port-ref" in that subtree.
Section 8.1 shows a shortened full parent data-tree fixture for
context; it is not the literal body of this subtree GET.

## 7.2.  Retrieve a Single Interface

```http
   GET /restconf/data/inventory-network-interfaces:\
       network-interface-inventory/device=RouterB.subdomain1.region3.isp.example.net/\
       ietf-interfaces:interfaces/\
       interface=Bundle-Ether10
```

## 7.3.  Filtering by Interface Type

RESTCONF does not define generic server-side predicate filtering
on list entries. The base operation in this document returns the
interface list; RFC 8040 "fields" and "depth" may be used for
payload selection. A future revision MAY specify a YANG action or
another explicit filter contract for high-interface-count devices;
no such action or filter is defined by this revision.

## 7.4.  Protocol Buffers Encoding for Streaming Clients

RESTCONF/JSON (RFC 7951) is the canonical encoding for this
document's data model, but some clients -- in particular gRPC-
based telemetry and streaming consumers -- prefer a typed binary
wire format. A non-normative Protocol Buffers (proto3) schema,
"inventory-network-interfaces-02.proto", is published
alongside this document.

Relationship to gNMI.  gNMI (gRPC Network Management Interface,
gRPC-based model-agnostic subscribe/get for YANG-modeled data) is
an established streaming option for this data model [GNMI]. Clients
may subscribe to the mounted ietf-interfaces subtree with gNMI;
the service-specific proto is an alternative for clients that want
strongly typed messages without gNMI's generic "TypedValue" and
path-oriented API. It is optional and does not replace gNMI or the
RESTCONF operations in Sections 7.1-7.3.

The proto schema defines:

- "Interface", "Device", and "DeviceInterfaces" messages that
  represent data from both YANG modules in Section 6. The proto
  uses snake_case names and includes helper fields such as
  "type_name" for IANA values outside its finite enum; the proto
  comments document the mapping;

- enum types for "InterfaceType", "AdminStatus", "OperStatus",
  "DuplexMode", "Direction", "ServiceState", and "EventType"
  corresponding to this module's YANG enumerations;

- a unary "DeviceInterfaceInventory.GetDeviceInterfaces" RPC,
  equivalent to the RESTCONF GET in Section 7.1;

- a paginated "ListDeviceInterfaces" RPC that returns
  "ListDeviceInterfacesResponse" with "device" entries plus a
  "next_page_token", for clients iterating over a very large
  fleet; and

- a server-streaming "StreamDeviceInterfaceEvents" RPC that
  pushes a "DeviceInterfaceEvent" (create / update / delete /
  initial snapshot, with a timestamp and optional "changed_fields")
  per interface event. For DELETE, "interface" carries the
  last-known pre-delete snapshot; initial synchronization uses the
  distinct SNAPSHOT event type.

The protobuf schema is informative. Its field naming and helper
enum fields are not a normative wire encoding of YANG; the YANG
modules in Section 6 are authoritative if the representations
differ. Any companion protobuf schema needs to be updated to reflect
the removal of the per-interface inventory "ne-ref".

# 8.  Examples

## 8.1.  Example: LAG, LAG Members, and VLAN Sub-Interfaces

The following shortened RFC 7951 [RFC7951] example shows the full
parent data-tree shape using the worked example in Appendix A. It
is a fixture, not the literal body of the subtree GET in Section
7.1; that GET returns the inner "ietf-interfaces:interfaces"
resource representation. The complete three-router fixture is
identified as "05-device-interfaces-schema-mount-v4.json"; that
companion fixture needs to be updated to match this revision.

The topology relationships and documented-prefix IP addresses come
from the synthetic source payloads. The if-index, MAC address,
admin/oper states, MTU, duplex, and service-state values are
illustrative placeholders, not measurements from a live device.
In the RFC 7951 encoding below, phys-address is base64 because
ietf-interfaces defines it using the YANG binary type.

```json
   {
  "inventory-network-interfaces:network-interface-inventory": {
    "device": [
      {
        "ne-ref": "RouterB.subdomain1.region3.isp.example.net",
        "ietf-interfaces:interfaces": {
          "interface": [
            {
              "name": "TenGigE0/1/0/0",
              "type": "iana-if-type:ethernetCsmacd",
              "admin-status": "up",
              "oper-status": "up",
              "if-index": 10,
              "phys-address": "qrvMCwEA",
              "speed": "10000000000",
              "higher-layer-if": [
                "Bundle-Ether10"
              ],
              "lower-layer-if": [],
              "device-interface-attributes:mtu": 9216,
              "device-interface-attributes:duplex-mode": "full-duplex",
              "device-interface-attributes:service-state": "in-service",
              "device-interface-attributes:floating": false,
              "device-interface-attributes:inventory-ref": {
                "port-ref": "TenGigE0/1/0/0"
              },
              "device-interface-attributes:topology-ref": [
                {
                  "network-ref": "mobilecore.region3.l2",
                  "node-ref": "RouterB.subdomain1.region3.isp.example.net",
                  "tp-ref": "TenGigE0/1/0/0"
                }
              ]
            },
            {
              "name": "TenGigE0/1/0/1",
              "type": "iana-if-type:ethernetCsmacd",
              "admin-status": "up",
              "oper-status": "up",
              "higher-layer-if": [
                "Bundle-Ether10"
              ],
              "lower-layer-if": [],
              "device-interface-attributes:mtu": 9216,
              "device-interface-attributes:duplex-mode": "full-duplex",
              "device-interface-attributes:service-state": "in-service",
              "device-interface-attributes:floating": false
            },
            {
              "name": "Bundle-Ether10",
              "type": "iana-if-type:ieee8023adLag",
              "admin-status": "up",
              "oper-status": "up",
              "higher-layer-if": [],
              "lower-layer-if": [
                "TenGigE0/1/0/0",
                "TenGigE0/1/0/1"
              ],
              "device-interface-attributes:mtu": 9216,
              "device-interface-attributes:service-state": "in-service",
              "device-interface-attributes:floating": false,
              "device-interface-attributes:ip-address": [
                "198.51.100.0"
              ]
            },
            {
              "name": "TenGigE0/2/0/0",
              "type": "iana-if-type:ethernetCsmacd",
              "admin-status": "up",
              "oper-status": "up",
              "higher-layer-if": [
                "TenGigE0/2/0/0.100",
                "TenGigE0/2/0/0.200",
                "TenGigE0/2/0/0.300"
              ],
              "lower-layer-if": []
            },
            {
              "name": "TenGigE0/2/0/0.100",
              "type": "iana-if-type:l2vlan",
              "admin-status": "up",
              "oper-status": "up",
              "higher-layer-if": [],
              "lower-layer-if": [
                "TenGigE0/2/0/0"
              ],
              "device-interface-attributes:vlan-id": 100,
              "device-interface-attributes:service-state": "in-service",
              "device-interface-attributes:floating": false,
              "device-interface-attributes:ip-address": [
                "192.0.2.1"
              ]
            },
            {
              "name": "TenGigE0/2/0/1",
              "type": "iana-if-type:ethernetCsmacd",
              "admin-status": "down",
              "oper-status": "down",
              "higher-layer-if": [],
              "lower-layer-if": [],
              "device-interface-attributes:mtu": 1500,
              "device-interface-attributes:duplex-mode": "full-duplex",
              "device-interface-attributes:service-state": "out-of-service",
              "device-interface-attributes:floating": true,
              "device-interface-attributes:inventory-ref": {
                "port-ref": "TenGigE0/2/0/1"
              },
              "device-interface-attributes:notes": "Inventory-only physical port; no topology termination point or link"
            }
          ]
        }
      }
    ]
  }
}
```

Note how "higher-layer-if"/"lower-layer-if" give the LAG<->member
and VLAN<->parent relationships directly, in both directions, with
no topology-network traversal required; "inventory-ref" and
"topology-ref" remain available for clients that still need to
correlate back into inventory or topology views. "mtu",
"duplex-mode", "service-state", and "floating" (Appendix E)
provide commonly needed attributes that RFC 8343 does not define
in its base module.

The "TenGigE0/2/0/1" entry illustrates that "floating" can be
"true" (no topology-ref entries) while "inventory-ref" remains
populated with the known chassis-port component.


--- back

# Status of This Memo
{:numbered="false"}

This Internet-Draft is submitted in full conformance with the
provisions of BCP 78 and BCP 79. The contact and organization
fields are working placeholders and must be completed before
submission.

Internet-Drafts are working documents of the Internet Engineering
Task Force (IETF).  Note that other groups may also distribute
working documents as Internet-Drafts.  The list of current
Internet-Drafts is at https://datatracker.ietf.org/drafts/current/.

Internet-Drafts are draft documents valid for a maximum of six
months and may be updated, replaced, or obsoleted by other
documents at any time.  It is inappropriate to use Internet-Drafts
as reference material or to cite them other than as "work in
progress".

This Internet-Draft will expire on 29 April 2027.

# Copyright Notice
{:numbered="false"}

Copyright (c) 2026 IETF Trust and the persons identified as the
document authors.  All rights reserved.  This document is subject
to BCP 78 and the IETF Trust's Legal Provisions Relating to IETF
Documents (https://trustee.ietf.org/license-info) in effect on the
date of publication of this document.

# 9.  Security Considerations

The parent device list exposes device identifiers and interface
inventory relationships. Implementations MUST apply authorization
to both the parent device list and every mounted interface schema.
Access control on a parent data node does not by itself establish
the same policy for mounted data. Servers MUST configure and test
access control for mounted instances in accordance with RFC 8528
and the Network Configuration Access Control Model (NACM, RFC 8341).

The YANG augmentation modules defined here are operational/read-only
(their added nodes are config false). The mounted ietf-interfaces
schema itself contains both configuration and operational-state
nodes; this document does not claim that the whole mounted schema
is config false. A GET-only facade MUST reject write methods unless
a separate write contract and authorization policy are specified.

Interface identifiers, topology relationships, and device inventory
can be sensitive network information. Implementations SHOULD limit
access by user, tenant, and device scope, protect RESTCONF/API
transport with TLS, and apply equivalent authorization to streamed
events and unary reads.

# 10.  IANA Considerations

This document has no IANA actions. The "urn:example" namespaces
and "example-" module names are placeholders for this individual
draft. If the work is adopted by an IETF working group, final
namespaces and module names will be selected and appropriate
registrations requested in a later revision.

# 11.  References

## 11.1.  Normative References

[RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
Requirement Levels", BCP 14, RFC 2119, March 1997.

[RFC6991]  Schoenwaelder, J., Ed., "Common YANG Data Types",
RFC 6991, July 2013.

[RFC7950]  Bjorklund, M., Ed., "The YANG 1.1 Data Modeling
Language", RFC 7950, August 2016.

[RFC7951]  Lhotka, L., "JSON Encoding of Data Modeled with YANG",
RFC 7951, August 2016.

[RFC8040]  Bierman, A., Bjorklund, M., and K. Watsen, "RESTCONF
Protocol", RFC 8040, January 2017.

[RFC8174]  Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC
2119 Key Words", BCP 14, RFC 8174, May 2017.

[RFC8340]  Bjorklund, M. and L. Berger, Ed., "YANG Tree
Diagrams", BCP 215, RFC 8340, March 2018.

[RFC8341]  Bierman, A. and M. Bjorklund, "Network Configuration
Access Control Model", STD 91, RFC 8341, March 2018.

[RFC8343]  Bjorklund, M., "A YANG Data Model for Interface
Management", RFC 8343, March 2018.

[RFC8345]  Clemm, A., Medved, J., Varga, R., Bahadur, N.,
Ananthakrishnan, H., and X. Liu, "A YANG Data Model
for Network Topologies", RFC 8345, March 2018.

[RFC8407]  Bierman, A., "Guidelines for Authors and Reviewers of
Documents Containing YANG Data Models", BCP 216,
RFC 8407, October 2018.

[RFC8525]  Bierman, A., Bjorklund, M., and K. Watsen, "YANG
Library", RFC 8525, March 2019.

[RFC8528]  Bjorklund, M. and L. Lhotka, "YANG Schema Mount",
RFC 8528, March 2019.

## 11.2.  Informative References

[RFC8346]  Clemm, A., Medved, J., Varga, R., Liu, X.,
Ananthakrishnan, H., and N. Bahadur, "A YANG Data
Model for Layer 3 Topologies", RFC 8346, March 2018.

[RFC8944]  Dong, J., Wei, X., Wu, Q., Boucadair, M., and A.
Liu, "A YANG Data Model for Layer 2 Network
Topologies", RFC 8944, November 2020.

[IVY-INVENTORY]
"A YANG Data Model for Network and Network Element
Inventory", draft-ietf-ivy-network-inventory-yang-19
(work in progress).

[IVY-TOPOLOGY]
"A YANG Data Model for Network Inventory Topology",
draft-ietf-ivy-network-inventory-topology-11 (work in
progress).

[RFC2863]  McCloghrie, K. and F. Kastenholz, "The Interfaces
Group MIB", RFC 2863, June 2000.  (IF-MIB)

[RFC3635]  Flick, J., "Definitions of Managed Objects for the
Ethernet-like Interface Types", RFC 3635,
September 2003.  (EtherLike-MIB)

[RFC4293]  Routhier, S., Ed., "Management Information Base for
the Internet Protocol (IP)", RFC 4293, April 2006.
(IP-MIB)

[RFC4363]  Levi, D., Ed., and D. Harrington, Ed., "Definitions of
Managed Objects for Bridges with Traffic Classes,
Multicast Filtering, and Virtual LAN Extensions",
RFC 4363, January 2006.  (Q-BRIDGE-MIB)

[RFC6933]  Bierman, A., Romascanu, D., Quittek, J., and M.
Chandramouli, "Entity MIB (Version 4)", RFC 6933,
May 2013.  (ENTITY-MIB)

[OC-INTERFACES]
OpenConfig Working Group, "openconfig-interfaces.yang",
OpenConfig public models repository
(github.com/openconfig/public), 'mtu' leaf.

[OC-ETHERNET]
OpenConfig Working Group, "openconfig-if-ethernet.yang",
OpenConfig public models repository, 'duplex-mode' and
'auto-negotiate' leaves.

[OC-AGGREGATE]
OpenConfig Working Group, "openconfig-if-aggregate.yang",
OpenConfig public models repository,
'aggregation/state/member' and 'ethernet/config/
aggregate-id' leaves.

[OC-VLAN]  OpenConfig Working Group, "openconfig-vlan.yang",
OpenConfig public models repository,
'vlan/match/single-tagged/config/vlan-id' leaf.

[OC-IF-IP] OpenConfig Working Group, "openconfig-if-ip.yang",
OpenConfig public models repository,
'ipv4/addresses/address/config/ip' leaf.

[OC-TERMINAL-DEVICE]
OpenConfig Working Group, "openconfig-terminal-device.
yang", OpenConfig public models repository,
'logical-channels/channel/config/{rate-class,
trib-protocol}' leaves.

[OC-PLATFORM-PORT]
OpenConfig Working Group, "openconfig-platform-port.
yang", OpenConfig public models repository,
'hardware-port' leaf.

[GNMI]     OpenConfig Working Group, "gRPC Network Management
Interface (gNMI) Specification", OpenConfig public
reference repository, gNMI specification.

# Appendix A.  Full Worked Example (mobilecore.region3)

The complete source data used to ground Section 8 --
three IOS XR routers, their IVY network-inventory, IVY
inventory-topology, RFC 8944 L2 topology, and RFC 8346-style L3
topology -- plus the full parent data-tree fixture from which the
Section 7.1 mounted-subtree response is derived -- are published as
a companion file set:

```text
   01-ivy-network-inventory.json
   02-ivy-inventory-topology.json
   03-l2-topology.json
   04-l3-topology.json
   05-device-interfaces-schema-mount-v4.json   (worked fixture)
   inventory-network-interfaces@2026-10-29.yang (parent)
   device-interface-attributes@2026-10-29.yang (mounted)
   inventory-network-interfaces-03.proto (Section 7.4)
```

These files are not reproduced in full here for length. They are
companion working artifacts; this individual draft does not assign
a stable public artifact URL. The parent YANG module and worked fixture
need to be updated to reflect the renamed top-level container and the
mount point on the device list. The mounted YANG module, worked fixture,
and protobuf schema also need to be updated to reflect the removal of
the per-interface inventory "ne-ref".

Table 1 summarizes RouterB's interfaces and their representation in
the source inventory and topology data.

```text
+=====================+===========+===============================+
| Interface           | Type      | Appears in                    |
+=====================+===========+===============================+
| TenGigE0/2/0/0       | physical  | physical, L2                  |
| TenGigE0/2/0/0.100-  | VLAN      | L2, L3                         |
|   .300               |           |                                |
| TenGigE0/1/0/0,      | physical, | physical, L2                  |
|   TenGigE0/1/0/1      | LAG member|                                |
| Bundle-Ether10        | LAG       | L2, L3                         |
| TenGigE0/2/0/1,       | physical, | physical only (inventory-only)|
|   HundredGigE0/2/0/1  | unused    |                                |
+=====================+===========+===============================+
```

Table 1: RouterB interfaces vs. existing models

# Appendix B.  Changes from -00

- Renamed the module from "ietf-device-interface-inventory" to
  "inventory-network-interfaces" and changed the namespace
  from "urn:ietf:params:xml:ns:yang:..." to
  "urn:example:ns:yang:...", per RFC 8407 Sections 4.1 and 4.8
  (reserved IETF prefixes). The "ietf-" prefix will be
  reinstated when and if the draft is adopted by a WG.

- Restored the "Expires:" header line and the "This Internet-
  Draft will expire on ..." Status-of-this-Memo sentence,
  without which idnits and the datatracker submission tooling
  reject the draft.

- Added an "Author's Address" section.

- Clarified the semantics of the "floating" leaf: it is "true"
  when the interface's "topology-ref" list is empty, independent
  of whether "inventory-ref" is set. The -00 description keyed
  "floating" off "inventory-ref" presence, which contradicted
  the companion JSON instance.

- Changed "alarm-ref" from a leaf-list of instance-identifier to
  a structured list keyed by the RFC 8632 (resource,
  alarm-type-id, alarm-type-qualifier) tuple, with
  "require-instance false" on the "resource" leaf, so clients
  can match against the alarm-list directly without parsing an
  opaque instance-identifier and so a purged alarm does not
  break validation.

- Renamed the mount-point parent container from "device-root" to
  "interfaces-root" so the container identifier and the
  mount-point label were no longer the same string. In -01, the
  mount-point label remained "device-root".

- Explicitly marked the operational-state leaves
  ("current-alarm-severity", "alarm-ref", "floating")
  "config false".

- Updated the companion ".proto" to match: renamed package to
  "example.device_interface_inventory.v1", added gNMI
  relationship note, structured "alarm_ref" as a message type
  instead of a repeated string, collapsed "EventType" to just
  CREATED/UPDATED/DELETED with an optional "changed_fields"
  list, added pagination to "ListDeviceInterfaces", added
  "type_name" sibling of "type" for IANA interface-type values
  not covered by the enum, added "interface_count" convenience
  field on "Device", and dropped the "Service" suffix from the
  service name.

- Added a stronger OpenConfig corroboration paragraph to
  Section 4 (OpenConfig's addition of "aggregate-id" after the
  fact validates the design choice to use RFC 8343 Section 3.3's
  generic mechanism).

- Updated Section 5.3 "topology-ref" entry to state plainly that
  OpenConfig has no equivalent, removing a weak LLDP comparison.

- Converted the embedded YANG "//" line comment to "/* */" block
  comment form for strict parser compatibility.

- Fixed the worked-example MTU values in the companion JSON
  (physical Ethernet 1500, LAG and LAG members 9216) and added a
  non-cleared alarm example on RouterB's "TenGigE0/2/0/1" to
  illustrate the new structured "alarm-ref" shape.

- Several editorial and nit fixes identified in the -00 review:
  broken-up abstract paragraph, removal of the awkward
  "SNMP-IF-MIB-derived" compound, YANG revision statement no
  longer refers to a "companion document" ambiguously, and so
  on.

# Appendix C.  Changes from -01

- Split the YANG model into parent and mounted modules. The parent
  defines the device list and mount point; the mounted augmentation
  is loaded with ietf-interfaces.

- Replaced cross-mount leafrefs in the mounted schema with string
  correlation identifiers to respect RFC 8528 mount-jail rules.
  The parent device ne-ref remains a leafref in the parent schema.

- Specified that servers advertise the mounted module set through
  RFC 8528 schema-mount data and RFC 8525 YANG Library.

- Corrected the tree diagram, embedded YANG modules, JSON module
  prefixes, RESTCONF path and fixture/response distinction.

- Corrected security text: only the augmentation nodes are
  config false; ietf-interfaces retains its configuration and
  operational-state model.

- Removed IANA registration actions for the placeholder
  "urn:example" namespace; added RFC 8525 and RFC 7950 references.

- Corrected the proto mapping text and event semantics.

# Appendix D.  Changes from -02

- Removed the mounted YANG attributes "current-alarm-severity"
  and "alarm-ref"; no alarm model or alarm correlation is defined
  by this draft revision.

- Removed the corresponding protobuf enum/message/fields and
  omitted alarm fields from the worked JSON fixture. Protobuf field
  numbers 30 and 31 and their names remain reserved to prevent
  accidental reuse by generated clients. The protobuf package is
  advanced to v2 because removing fields changes the generated
  application API.

- Removed the alarm severity attribute row and alarm-specific
  IF-MIB/OpenConfig references from the current attribute mapping
  and references list. The proto reserves the former field numbers
  and names for wire compatibility but exposes no alarm fields.
  Historical change appendices may mention alarm fields that
  existed in earlier revisions.

# Appendix E.  Comparison Against Commonly Implemented Interface Attributes

The attributes in this document were compared against the kind of
attribute set commonly exposed by
deployed, SNMP-IF-MIB-derived interface-inventory implementations
(vendor-neutral by construction; no specific product or API is
referenced). Table 2 summarizes the comparison; attributes already
covered by RFC 8343 or by this document's augmentation are marked
"covered", attributes this document now adds as a result of the
comparison are marked "added (Section 6)", and attributes judged
out of scope for this revision are marked "deferred" with a
one-line reason.

```text
+==========================+====================+==================+
| Commonly implemented      | Disposition        | Notes            |
| interface attribute       |                     |                  |
+==========================+====================+==================+
| port/interface name        | covered            | if:name          |
|   (various forms)          |                    |                  |
| description                | covered            | if:description   |
| interface type              | covered           | if:type          |
| admin status,                | covered         | if:admin-status, |
|   operational status          |                 | if:oper-status   |
| interface index              | covered           | if:if-index      |
| MAC / physical address       | covered           | if:phys-address  |
| interface speed               | covered          | if:speed         |
| link up/down trap enable      | covered          | if:link-up-down- |
|                               |                   |  trap-enable    |
| maximum transmission unit      | added (Sec. 6)  | 'mtu'; RFC 8343  |
|   (MTU)                        |                 | Sec. 4 leaves   |
|                                |                  | this unmapped   |
| duplex mode                    | added (Sec. 6)  | 'duplex-mode'   |
| signal direction                | added (Sec. 6)| 'direction'     |
| transport layer rate            | added (Sec. 6)| 'layer-rate'    |
| interface service/lifecycle     | added (Sec. 6)| 'service-state' |
|   state                         |               |                 |
| no-topology-reference flag     | added (Sec. 6)| 'floating'      |
| separate port, connector, and    | deferred     | distinct        |
|   equipment identifiers          |              | connector-vs-   |
|                                   |              | port identity   |
|                                   |              | not modeled;    |
|                                   |              | 'inventory-ref. |
|                                   |              | port-ref' is an |
|                                   |              | approximation   |
| physical connector presence       | deferred    | optics-presence |
|   flag                            |             | not modeled     |
| promiscuous-mode flag, SPAN/      | deferred     | niche/internal; |
|   RSPAN monitoring flag,          |              |  no operator-   |
|   internal location index,        |             |  facing need    |
|   soak timers, supported MTU      |              |  identified yet |
|   range, shared risk resource     |              |                 |
|   group id, entity alias          |              |                 |
|   mapping identifier, LAG end     |              |                 |
|   identifier, alias name          |              |                 |
+==========================+====================+==================+
```

Table 2: Commonly implemented attribute comparison

The "added" row items are incorporated directly into the YANG
module in Section 6 and the tree diagram in Section 5.2; the
"deferred" items are recorded here so a future revision does not
have to repeat this comparison from scratch.

# Appendix F.  OpenConfig Comparisons

## F.1.  Interface Layering

The bidirectional LAG/VLAN stacking design in Section 5.1 is
validated by real-world experience in OpenConfig: when
openconfig-interfaces alone proved insufficient for LAG
membership queries, openconfig-if-aggregate added both directions
of the same relationship RFC 8343 Section 3.3 provides
generically -- an aggregate interface's "aggregation/state/
member" leaf-list points down at its member interfaces, and each
member's "ethernet/config/aggregate-id" leaf points back up at
its aggregate -- for the specific case of LAGs. That OpenConfig
had to add this reverse pointer explicitly, after the fact and
for one interface type, is strong practical evidence that a
generic bidirectional stacking mechanism is required at the
interface-model layer. This document relies on the RFC 8343
generic mechanism instead of a LAG-specific one so that the same
"higher-layer-if"/"lower-layer-if" pair also covers VLAN
sub-interfaces and other interface-type stacking without a
separate leaf per interface type.

## F.2.  Attribute Comparisons

The following comparisons identify related OpenConfig leaves and
cases for which no direct counterpart was identified:

- "inventory-ref/port-ref": openconfig-platform-port's
  "hardware-port" leaf provides a similar per-device binding.

- "topology-ref": no stored interface-level topology reference was
  identified.

- "vlan-id": openconfig-vlan defines
  "/interfaces/interface/subinterfaces/subinterface/vlan/match/
  single-tagged/config/vlan-id", with the same range (1-4094).

- "ip-address": openconfig-if-ip defines
  "/interfaces/interface/subinterfaces/subinterface/ipv4/addresses/
  address/config/ip" for IPv4 addresses.

- "mtu": openconfig-interfaces defines
  "/interfaces/interface/config/mtu" as uint16, matching this
  document's type.

- "duplex-mode": openconfig-if-ethernet defines
  "/interfaces/interface/ethernet/config/duplex-mode" with FULL and
  HALF values, and models auto-negotiation separately with
  "auto-negotiate". This document's "auto" value is a convenience
  for servers that have not yet resolved a negotiated duplex.

- "direction": no single generic leaf was identified. A related
  concept is the client/line role in an openconfig-terminal-device
  logical-channel-assignments entry for optical transport.

- "layer-rate": openconfig-terminal-device defines
  "/terminal-device/logical-channels/channel/config/
  {rate-class,trib-protocol}" with identityref values such as
  PROT_OTU4 and PROT_ODU4.

- "service-state": no interface-scoped counterpart was identified.

- "floating": no equivalent boolean was identified. An absent
  openconfig-platform-port "hardware-port" indicates no port binding;
  it does not establish whether an interface has topology references.

- "notes": no counterpart was identified for this unstructured field.

# Author's Address

# Appendix G.  SNMP MIB Object Mapping (Informative)

This appendix lists, for each augmentation node, a commonly referenced
SNMP object that carries related information in deployed implementations.
This mapping is informative and does not constrain implementations.

```text
   +================+=============================+================+
   | Node(s)        | Closest SNMP object         | Notes          |
   +================+=============================+================+
   | inventory-ref  | ENTITY-MIB entAliasMapping* | ifIndex -> ent |
   | topology-ref   | none                        |                |
   | vlan-id        | Q-BRIDGE-MIB dot1qVlan*     | not IF-MIB      |
   | ip-address     | IP-MIB ipAddressIfIndex     |                |
   | mtu            | IF-MIB ifMtu                | not in RFC8343  |
   | duplex-mode    | EtherLike-MIB dot3StatsDuplexStatus | not IF-MIB |
   | direction      | none                        |                |
   | layer-rate     | none                        |                |
   | service-state  | none                        |                |
   | floating       | none                        |                |
   | notes          | none                        |                |
   +================+=============================+================+
```

TBD
Email: TBD
