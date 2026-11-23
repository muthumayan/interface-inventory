---
title: "A YANG Data Model for Device-Scoped Interface Inventory Retrieval"
abbrev: "Device Interface Inventory Retrieval"
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

The RFC 8345 family of network-topology data models (RFC 8345,
RFC 8346, RFC 8944) and the IVY network-inventory drafts
(draft-ietf-ivy-network-inventory-yang,
draft-ietf-ivy-network-inventory-topology) together give a rich,
multi-layer view of a network, but neither provides a single,
device-scoped query that returns every interface on a network
element -- physical ports, Link Aggregation Group (LAG) bundles,
LAG member interfaces, and VLAN sub-interfaces -- the way an
IF-MIB ifTable walk does for a single device.

This document defines that gap precisely, shows it against a
worked three-router example, and proposes two YANG modules: a parent
device/mount-point model and a mounted interface-attribute
augmentation. The design reuses ietf-interfaces [RFC8343] at a
per-device mount point using the YANG Schema Mount mechanism
[RFC8528], with string correlation references back to RFC 8345
topology and IVY inventory data.

The resulting attribute set was additionally checked against
attributes commonly exposed by deployed IF-MIB-derived interface-
inventory implementations to confirm no commonly used attribute
(MTU, duplex mode, signal direction, transport layer rate, and
interface-level service state) was left out; those attributes are
included in the mounted augmentation.

Every attribute this document adds is further cross-referenced,
with a definition and an example, against the closest SNMP MIB
object (IF-MIB, ENTITY-MIB, EtherLike-MIB, IP-MIB, Q-BRIDGE-MIB)
and the closest published OpenConfig leaf, where one exists, so
implementers coming from either lineage can map directly onto
these modules.

--- middle

# 1. Introduction


Operators and network management systems routinely need to answer
a simple question about a single device: "what interfaces does
this box have, what type is each one, what state is it in, and how
are they layered on top of each other (physical port -> LAG ->
VLAN sub-interface)?"  On classical SNMP-managed devices this is
answered by one MIB walk of ifTable/ifXTable plus ifStackTable.

The IETF YANG topology family, RFC 8345 [RFC8345] and its
companions RFC 8346 [RFC8346] (L3) and RFC 8944 [RFC8944] (L2),
together with the in-progress IVY network-inventory drafts
[IVY-INVENTORY][IVY-TOPOLOGY], model interfaces only as
"termination points" that are scoped to a particular network
("network-id") and layer.  This is the right design for topology
and connectivity use cases, but it leaves no equivalent of
ifTable: there is no data node, container, or query that is keyed
purely by device and returns every interface on it.

This document:

- states the gap precisely (Section 3), grounded in a worked
three-router example;

- explains how RFC 8343 [RFC8343] Section 3.3 provides generic
interface-layering state and how RFC 8528 [RFC8528] can mount a
schema at a device/LNE scope chosen by a data model (Section 4);
neither RFC defines the proposed facade URI; and

- specifies separate parent and mounted YANG modules, RESTCONF
examples, and a non-normative proto3 mapping (Sections 5-8).

# 2. Conventions and Terminology


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

# 3. Problem Statement


This section is grounded in a synthetic but representative
three-router dataset, "mobilecore.region3" (three IOS XR routers,
RouterA/RouterB/RouterC), reproduced in full in Appendix A.  It
consists of an IVY network-inventory document, an IVY inventory-
topology document, an RFC 8944 L2 topology document, and an
RFC 8346-style L3 topology document, all consistent with each
other and following [IVY-INVENTORY] draft-19 and [IVY-TOPOLOGY]
draft-11.

## 3.1. Interfaces Are Modeled Only Inside Topologies


A single logical interface on RouterB, "Bundle-Ether10", appears
in up to three separate "network-id" trees:

- "mobilecore.region3.physical" (IVY inventory-topology): the
underlying physical ports only, not the LAG itself;

