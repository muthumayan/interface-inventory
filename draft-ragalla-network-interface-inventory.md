---
title: "Network-wide Interface Inventory"
abbrev: "Network Interface Inventory"
category: std

docname: draft-ragalla-network-interface-inventory-latest
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
  github: https://github.com/muthumayan/interface-inventory
  latest: https://datatracker.ietf.org/doc/draft-ragalla-device-interface-inventory/

author:
 -
  fullname: Ramkrishna Galla
  organization: Cisco Systems, Inc.
 -
  fullname: Muthumayan Madhayyan
  organization: Cisco Systems, Inc.


normative:
  RFC2119:
  RFC6991:
  RFC7950:
  RFC7951:
  RFC8040:
  RFC8174:
  RFC8340:
  RFC8341:
  RFC8343:
  RFC8345:
  RFC8344:
  RFC8407:
  RFC8525:
  RFC8528:

informative:
  RFC8346:
  RFC8944:
  IVY-INVENTORY:
    title: "A YANG Data Model for Network and Network Element Inventory"
    target: https://datatracker.ietf.org/doc/draft-ietf-ivy-network-inventory-yang/
  IVY-TOPOLOGY:
    title: "A YANG Data Model for Network Inventory Topology"
    target: https://datatracker.ietf.org/doc/draft-ietf-ivy-network-inventory-topology/
  OC-INTERFACES:
    title: "OpenConfig Interfaces Model"
    target: https://github.com/openconfig/public/blob/master/release/models/interfaces/openconfig-interfaces.yang
  OC-ETHERNET:
    title: "OpenConfig Ethernet Model"
    target: https://github.com/openconfig/public/blob/master/release/models/interfaces/openconfig-if-ethernet.yang
  OC-AGGREGATE:
    title: "OpenConfig Aggregate Interface Model"
    target: https://github.com/openconfig/public/blob/master/release/models/interfaces/openconfig-if-aggregate.yang
  OC-VLAN:
    title: "OpenConfig VLAN Model"
    target: https://github.com/openconfig/public/blob/master/release/models/vlan/openconfig-vlan.yang
  OC-IF-IP:
    title: "OpenConfig IP Interface Model"
    target: https://github.com/openconfig/public/blob/master/release/models/interfaces/openconfig-if-ip.yang
  OC-TERMINAL-DEVICE:
    title: "OpenConfig Terminal Device Model"
    target: https://github.com/openconfig/public/blob/master/release/models/optical-transport/openconfig-terminal-device.yang
  OC-PLATFORM-PORT:
    title: "OpenConfig Platform Port Model"
    target: https://github.com/openconfig/public/blob/master/release/models/platform/openconfig-platform-port.yang
  GNMI:
    title: "gRPC Network Management Interface (gNMI) Specification"
    target: https://github.com/openconfig/gnmi

...

--- abstract

This document defines a YANG data model for a network-wide inventory of
physical and logical interfaces, organized by network element. The model
provides interface attributes, operational state, and layering
relationships, with correlations to network inventory and topology data.

--- middle

# Introduction

Network operators and management systems need a consistent inventory
of physical and logical interfaces across network elements. Such an
inventory supports per-device retrieval of interface attributes and
state, as well as correlation with network inventory and topology models.

An SDN controller can expose this inventory to northbound clients, such
as higher-level controllers and automation systems, to support
network-wide operations.

{{RFC8343}} defines interface data, including operational state
and interface-layering relationships, for a server. The IVY network
inventory model {{IVY-INVENTORY}} describes network elements and their
components, while {{RFC8345}} and the IVY inventory topology
model {{IVY-TOPOLOGY}} describe topology and its relationship to inventory.
These models provide complementary information.

This document defines a network-wide interface inventory organized by
network element. It reuses {{RFC8343}} interface data and provides
correlations to network inventory and topology. Sections 5 and 6
describe the data model, Section 7 describes operations, and Appendix
A provides a worked example. Discovery and reconciliation of the
underlying data are implementation-specific.

# Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they
appear in all capitals, as shown here.

Tree diagrams used in this document follow the notation defined in
{{RFC8340}}.

