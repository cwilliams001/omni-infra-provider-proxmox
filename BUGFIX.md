# Bug Fix: Respect User-Specified Node in Machine Class Configuration

## Problem Statement

The Omni infrastructure provider for Proxmox was ignoring the `node` field specified in machine class configurations. Despite users explicitly configuring VMs to be created on specific Proxmox cluster nodes (e.g., hv02, hv04), the provider consistently placed all VMs on a single node (typically the one with the most available memory, often hv01).

This issue was originally reported in [Issue #33](https://github.com/siderolabs/omni-infra-provider-proxmox/issues/33).

### Observed Behavior (Before Fix)

- Machine class correctly stored the `node` field in provider data
- All VMs were created on the same node (usually hv01) regardless of configuration
- Node-level resource distribution and workload balancing were impossible
- No error messages indicated the configuration was being ignored

### Expected Behavior

- When `node` is specified in machine class provider data, VMs should be created on that specific node
- When `node` is NOT specified, the provider should automatically select a node (backward compatibility)

## Root Cause Analysis

The issue was located in the `pickNode` provisioning step (`internal/pkg/provider/provision.go:47-79`).

### The Problem

The `pickNode` step always executed at the start of VM provisioning and **unconditionally** selected a node based on available memory, without checking if the user had already specified a preferred node in the provider data:

```go
provision.NewStep("pickNode", func(ctx context.Context, logger *zap.Logger, pctx provision.Context[*resources.Machine]) error {
    nodes, err := p.proxmoxClient.Nodes(ctx)
    if err != nil {
        return err
    }

    // ... logic to find node with most free memory ...

    pctx.State.TypedSpec().Value.Node = nodeName  // ALWAYS overwrites

    logger.Info("picked the node for the Proxmox VM", zap.String("node", nodeName))

    return nil
}),
```

**Key Issues:**
1. Never read the provider data to check for user-specified node
2. Always overwrote the node field with auto-selected value
3. Executed before provider data was unmarshaled in subsequent steps

## Solution

Modified the `pickNode` step to:

1. **First** unmarshal the provider data and check if `data.Node` is specified
2. **Use the configured node** if provided (skip automatic selection)
3. **Fall back to automatic selection** only when no node is specified (preserves backward compatibility)

### Code Changes

**File:** `internal/pkg/provider/provision.go` (lines 47-97)

```go
provision.NewStep("pickNode", func(ctx context.Context, logger *zap.Logger, pctx provision.Context[*resources.Machine]) error {
    // First, check if a node was explicitly specified in the provider data
    var data Data

    err := pctx.UnmarshalProviderData(&data)
    if err != nil {
        return err
    }

    // If a node was specified in the provider data, use it
    if data.Node != "" {
        pctx.State.TypedSpec().Value.Node = data.Node

        logger.Info("using configured node for the Proxmox VM", zap.String("node", data.Node))

        return nil
    }

    // Otherwise, automatically pick a node based on available memory
    nodes, err := p.proxmoxClient.Nodes(ctx)
    if err != nil {
        return err
    }

    var (
        maxFree  uint64
        nodeName string
    )

    if len(nodes) == 0 {
        return fmt.Errorf("no nodes available")
    }

    for _, node := range nodes {
        if node.Status != "online" {
            continue
        }

        freeMem := node.MaxMem - node.Mem
        if freeMem > maxFree {
            maxFree = freeMem
            nodeName = node.Node
        }
    }

    pctx.State.TypedSpec().Value.Node = nodeName

    logger.Info("automatically picked the node for the Proxmox VM", zap.String("node", nodeName))

    return nil
}),
```

### Log Message Changes

Updated log messages to distinguish between the two behaviors:

- **User-specified node**: `"using configured node for the Proxmox VM"`
- **Auto-selected node**: `"automatically picked the node for the Proxmox VM"`

## Testing

### Test Environment
- Omni version: 1.3.4
- Proxmox cluster: 3 nodes (hv01, hv02, hv04)
- Deployment: On-premises via Docker Compose

### Test Cases

1. **Node explicitly specified (hv02)**
   - Machine class with `node: "hv02"`
   - Result: VM created on hv02 ✅
   - Log: `"using configured node for the Proxmox VM" node="hv02"`

2. **Node explicitly specified (hv04)**
   - Machine class with `node: "hv04"`
   - Result: VM created on hv04 ✅
   - Log: `"using configured node for the Proxmox VM" node="hv04"`

3. **Node NOT specified (backward compatibility)**
   - Machine class without `node` field
   - Result: VM created on node with most free memory ✅
   - Log: `"automatically picked the node for the Proxmox VM" node="hv01"`

4. **Multiple distributed control planes**
   - Created 3 control plane machine classes (hv01-control-plane, hv02-control-plane, hv04-control-plane)
   - Applied cluster template with distributed control planes
   - Result: 1 control plane per node + 1 worker per node ✅

### Real-World Deployment

Successfully deployed a production cluster with:
- 3 control planes distributed across hv01, hv02, hv04
- 3 workers distributed across hv01, hv02, hv04
- All nodes placed correctly according to machine class configuration

## Backward Compatibility

✅ **Fully backward compatible**

- Existing configurations without a `node` field continue to work exactly as before
- Automatic node selection based on available memory still functions when node is not specified
- No breaking changes to API or configuration schema

## Impact

This fix enables:
- ✅ Explicit node placement for VMs
- ✅ Proper resource distribution across Proxmox cluster nodes
- ✅ High availability by distributing control planes across physical hosts
- ✅ Workload balancing across the cluster
- ✅ Use of node-specific hardware or storage configurations

## Additional Notes

### AI Assistance

This fix was developed with assistance from Claude (Anthropic's AI assistant) to:
- Analyze the codebase and identify the root cause
- Design and implement the solution
- Create comprehensive testing procedures
- Document the changes

The human developer verified the fix through real-world testing in a production-like environment.

### Related Files

- `internal/pkg/provider/provision.go` - Main fix location
- `internal/pkg/provider/data.go` - Provider data structure (already had `Node` field)
- `cmd/omni-infra-provider-proxmox/data/schema.json` - Schema definition (already documented `node` parameter)

## References

- Original Issue: https://github.com/siderolabs/omni-infra-provider-proxmox/issues/33
- Provider Data Structure: `internal/pkg/provider/data.go:9`
- Schema Documentation: `cmd/omni-infra-provider-proxmox/data/schema.json:16-19`