- "mobilecore.region3.l2" (RFC 8944): the LAG as an
"ietf-network-topology:termination-point" with "lag: true" and
a "member-link-tp" leaf-list; and

- "mobilecore.region3.l3": the LAG's IP-addressed sub-interface.

There is no single resource keyed by "RouterB" alone that returns
all of this.

## 3.2. No Reverse LAG-Membership Reference


In the L2 topology, "Bundle-Ether10" carries a forward-only
leaf-list:

"member-link-tp": [ "TenGigE0/1/0/0", "TenGigE0/1/0/1" ]

Given "TenGigE0/1/0/0" alone, a client cannot determine that it is
a LAG member without enumerating every node's termination points
looking for a bundle whose "member-link-tp" contains it.  The same
is true in reverse for VLAN sub-interfaces: "TenGigE0/2/0/0.100"
is correlated to its parent port "TenGigE0/2/0/0" only via
"supporting-termination-point", which is a per-topology
relationship, not a device-level one.

## 3.3. No Device-Scoped Query


Because every termination point lives inside a "network" keyed by
"network-id", there is no way to ask, in one request, "every
interface on network element X" without already knowing which of
the (potentially many) topology network-id values are relevant and
issuing one request per network-id, per layer.

## 3.4. No Interface-Level Operational State


RFC 8345 intentionally does not define "admin-status",
"oper-status", "if-index", "phys-address", "speed", or interface
counters; these are explicitly the domain of RFC 8343
[RFC8343].  A device-level interface query needs both: the
RFC 8343 state model and a device-scoped anchor that RFC 8345 does
not provide.

## 3.5. Worked Example


Table 1 shows every interface on RouterB in the worked example
(Appendix A) and which document(s) above it currently appears in.

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

No single query returns this table today.

## 3.6. Comparison Against Commonly Implemented Interface Attributes


To validate that this document's attribute set is sufficient, it
was compared against the kind of attribute set commonly exposed by
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
   | no-physical-component flag      | added (Sec. 6)| 'floating'      |
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

# 4. Relationship to Existing Work


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

# 5. Data Model Overview


## 5.1. Design Summary


- "device-interfaces/device" is keyed by "ne-ref", a leafref into
the IVY "network-element" list -- the same device identifier
already used by inventory, so no new device identity scheme is
introduced.

- Each "device" list entry contains an "interfaces-root"
container that declares an RFC 8528 mount point (label
"device-root"), under which the server exposes a mounted
"ietf-interfaces:interfaces" subtree. The server advertises the
mounted module set in schema-mount/YANG Library data.

- Every "if:interface" entry in that mounted subtree is augmented
(Section 6) with:

*  an "inventory-ref" carrying string identifiers for the IVY
network element/component, when one exists; and

*  zero or more "topology-ref" entries carrying string
identifiers for the RFC 8345 network/node/termination-point
views corresponding to the interface; and

*  "vlan-id" and "ip-address" convenience leaves populated
from the corresponding RFC 8944/RFC 8346-style topology
attributes, so a client does not have to dereference
"topology-ref" just to read a VLAN tag or an IP address.

- "higher-layer-if" and "lower-layer-if" are NOT redefined; they
are inherited unmodified from RFC 8343. The server populates
this operational view from device/interface data and available
LAG/VLAN relationships; the discovery and reconciliation
mechanism is implementation-specific.

## 5.2. Tree Diagram


Parent schema module: example-device-interface-inventory

```text
     +--ro device-interfaces
        +--ro device* [ne-ref]
           +--ro ne-ref             -> /nei:network-inventory/.../ne-id
           +--ro interfaces-root
              +--mp device-root
```

(mounted schema set includes ietf-interfaces and
example-device-interface-attributes)

Mounted schema augmentation module:
example-device-interface-attributes