The terms "network element" (NE), "component", and "port" are
used as defined in {{IVY-INVENTORY}}.  The terms "network", "node",
"termination point" (TP), and "supporting-node"/"supporting-
termination-point" are used as defined in {{RFC8345}}.

# Problem Statement

A SDN controller may collect interface data from multiple network elements. {{RFC8343}} provides an interface model for a network-element, including interface state and layering. The controller also keeps an inventory of all network elements and their physical component inventory. Depending on the layer in which the controller operates it also keeps track of the topological view of the network (L1, L2 or L3).

The IVY inventory model {{IVY-INVENTORY}} describes the network-element inventory, while {{RFC8345}} and the IVY inventory topology model {{IVY-TOPOLOGY}} describe topology and its relationship to inventory. While these models are self-sufficient in mapping topology to physical inventory, they lack the relationship that ties the packet forwarding construct (a.k.a interfaces) to the physical inventory.

A northbound client (like a multi-domain controller or an OSS) would need to correlate information from several datasets (inventory, topology, interfaces) in order to reconstruct a network wide view.

As an example, a controller can learn from the {{RFC8345}} topology model which link connects two network devices. If that link is realized as a link aggregation group (LAG) and the link degrades (for example, due to loss/errors on one member), troubleshooting typically requires identifying the member interfaces of the LAG on each device. A controller may already collect the LAG and member-interface relationships while building its topology view, but {{RFC8345}} does not provide a device-scoped interface inventory (with layering) that can be correlated directly to the topology termination points.

This document's proposed model allows the controller to expose that per-device interface inventory and its correlations northbound, so that clients can navigate from a topology link to the underlying per-device member interfaces.

As another example, a northbound performance management system may want to correlate the topological state with performance metrics from specific interfaces. Mere knowledge of physical ports and of termination points may not suffice when LAG or VLAN interfaces are involved.

This proposed model aggregates interface information from multiple network devices and presents them contextually combining inventory and topological references. The model's
completeness and accuracy depend on the data available to the server and on the controller's discovery and reconciliation mechanisms.


# Relationship to Existing Work

{{RFC8343}} Section 3.3 ("Interface Layering") already defines the generic mechanism this document needs: two read-only leaf-lists, "higher-layer-if" and "lower-layer-if", present on every "/interfaces/interface" entry, explicitly described as the YANG equivalent of the IF-MIB "ifStackTable" ({{RFC8343}} Section 4). A LAG's members populate the LAG's "lower-layer-if"; each member's "higher-layer-if" points back at the LAG.  A VLAN sub-interface's "lower-layer-if" points at its parent physical interface.  This document does not redefine LAG or VLAN semantics; it relies
entirely on {{RFC8343}} Section 3.3 for interface stacking.

{{RFC8528}} ("YANG Schema Mount") defines a mechanism for attaching a schema tree at a mount point. Appendix A illustrates device/LNE-scoped mounted data, but does not prescribe a device key or require one mount per device. This proposal chooses that organization as a data-model design. It uses separate parent and mounted modules. {{RFC8528}} confines paths in a mounted schema to the mount jail; therefore the mounted augmentation represents cross-model inventory/topology identifiers as strings rather than leafrefs into parent-schema data. A server may use shared-schema
parent-reference metadata for XPath evaluation, but this proposal does not require it. This document does not change {{RFC8345}}, {{RFC8346}}, {{RFC8944}}, or the IVY drafts.

# Data Model Overview

## Design Summary

- "network-interface-inventory/device" is keyed by "ne-ref", a leafref into the IVY "network-element" list -- the same device identifier already used by inventory, so no new device identity scheme is introduced.

- Each "device" list entry is a {{RFC8528}} mount point labeled "interfaces-root", under which the server exposes a mounted "ietf-interfaces:interfaces" subtree. The server advertises the mounted module set in schema-mount/YANG Library data.

