# Auto-traits / node features

With the growing popularity of Redfish we're facing the situation where nodes with the same driver and the same underlying protocol have drastically different capabilities. While some work is under way to provide additional validation (e.g. check that `redfish-virtual-media` is compatible with the given node).

We already have a convenient mechanism to report a node's capabilities: traits. This RFE proposes discovering certain traits automatically.

## User stories and examples

### Optional virtual media

As an Ironic user I want to use *virtual media* with my Redfish nodes, but only if it is available.

```python=
conn = openstack.connect()
node = conn.baremetal.get_node(sys.argv[1])
i_info = {"image_source": "http://server/image"}
if ("IRONIC_REDFISH_VMEDIA" in node.traits
        and node.boot_interface != "redfish-virtual-media"):
    i_info["boot_interface"] = "redfish-virtual-media"
node = conn.baremetal.update_node(node, instance_info=i_info)
conn.baremetal.set_node_provision_state(node, "active")
```

### Pick a node with firmware update capability

As an Ironic user I want to deploy a node that has Redfish firmware update capability.

```python=
conn = openstack.connect()
alloc = conn.baremetal.create_allocation(
    resource_class="baremetal",
    traits=["IRONIC_REDFISH_FIRMWAREUPDATE"])
alloc = conn.baremetal.wait_for_allocation(alloc)
node = conn.baremetal.get_node(sys.argv[1])
conn.baremetal.set_node_provision_state(
    node, "active", deploy_steps=[...])
```

## Proposal

Make the conductor and hardware types automatically discover certain (pre-defined and documented) properties of a node and insert them as traits. The traits will use the prefix `IRONIC_` which is not compatible with *os-traits* and thus won't conflict with any existing traits. The new traits will only be visible in a new API version.

:::warning
TBD: do we report them to *Placement*? If yes, how do we ensure compatibility? Opinion: we probably don't want to report to Placement (and thus make available to end users) anything that an operator hasn't approved.
:::

The format is `IRONIC_<SCOPE>_<FEATURE>[_<SUBFEATURE>]` where:
* `<SCOPE>` is either `CORE` or a hardware type (upper case). The `CORE` scope is reserved for Ironic itself.
* `<FEATURE>` is a specific feature name.
* `<SUBFEATURE>` is an optional subfeature name.

Auto-traits will be collected on `manage` and periodically.

An important aspect of the auto-traits is that they only depend on the hardware type, but not on the combination of interfaces. This may be confusing, but it is done to enable the use case above where an interface is defined based on whether a node has a certain feature.

### List of auto-traits

For `Redfish`:
* `IRONIC_REDFISH_VMEDIA` - has virtual media support.
* `IRONIC_REDFISH_FIRMWAREUPGRADE` - can update firmware.
* `IRONIC_REDFISH_SECUREBOOT` - can manage secure boot.

:::warning
TBD: do we distinguish between Redfish and iDRAC-Redfish? How?
:::