augment /if:interfaces/if:interface:
```text
       +--ro inventory-ref?
       |  +--ro ne-ref?        string
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


## 5.3. Attribute Definitions, Protocol/Model References, and Examples


This section gives a plain-language definition, an example value,
and -- where one exists -- the closest SNMP MIB object and the
closest published OpenConfig leaf for every attribute this
document's augmentation adds to a mounted "if:interface" entry.
OpenConfig references are included because, independently of this
document, OpenConfig has already faced and partly solved the same
"device-scoped interface with LAG/VLAN stacking" problem; where an
OpenConfig leaf exists it is cited as corroborating prior art, and
where none exists that is stated explicitly rather than left
silent, since an operator migrating between the two model
families needs to know which attributes have no counterpart.

inventory-ref (container: ne-ref, port-ref)
Definition: Correlates this interface to the physical or
logical component in the IVY network-inventory it is bound to,
so a client can still answer "what hardware is this" without
a second query.
MIB reference: ENTITY-MIB.entAliasMappingTable, RFC 6933
(maps ifIndex to entPhysicalIndex).
OpenConfig reference: openconfig-platform-port's
"hardware-port" leaf on an interface performs the same
per-device binding, but has no cross-device "ne-ref" concept.
Example: { "ne-ref": "RouterB.subdomain1.region3.bt.uk",
"port-ref": "TenGigE0/1/0/0" }

topology-ref (list: network-ref, node-ref, tp-ref)
Definition: Correlates this interface to one or more RFC 8345
termination points (physical, L2, and/or L3 views).
MIB reference: none.
OpenConfig reference: none; OpenConfig does not model stored
topology references at the interface level at all.
Example: [ { "network-ref": "mobilecore.region3.l2",
"node-ref": "RouterB.subdomain1.region3.bt.uk",
"tp-ref": "TenGigE0/1/0/0" } ]

vlan-id
Definition: The outer 802.1Q VLAN tag carried by this
sub-interface.
MIB reference: Q-BRIDGE-MIB.dot1qVlanStaticTable (RFC 4363);
not part of IF-MIB itself.
OpenConfig reference:
openconfig-vlan:/interfaces/interface/subinterfaces/
subinterface/vlan/match/single-tagged/config/vlan-id -- same
range (1-4094), confirming the chosen type.
Example: 100

ip-address
Definition: IP address(es) bound to this interface.
MIB reference: IP-MIB.ipAddressIfIndex, RFC 4293.
OpenConfig reference:
openconfig-if-ip:/interfaces/interface/subinterfaces/
subinterface/ipv4/addresses/address/config/ip.
Example: [ "192.0.2.1" ]

mtu
Definition: Maximum transmission unit, in octets, for this
interface.
MIB reference: IF-MIB.ifMtu, RFC 2863. RFC 8343 Section 4
explicitly declines to map ifMtu, leaving it to interface-
type-specific augmentations -- which is exactly what this
leaf is.
OpenConfig reference:
openconfig-interfaces:/interfaces/interface/config/mtu
(type uint16); this document's "mtu" leaf is typed uint16 to
match.
Example: 9216 (jumbo MTU on a LAG bundle)

duplex-mode
Definition: The Ethernet duplex mode the interface is
operating in.
MIB reference: EtherLike-MIB.dot3StatsDuplexStatus, RFC 3635
(not part of IF-MIB proper).
OpenConfig reference:
openconfig-if-ethernet:/interfaces/interface/ethernet/config/
duplex-mode, enumeration { FULL, HALF }. OpenConfig models
auto-negotiation as a separate boolean "auto-negotiate" rather
than a third duplex-mode value; this document's "auto" value
is a convenience rollup for servers that have not yet resolved
a negotiated duplex.
Example: "full-duplex"

direction
Definition: Signal direction through this interface or
sub-interface.
MIB reference: none.
OpenConfig reference: none as a single generic leaf; the
closest analog is the implicit client/line role assigned in
an openconfig-terminal-device logical-channel-assignments
entry, which only applies to optical transport ports.
Example: "bidir"

layer-rate
Definition: The transport layer rate of the signal carried by
this interface (e.g. Ethernet, OTU4, ODU4, IP).
MIB reference: none.
OpenConfig reference:
openconfig-terminal-device:/terminal-device/logical-channels/
channel/config/{rate-class,trib-protocol}, identityref values
such as PROT_OTU4 / PROT_ODU4.
Example: "OTU4"

service-state
Definition: The interface's lifecycle/service state (e.g.
planned, in-test, in-service, maintenance, out-of-service),
distinct from the physical "admin-status"/"oper-status" pair.
MIB reference: none.
OpenConfig reference: none at interface scope; this document
treats "service-state" as a genuine gap in both IF-MIB and
OpenConfig, not merely in RFC 8343.
Example: "in-service"

floating
Definition: True when this interface is not currently
participating in any topology view, i.e. its "topology-ref"
list is empty. The leaf is independent of "inventory-ref": a
"floating" interface MAY still carry an "inventory-ref" when
the underlying physical port is known to the inventory system
but is not wired into any topology (for example, an unused
chassis port). This semantic was clarified in -01; -00
incorrectly described this leaf as keyed off "inventory-ref".
MIB reference: none.
OpenConfig reference: none as an explicit boolean; an absent
openconfig-platform-port "hardware-port" leaf signals the same
condition implicitly.
Example: false (true for an unused, inventory-only physical
port such as RouterB's "TenGigE0/2/0/1" in Appendix A, which
still carries an "inventory-ref" to the chassis port).

notes
Definition: Free-text annotation for anything not captured by
a structured leaf above.
MIB/OpenConfig reference: none; intentionally unstructured.
Example: "Inventory-only physical port; no topology
termination point or link"

# 6. YANG Modules


The model is split to respect RFC 8528 mount-jail semantics. The
parent module is loaded in the management system's parent schema.
At each mount point, the server advertises a mounted schema set that
includes ietf-interfaces and example-device-interface-attributes.
The same mounted module set MAY be shared by all devices when their
schemas are identical. The server MUST publish the applicable
mount-point schema through RFC 8528 schema-mount data and the YANG
Library mechanism (RFC 8525).

The mounted augmentation contains no leafref into the parent
inventory or topology trees. Its cross-model correlation values are
strings: RFC 8528 confines mounted paths to the mount jail unless a
server configures shared-schema parent-reference metadata. This
proposal does not require parent-reference metadata.

## 6.1. Parent module: example-device-interface-inventory


<CODE BEGINS> file "example-device-interface-inventory@2026-10-29.yang"

module example-device-interface-inventory {
yang-version 1.1;
namespace "urn:example:ns:yang:example-device-interface-inventory";
prefix "edii";

import ietf-yang-schema-mount { prefix yangmnt; }
import ietf-network-inventory { prefix nei; }

organization "TBD";
contact "TBD";

description
"Parent data model for a device-scoped interface inventory facade.
It identifies a network element and defines an RFC 8528 schema
mount point where the mounted schema set contains ietf-interfaces
and example-device-interface-attributes.

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

container device-interfaces {
config false;
description
"Device-scoped interface inventory mount points.";

list device {
key "ne-ref";
description
"One entry for each network element served by this management
system. The ne-ref leafref is evaluated in the parent schema,
outside the mounted schema jail.";

leaf ne-ref {
type leafref {
path "/nei:network-inventory/nei:network-elements/"
```text
                + "nei:network-element/nei:ne-id";