- Every "if:interface" entry in that mounted subtree is augmented (Section 6) with:

  - an "inventory-ref" carrying the IVY port component identifier, when one is known, within the network element identified by the enclosing device's "ne-ref"; and

  - zero or more "topology-ref" entries carrying string identifiers for the {{RFC8345}} network/node/termination-point views corresponding to the interface. Interfaces may exist without participating in any topology view; in that case, the "topology-ref" list is not present; and

  - a "vlan-id" convenience leaf populated from the corresponding {{RFC8944}}-style topology attributes, so a client does not have to dereference "topology-ref" just to read a VLAN tag. {{RFC8343}} can represent VLAN subinterfaces and their layering relationships, but it does not define a standard leaf for the VLAN tag value itself.

  - IP addressing and per-family MTU are represented using {{RFC8344}} (ietf-ip) within the mounted schema set, rather than being duplicated as additional augmentation leaves.

- "higher-layer-if" and "lower-layer-if" are NOT redefined; they are inherited unmodified from {{RFC8343}}. The server populates this operational view from device/interface data and available LAG/VLAN relationships; the discovery and reconciliation mechanism is implementation-specific.

The following illustration shows the hierarchy and example correlations.
Device identifiers are abbreviated, and RouterX represents another
network element. Arrows indicate relationships, not data containment.

~~~
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
~~~

## Tree Diagram

Parent schema module: inventory-network-interfaces

~~~
  +--ro network-interface-inventory
     +--ro device* [ne-ref]
        +--ro ne-ref             -> /nei:network-inventory/.../ne-id
        +--mp interfaces-root
           (mounted schema set includes ietf-interfaces, ietf-ip, and
            device-interface-attributes)
~~~

Mounted schema augmentation module:
device-interface-attributes

~~~
  augment /if:interfaces/if:interface:
    +--ro inventory-ref?
    |  +--ro port-ref?      string
    +--ro topology-ref* [network-ref node-ref tp-ref]
    |  +--ro network-ref    string
    |  +--ro node-ref       string
    |  +--ro tp-ref         string
    +--ro vlan-id?                  uint16
    +--ro controller-annotation?    string
~~~

## Augmentation Overview

The mounted interface augmentation provides three categories of information:

- Correlation identifiers to connect per-device interface entries to inventory and topology views.

- Convenience leaves derived from topology data, so clients do not need to dereference topology termination points to retrieve a VLAN tag value. {{RFC8343}} can represent VLAN subinterfaces, but does not define a standard leaf for the VLAN tag value.

- An optional controller-provided annotation for information that is specific to the controller's inventory view.

- Most common interface attributes are already addressed in {{RFC8343}}. There are
other interesting interface attributes such IP addressing and IPv4/IPv6 MTU. These are provided by {{RFC8344}} (ietf-ip), which in turn builds on {{RFC8343}}.

Normative definitions for the augmentation are in the YANG modules in
Section 6. Examples are in Appendix A. Informative comparisons and
mappings are in Appendices B and C.

~~~
   +=======================+==============================+
   | Node(s)               | Purpose                      |
   +=======================+==============================+
   | inventory-ref         | Correlate to inventory port  |
   | topology-ref          | Correlate to topology TP(s)  |
   | vlan-id               | Convenience VLAN tag value   |
   | controller-annotation | Controller annotation        |
   +================+=====================================+
~~~

# YANG Modules

The model is split to respect {{RFC8528}} mount-jail semantics. The parent module is loaded in the management system's parent schema. At each mount point, the server advertises a mounted schema set that includes ietf-interfaces, ietf-ip, and device-interface-attributes.

The same mounted module set MAY be shared by all devices when their schemas are identical. The server MUST publish the applicable mount-point schema through {{RFC8528}} schema-mount data and the YANG Library mechanism ({{RFC8525}}).

The mounted augmentation contains no leafref into the parent inventory or topology trees. Its cross-model correlation values are strings: {{RFC8528}} confines mounted paths to the mount jail unless a server configures shared-schema parent-reference metadata. This proposal does not require parent-reference metadata.

## Parent module: inventory-network-interfaces

&lt;CODE BEGINS&gt; file "inventory-network-interfaces@2026-10-29.yang"

~~~ yang
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
     mount point where the mounted schema set contains ietf-interfaces,
     ietf-ip, and device-interface-attributes.

     The module does not define an HTTP API. A server may expose the
     device list and mounted interface data through NETCONF, RESTCONF,
     or another management interface.";

  revision 2026-11-24 {
    description
      "Initial version of the network-wide interface inventory model.";
    reference "draft-ragalla-network-interface-inventory";
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
         be false. The mounted module set MUST include ietf-interfaces,
         ietf-ip, and device-interface-attributes.";

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
~~~

&lt;CODE ENDS&gt;

## Mounted augmentation: device-interface-attributes

&lt;CODE BEGINS&gt; file "device-interface-attributes@2026-11-24.yang"

~~~ yang
module device-interface-attributes {
  yang-version 1.1;
  namespace "urn:example:ns:yang:device-interface-attributes";
  prefix "edia";

  import ietf-interfaces  { prefix if; }

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

  revision 2026-11-24 {
    description
      "Initial version of the device interface attributes augmentation.";
    reference "draft-ragalla-network-interface-inventory";
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
       list. IP addressing and per-family MTU are represented using
       RFC 8344 (ietf-ip) when that module is included in the mounted
       schema set. The base ietf-interfaces module remains unchanged.";

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
         this interface. Interfaces may exist without participating in
         any topology view; in that case, this list is not present.
         An interface represented in multiple topology layers has one
         entry per layer/view.";
      uses topology-ref;
    }

    leaf vlan-id {
      config false;
      type uint16 { range "1..4094"; }
      description
        "Outer VLAN identifier (VLAN tag value) when applicable. This is
         a convenience attribute: RFC 8343 can represent VLAN
         subinterfaces, but does not define a standard leaf for the VLAN
         tag value itself; implementations typically derive it from L2
         configuration and/or topology attributes. When the interface is
         not VLAN-tagged (for example, a physical parent interface), this
         leaf is not present.";
      reference
        "RFC 8944; openconfig-vlan single-tagged vlan-id";
    }
    leaf controller-annotation {
      config false;
      type string;
      description
        "Optional controller-provided annotation for the interface.";
    }
  }
}
~~~

&lt;CODE ENDS&gt;

# RESTCONF Operations

These examples use the RESTCONF URI form defined by {{RFC8040}} for a resource beneath a schema-mount point. They assume the parent and mounted schemas described in Section 6 are advertised by the server through schema-mount/YANG Library data. Implementations that do not expose RESTCONF may provide equivalent data through another API.

## Retrieve All Interfaces on a Device

A client retrieves every interface on a device with a single GET against the mounted interfaces container. URI path key values MUST be percent-encoded when they contain reserved characters:

~~~ http
   GET /restconf/data/inventory-network-interfaces:\
       network-interface-inventory/device=RouterB.subdomain1.region3.isp.example.net/\
       ietf-interfaces:interfaces
~~~

The response representation is the mounted "ietf-interfaces:interfaces" subtree plus augmentation data. The requested device key supplies the network-element context for any "inventory-ref/port-ref" in that subtree.
Appendix A.2 shows a shortened full parent data-tree fixture for context; it is not the literal body of this subtree GET.

## Retrieve a Single Interface

~~~ http
   GET /restconf/data/inventory-network-interfaces:\
       network-interface-inventory/device=RouterB.subdomain1.region3.isp.example.net/\
       ietf-interfaces:interfaces/\
       interface=Bundle-Ether10
~~~

## Filtering by Interface Type

RESTCONF does not define generic server-side predicate filtering on list entries. The base operation in this document returns the interface list; {{RFC8040}} "fields" and "depth" may be used for payload selection. A future revision MAY specify a YANG action or another explicit filter contract for high-interface-count devices; no such action or filter is defined by this revision.

## Protocol Buffers Encoding for Streaming Clients

RESTCONF/JSON ({{RFC7951}}) is the canonical encoding for this document's data model, but some clients -- in particular gRPC- based telemetry and streaming consumers -- prefer a typed binary wire format. A non-normative Protocol Buffers (proto3) schema, "inventory-network-interfaces-02.proto", is published alongside this document.