```

}
description
"Identifier of the network element owning the mounted
interface list.";
}

container interfaces-root {
description
"Mount point parent. The mount label is device-root. The
server advertises the mounted schema through RFC 8528
schema-mounts data, using an inline or shared-schema entry
as appropriate. For this read-only inventory view, the
schema-mount config value MUST be false. The mounted module
set MUST include ietf-interfaces and
example-device-interface-attributes.";

yangmnt:mount-point "device-root" {
description
"RFC 8528 mount point for the per-device interface model.";
}
}
}
}
}

<CODE ENDS>

## 6.2. Mounted augmentation: example-device-interface-attributes


<CODE BEGINS> file "example-device-interface-attributes@2026-10-29.yang"

module example-device-interface-attributes {
yang-version 1.1;
namespace "urn:example:ns:yang:example-device-interface-attributes";
prefix "edia";

import ietf-interfaces  { prefix if; }
import ietf-inet-types  { prefix inet; }

organization "TBD";
contact "TBD";

description
"Read-only augmentation of a mounted ietf-interfaces schema for
device-scoped interface inventory. This module is intended to be
included in the mounted schema set at the device-root mount point
defined by example-device-interface-inventory.

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
"Reference to the inventory element/component associated with
this interface. Values are strings because IVY inventory data
may be outside the mounted schema.";

leaf ne-ref {
type string;
description
"Network-element identifier in the parent inventory. When
inventory-ref is present, implementations SHOULD populate
this value; the enclosing device list's ne-ref is authoritative.";
}
leaf port-ref {
type string;
description
"Component identifier of the physical port, when one exists.";
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
"Correlation to the physical/logical inventory component, if
one is known.";
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
"RFC 2863 IF-MIB ifMtu; openconfig-interfaces mtu";
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

<CODE ENDS>
# 7. RESTCONF Operations


These examples use the RESTCONF URI form defined by RFC 8040 for a
resource beneath a schema-mount point. They assume the parent and
mounted schemas described in Section 6 are advertised by the server
through schema-mount/YANG Library data. Implementations that do not
expose RESTCONF may provide equivalent data through another API.

## 7.1. Retrieve All Interfaces on a Device


A client retrieves every interface on a device with a single GET
against the mounted interfaces container. URI path key values MUST
be percent-encoded when they contain reserved characters:

GET /restconf/data/example-device-interface-inventory:\
device-interfaces/device=RouterB.subdomain1.region3.bt.uk/\
interfaces-root/ietf-interfaces:interfaces

The response representation is the mounted
"ietf-interfaces:interfaces" subtree plus augmentation data.
Section 8.1 shows a shortened full parent data-tree fixture for
context; it is not the literal body of this subtree GET.

## 7.2. Retrieve a Single Interface


GET /restconf/data/example-device-interface-inventory:\
device-interfaces/device=RouterB.subdomain1.region3.bt.uk/\
interfaces-root/ietf-interfaces:interfaces/\
interface=Bundle-Ether10

## 7.3. Filtering by Interface Type


RESTCONF does not define generic server-side predicate filtering
on list entries. The base operation in this document returns the
interface list; RFC 8040 "fields" and "depth" may be used for
payload selection. A future revision MAY specify a YANG action or
another explicit filter contract for high-interface-count devices;
no such action or filter is defined by this revision.

## 7.4. Protocol Buffers Encoding for Streaming Clients


RESTCONF/JSON (RFC 7951) is the canonical encoding for this
document's data model, but some clients -- in particular gRPC-
based telemetry and streaming consumers -- prefer a typed binary
wire format. A non-normative Protocol Buffers (proto3) schema,
"example-device-interface-inventory-02.proto", is published
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
differ.

# 8. Examples


## 8.1. Example: LAG, LAG Members, and VLAN Sub-Interfaces


The following shortened RFC 7951 [RFC7951] example shows the full
parent data-tree shape using the worked example in Appendix A. It
is a fixture, not the literal body of the subtree GET in Section
7.1; that GET returns the inner "ietf-interfaces:interfaces"
resource representation. The complete three-router fixture is
published as "05-device-interfaces-schema-mount-v4.json".

The topology relationships and documented-prefix IP addresses come
from the synthetic source payloads. The if-index, MAC address,
admin/oper states, MTU, duplex, and service-state values are
illustrative placeholders, not measurements from a live device.
In the RFC 7951 encoding below, phys-address is base64 because
ietf-interfaces defines it using the YANG binary type. 

{
"example-device-interface-inventory:device-interfaces": {
"device": [
{
"ne-ref": "RouterB.subdomain1.region3.bt.uk",
"interfaces-root": {
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
"example-device-interface-attributes:mtu": 9216,
"example-device-interface-attributes:duplex-mode": "full-duplex",
"example-device-interface-attributes:service-state": "in-service",
"example-device-interface-attributes:floating": false,
"example-device-interface-attributes:inventory-ref": {
"ne-ref": "RouterB.subdomain1.region3.bt.uk",
"port-ref": "TenGigE0/1/0/0"
},
"example-device-interface-attributes:topology-ref": [
{
"network-ref": "mobilecore.region3.l2",
"node-ref": "RouterB.subdomain1.region3.bt.uk",
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
"example-device-interface-attributes:mtu": 9216,
"example-device-interface-attributes:duplex-mode": "full-duplex",
"example-device-interface-attributes:service-state": "in-service",
"example-device-interface-attributes:floating": false
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
"example-device-interface-attributes:mtu": 9216,
"example-device-interface-attributes:service-state": "in-service",
"example-device-interface-attributes:floating": false,
"example-device-interface-attributes:ip-address": [
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
"example-device-interface-attributes:vlan-id": 100,
"example-device-interface-attributes:service-state": "in-service",
"example-device-interface-attributes:floating": false,
"example-device-interface-attributes:ip-address": [
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
"example-device-interface-attributes:mtu": 1500,
"example-device-interface-attributes:duplex-mode": "full-duplex",
"example-device-interface-attributes:service-state": "out-of-service",
"example-device-interface-attributes:floating": true,
"example-device-interface-attributes:inventory-ref": {
"ne-ref": "RouterB.subdomain1.region3.bt.uk",
"port-ref": "TenGigE0/2/0/1"
},
"example-device-interface-attributes:notes": "Inventory-only physical port; no topology termination point or link"
}
]
}
}
}
]
}
}

Note how "higher-layer-if"/"lower-layer-if" give the LAG<->member
and VLAN<->parent relationships directly, in both directions, with
no topology-network traversal required; "inventory-ref" and
"topology-ref" remain available for clients that still need to
correlate back into inventory or topology views. "mtu",
"duplex-mode", "service-state", and "floating" (Section 3.6)
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

# 9. Security Considerations


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

# 10. IANA Considerations


This document has no IANA actions. The "urn:example" namespaces
and "example-" module names are placeholders for this individual
draft. If the work is adopted by an IETF working group, final
namespaces and module names will be selected and appropriate
registrations requested in a later revision.

# 11. References


## 11.1. Normative References


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

## 11.2. Informative References


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

# Appendix A. Full Worked Example (mobilecore.region3)


The complete source data used to ground Section 3 and Section 8 --
three IOS XR routers, their IVY network-inventory, IVY
inventory-topology, RFC 8944 L2 topology, and RFC 8346-style L3
topology -- plus the full parent data-tree fixture from which the
Section 7.1 mounted-subtree response is derived -- are published as
a companion file set:

01-ivy-network-inventory.json
02-ivy-inventory-topology.json
03-l2-topology.json
04-l3-topology.json
05-device-interfaces-schema-mount-v4.json   (worked fixture)
example-device-interface-inventory@2026-10-29.yang (parent)
example-device-interface-attributes@2026-10-29.yang (mounted)
example-device-interface-inventory-03.proto (Section 7.4)

These files are not reproduced in full here for length. They are
companion working artifacts; this individual draft does not assign
a stable public artifact URL.

# Appendix B. Changes from -00


- Renamed the module from "ietf-device-interface-inventory" to
"example-device-interface-inventory" and changed the namespace
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
mount-point label are no longer the same string. The
mount-point label itself remains "device-root".

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

# Appendix C. Changes from -01


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

# Appendix D. Changes from -02


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

# Author's Address


TBD
Email: TBD