Relationship to gNMI.  gNMI (gRPC Network Management Interface, gRPC-based model-agnostic subscribe/get for YANG-modeled data) is an established streaming option for this data model {{GNMI}}. Clients may subscribe to the mounted ietf-interfaces subtree with gNMI; the service-specific proto is an alternative for clients that want strongly typed messages without gNMI's generic "TypedValue" and path-oriented API. It is optional and does not replace gNMI or the RESTCONF operations in Sections 7.1-7.3.

The proto schema defines:

- "Interface", "Device", and "DeviceInterfaces" messages that represent data from both YANG modules in Section 6. The proto uses snake_case names and includes helper fields such as "type_name" for IANA values outside its finite enum; the proto comments document the mapping;

- enum types for "InterfaceType", "AdminStatus", "OperStatus", "DuplexMode", "Direction", "ServiceState", and "EventType" corresponding to this module's YANG enumerations;

- a unary "DeviceInterfaceInventory.GetDeviceInterfaces" RPC, equivalent to the RESTCONF GET in Section 7.1;

- a paginated "ListDeviceInterfaces" RPC that returns "ListDeviceInterfacesResponse" with "device" entries plus a "next_page_token", for clients iterating over a very large
  fleet; and

- a server-streaming "StreamDeviceInterfaceEvents" RPC that pushes a "DeviceInterfaceEvent" (create / update / delete / initial snapshot, with a timestamp and optional "changed_fields") per interface event. For DELETE, "interface" carries the last-known pre-delete snapshot; initial synchronization uses the distinct SNAPSHOT event type.

The protobuf schema is informative. Its field naming and helper enum fields are not a normative wire encoding of YANG; the YANG modules in Section 6 are authoritative if the representations differ. Any companion protobuf schema needs to be updated to reflect the removal of the per-interface inventory "ne-ref".

# Security Considerations

The parent device list exposes device identifiers and interface
inventory relationships. Implementations MUST apply authorization
to both the parent device list and every mounted interface schema.
Access control on a parent data node does not by itself establish
the same policy for mounted data. Servers MUST configure and test
access control for mounted instances in accordance with {{RFC8528}}
and the Network Configuration Access Control Model (NACM, {{RFC8341}}).

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

# IANA Considerations

This document has no IANA actions. The "urn:example" namespaces
and "example-" module names are placeholders for this individual
draft. If the work is adopted by an IETF working group, final
namespaces and module names will be selected and appropriate
registrations requested in a later revision.

--- back

# Illustrative Example

This section uses a synthetic three-router network to show how the proposed model brings together interface data, physical inventory, and topology references. It first describes the network and the behavior the example is intended to illustrate. It then shows the corresponding mounted interface inventory and the correlations exposed by the model.

## Sample Network

The topology payloads describe the following three-router connectivity.  RouterA, RouterB, and RouterC are network elements in subdomain1.region3.isp.example.net.

                            RouterA
            TenGigE0/2/0/0   /   \ HundredGigE0/2/0/0
                            /     \
                           /       \
                          /         \
       VLAN 100/200/300  /           \ 100G physical link
                        /             \
                       /               \
       TenGigE0/2/0/0 /                 \ HundredGigE0/2/0/0
                RouterB =============== RouterC
                        Bundle-Ether10
                            (LAG)
        Members at each end: TenGigE0/1/0/0 and TenGigE0/1/0/1

RouterA-to-RouterB connection has VLAN subinterfaces 100, 200, and 300 over physical TenGigE port.  RouterB and RouterC connect through the Bundle-Ether10 LAG; the two member links are shown below it. RouterA and RouterC have a direct HundredGigE link.

RouterB also has ports: TenGigE0/2/0/1 and HundredGigE0/2/0/1 that do not participate in topology i.e., there is no link or termination point.

The example is intended to illustrate three client tasks:

- navigating from a topology connection to the device interfaces that realize it, including the members of a LAG;
- following interface-layering relationships between a physical interface and its VLAN subinterfaces; and
- distinguishing a known inventory port from a topology termination point, as demonstrated by RouterB's inventory-only ports.

These relationships can be represented without requiring a client to reconstruct them by joining unrelated datasets. The mounted interface tree provides the per-device interface state and layering, while the augmentation supplies the inventory and topology correlations.

RouterB is chosen as one sample router to showcase how its interfaces are represented in network interface inventory and to relate them to topology and physical inventory models. RouterB has a combination of VLAN, Bundle and physical interfaces, making it a good choice to cover the concept:

-  TenGigE0/2/0/0: physical interface in the physical inventory and L2 topology views.
-  TenGigE0/2/0/0.100 through .300: VLAN subinterfaces in L2 and L3 topology.
-  TenGigE0/1/0/0 and TenGigE0/1/0/1: physical LAG members.
-  Bundle-Ether10: LAG represented in L2 and L3.
-  TenGigE0/2/0/1 and HundredGigE0/2/0/1: inventory-only ports.

## Interface Inventory Representation

The proposed model covers RouterB's interface inventory through one device entry keyed by its network-element reference. Under that entry, the mounted "ietf-interfaces:interfaces" subtree carries the standard interface identity, type, state, speed, IP addressing, MTU, and layering relationships. The mounted augmentation adds:

- "inventory-ref" when an interface can be associated with a physical inventory port;
- "topology-ref" when the interface is represented by one or more topology termination points; and
- "vlan-id" and "controller-annotation" for the additional interface information defined by this proposal.

The following shortened {{RFC7951}} example shows the full parent data-tree shape using the worked example in this appendix. It is a fixture, not the literal body of the subtree GET in Section 7.1; that GET returns the inner "ietf-interfaces:interfaces" resource representation. The complete three-router fixture is identified as "05-network-interfaces-schema-mount-v4.json"; that companion fixture needs to be updated to match this revision.

The topology relationships and documented-prefix IP addresses come from the synthetic source payloads. The if-index, MAC address, and admin/oper states are illustrative placeholders, not measurements from a live device. The phys-address values use the canonical colon-separated hexadecimal form defined for {{RFC6991}}'s phys-address type.

~~~ json
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
              "phys-address": "00:01:02:03:04:05",
              "speed": "10000000000",
              "higher-layer-if": [
                "Bundle-Ether10"
              ],
              "lower-layer-if": [],
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
              "if-index": 11,
              "phys-address": "00:01:02:03:04:08",
              "higher-layer-if": [
                "Bundle-Ether10"
              ],
              "lower-layer-if": [],
              "device-interface-attributes:inventory-ref": {
                "port-ref": "TenGigE0/1/0/1"
              },
              "device-interface-attributes:topology-ref": [
                {
                  "network-ref": "mobilecore.region3.l2",
                  "node-ref": "RouterB.subdomain1.region3.isp.example.net",
                  "tp-ref": "TenGigE0/1/0/1"
                }
              ]
            },
            {
              "name": "Bundle-Ether10",
              "type": "iana-if-type:ieee8023adLag",
              "admin-status": "up",
              "oper-status": "up",
              "if-index": 12,
              "phys-address": "00:01:02:03:04:09",
              "higher-layer-if": [],
              "lower-layer-if": [
                "TenGigE0/1/0/0",
                "TenGigE0/1/0/1"
              ],
              "device-interface-attributes:topology-ref": [
                {
                  "network-ref": "mobilecore.region3.l2",
                  "node-ref": "RouterB.subdomain1.region3.isp.example.net",
                  "tp-ref": "Bundle-Ether10"
                }
              ],
              "ietf-ip:ipv4": {
                "mtu": 9216,
                "address": [
                  {
                    "ip": "198.51.100.0",
                    "prefix-length": 32
                  }
                ]
              }
            },
            {
              "name": "TenGigE0/2/0/0",
              "type": "iana-if-type:ethernetCsmacd",
              "admin-status": "up",
              "oper-status": "up",
              "if-index": 20,
              "phys-address": "00:01:02:03:04:06",
              "higher-layer-if": [
                "TenGigE0/2/0/0.100",
                "TenGigE0/2/0/0.200",
                "TenGigE0/2/0/0.300"
              ],
              "lower-layer-if": [],
              "device-interface-attributes:inventory-ref": {
                "port-ref": "TenGigE0/2/0/0"
              },
              "device-interface-attributes:topology-ref": [
                {
                  "network-ref": "mobilecore.region3.l2",
                  "node-ref": "RouterB.subdomain1.region3.isp.example.net",
                  "tp-ref": "TenGigE0/2/0/0"
                }
              ]
            },
            {
              "name": "TenGigE0/2/0/0.100",
              "type": "iana-if-type:l2vlan",
              "admin-status": "up",
              "oper-status": "up",
              "if-index": 21,
              "higher-layer-if": [],
              "lower-layer-if": [
                "TenGigE0/2/0/0"
              ],
              "device-interface-attributes:topology-ref": [
                {
                  "network-ref": "mobilecore.region3.l2",
                  "node-ref": "RouterB.subdomain1.region3.isp.example.net",
                  "tp-ref": "TenGigE0/2/0/0.100"
                }
              ],
              "device-interface-attributes:vlan-id": 100,
              "ietf-ip:ipv4": {
                "mtu": 1500,
                "address": [
                  {
                    "ip": "192.0.2.1",
                    "prefix-length": 26
                  }
                ]
              }
            },
            {
              "name": "TenGigE0/2/0/0.200",
              "type": "iana-if-type:l2vlan",
              "admin-status": "up",
              "oper-status": "up",
              "if-index": 23,
              "higher-layer-if": [],
              "lower-layer-if": [
                "TenGigE0/2/0/0"
              ],
              "device-interface-attributes:topology-ref": [
                {
                  "network-ref": "mobilecore.region3.l2",
                  "node-ref": "RouterB.subdomain1.region3.isp.example.net",
                  "tp-ref": "TenGigE0/2/0/0.200"
                }
              ],
              "device-interface-attributes:vlan-id": 200,
              "ietf-ip:ipv4": {
                "mtu": 1500,
                "address": [
                  {
                    "ip": "192.0.2.65",
                    "prefix-length": 26
                  }
                ]
              }
            },
            {
              "name": "TenGigE0/2/0/0.300",
              "type": "iana-if-type:l2vlan",
              "admin-status": "up",
              "oper-status": "up",
              "if-index": 24,
              "higher-layer-if": [],
              "lower-layer-if": [
                "TenGigE0/2/0/0"
              ],
              "device-interface-attributes:topology-ref": [
                {
                  "network-ref": "mobilecore.region3.l2",
                  "node-ref": "RouterB.subdomain1.region3.isp.example.net",
                  "tp-ref": "TenGigE0/2/0/0.300"
                }
              ],
              "device-interface-attributes:vlan-id": 300,
              "ietf-ip:ipv4": {
                "mtu": 1500,
                "address": [
                  {
                    "ip": "192.0.2.129",
                    "prefix-length": 26
                  }
                ]
              }
            },
            {
              "name": "TenGigE0/2/0/1",
              "type": "iana-if-type:ethernetCsmacd",
              "admin-status": "down",
              "oper-status": "down",
              "if-index": 22,
              "phys-address": "00:01:02:03:04:07",
              "higher-layer-if": [],
              "lower-layer-if": [],
              "device-interface-attributes:inventory-ref": {
                "port-ref": "TenGigE0/2/0/1"
              },
              "device-interface-attributes:controller-annotation": "Inventory-only physical port; no topology termination point or link"
            }
          ]
        }
      }
    ]
  }
}
~~~

Note how "higher-layer-if"/"lower-layer-if" give the LAG<->member and VLAN<->parent relationships directly, in both directions, with no topology-network traversal required; "inventory-ref" and "topology-ref" remain available for clients that still need to correlate back into inventory or topology views. Interface capacity, IP addressing, and per-family MTU come from the mounted base modules; they are not duplicated by the augmentation. "controller-annotation" allows a controller to attach an optional free-form note to an interface.

The "TenGigE0/2/0/1" entry illustrates that an interface may have no "topology-ref" entries while "inventory-ref" remains populated with the known chassis-port component.

The complete source data used in this appendix -- three routers, their IVY network-inventory, IVY inventory-topology, {{RFC8944}} L2 topology, and {{RFC8346}}-style L3 topology -- plus the full parent data-tree fixture from which the Section 7.1 mounted-subtree response is derived -- are published as a companion file set:

*  examples/mobilecore-region3/01-ivy-network-inventory.json
*  examples/mobilecore-region3/02-ivy-inventory-topology.json
*  examples/mobilecore-region3/03-l2-topology.json
*  examples/mobilecore-region3/04-l3-topology.json
*  examples/mobilecore-region3/05-network-interface-schema-mount-v4.json
*  inventory-network-interfaces@2026-11-25.yang (parent module)
*  device-interface-attributes@2026-11-25.yang (mounted module)
*  inventory-network-interfaces.proto (Section 7.4)

The YANG modules, protobuf schema, source topology payloads, and worked JSON fixture are included in this repository.  The JSON fixture is a synthetic example; interface state/MAC/ifIndex values not present in the source topology payloads are illustrative, not live-device observations.

# Northbound Interface Attribute Coverage

A controller's northbound interface should expose the attributes that
allow clients to identify an interface, understand its operational
state, assess its capacity, and correlate it with physical inventory
and topology. These attributes support common consumer workflows such
as fault isolation, capacity analysis, service visualization, and
navigation from a topology link to the interfaces that realize it.

Table 2 summarizes the interface attributes considered useful to
northbound consumers. It identifies whether each attribute is provided
by the base ietf-interfaces model or added by this document.

| Interface attribute | Northbound value | Model source |
| --- | --- | --- |
| Interface name | Identify the interface and correlate client data | `if:name` |
| Description | Present an operator-facing label or purpose | `if:description` |
| Interface type | Classify physical, LAG, VLAN, and logical interfaces | `if:type` |
| Administrative and operational status | Determine configured and current availability | `if:admin-status`, `if:oper-status` |
| Interface index | Correlate with device-level interface records | `if:if-index` |
| Physical address | Identify the interface-layer address when present | `if:phys-address` |
| Interface speed | Support capacity and performance analysis | `if:speed` |
| Link-up/down trap enable | Indicate whether link-state notifications are enabled | `if:link-up-down-trap-enable` |
| IP addressing and MTU | Support reachability, service, and packet-size analysis | `ietf-ip:ipv4/mtu`, `ietf-ip:ipv6/mtu` ({{RFC8344}}) |
| Inventory reference | Correlate an interface with a physical port | Section 6 augmentation |
| Topology reference | Correlate an interface with a topology termination point | Section 6 augmentation |
| VLAN identifier | Identify the VLAN associated with a subinterface | Section 6 augmentation |
| Interface layering | Navigate between physical, aggregate, and VLAN interfaces | `higher-layer-if`, `lower-layer-if` ({{RFC8343}}) |

Table 2: Attributes useful to northbound consumers

Together, these attributes provide the information needed by
northbound clients to identify, monitor, and correlate interfaces across
interface, inventory, and topology views. Vendor-specific hardware,
monitoring, policy, and lifecycle attributes are outside the scope of
this revision and may be added by future augmentations or
platform-inventory models.

# OpenConfig Interoperability

OpenConfig models are commonly used by controllers to discover
interface, platform, aggregation, VLAN, and IP information from network
devices. This document does not require devices to implement the
proposed network-wide inventory model directly. A controller may collect
data using OpenConfig models, correlate it with inventory and topology
data, and expose the resulting interface inventory northbound using the
model defined here.

The proposed model therefore acts as a controller-facing normalization
model. It preserves standard interface attributes, adds references to
physical inventory and topology objects, and represents interface
layering consistently across devices that may use different southbound
data models.

For example, OpenConfig aggregate and platform models can supply LAG
membership and physical-port information. The controller can map that
information to "higher-layer-if", "lower-layer-if", and
"inventory-ref" in the northbound representation. Similarly,
OpenConfig VLAN and IP models can supply VLAN identifiers, IP addresses,
and MTU values without requiring those attributes to be duplicated in
this augmentation.
