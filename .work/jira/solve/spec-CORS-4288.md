# Implementation Specification: CORS-4288

**JIRA**: https://issues.redhat.com/browse/CORS-4288
**Summary**: Review and update Azure supported machine families for OpenShift docs
**Created**: 2026-02-13

---

## Problem Statement

Current Azure machine type documentation in the OpenShift installer repo lists VM families using internal naming conventions (e.g., `standardDSv2Family`, `standardEISv3Family`) without clear mapping to Azure's official VM Series documentation. This creates confusion when users try to understand:

1. Which Azure VM Series are supported by OpenShift
2. How OpenShift's family names map to Azure's official Series names
3. The relationship between these families and specific SKU examples (e.g., `Standard_DS4_v2`)

**Current Issues**:
- Documentation lists bare family names without context
- No introduction explaining what these families represent
- No linkage to Azure's official VM size documentation
- Users confuse these with Azure ML VM SKU lists
- No guidance on how to use these families when configuring OpenShift

---

## Goals

1. **Improve Clarity**: Reframe documentation to clearly reference Azure VM Series families
2. **Provide Context**: Add introduction and mapping guidance between OpenShift and Azure naming
3. **Link to Official Docs**: Reference Azure's official VM size family documentation
4. **Maintain Accuracy**: Keep the existing tested family list accurate
5. **Add Examples**: Show how families relate to specific instance types in install-config

---

## Deliverables

### 1. Updated `tested_instance_types_x86_64.md`

**Changes**:
- Add comprehensive header explaining what Azure VM families are
- Explain the relationship between family names and specific SKUs
- Link to Azure's official VM size family documentation
- Add table format showing:
  - OpenShift family name
  - Azure Series name
  - Example SKU(s)
  - Use case category (General Purpose, Compute Optimized, Memory Optimized, etc.)
- Organize families by Azure's standard categories
- Add notes about regional availability and instance type selection

### 2. Updated `tested_instance_types_aarch64.md`

**Changes**:
- Apply same structure as x86_64 file
- Add header and mapping guidance
- Link to Azure documentation for ARM64-based VMs
- Organize by categories

### 3. Update `customization.md` (if needed)

**Changes**:
- Add reference to tested instance types documentation
- Update examples to reference the family documentation
- Add guidance on selecting appropriate instance types

---

## Implementation Plan

### Phase 1: Research and Mapping (Documentation Review)

1. **Review Azure's Official Documentation**:
   - VM size families: https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview
   - Understand Azure's categorization:
     - General Purpose (B, D, DS, Dv2, DSv2, Dav4, Dasv4, Dv3, DSv3, Dv4, DSv4, Dv5, DSv5, etc.)
     - Compute Optimized (F, FS, Fsv2, etc.)
     - Memory Optimized (E, ES, Ev3, ESv3, Ev4, ESv4, Ev5, ESv5, M, etc.)
     - Storage Optimized (LS, LSv2, LSv3, etc.)
     - GPU (NC, ND, NV, etc.)
     - High Performance Compute (H, HB, HC, HX, etc.)

2. **Map Azure Series to Family Names**:
   - Create mapping table: `Dsv2-series` → `standardDSv2Family` → `Standard_DS*_v2` VM sizes
   - Document pattern: Azure `{SeriesName}-series` → `standard{SeriesName}Family`

3. **Categorize Existing Families**:
   - Group by Azure's official categories
   - Identify general purpose, compute optimized, memory optimized, etc.

4. **Select Appropriate Example VM Sizes**:
   - Ensure examples meet OpenShift minimum requirements:
     - Control Plane: 8 vCPUs, 32GB RAM minimum
     - Worker: 2 vCPUs, 8GB RAM minimum (x86_64); 4 vCPUs, 16GB RAM (ARM64)
   - Show range of sizes (small, medium, large) where applicable
   - Prioritize commonly used sizes (e.g., Standard_D4s_v3, Standard_D8s_v3)

### Phase 2: Documentation Structure

**New File Structure for `tested_instance_types_x86_64.md`**:

```markdown
# Azure Tested Instance Types (x86_64)

## Overview

OpenShift Container Platform has been tested with the following Azure virtual machine families. These families align with [Azure's official VM size families](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview).

### Understanding Azure VM Families

Azure organizes VM sizes into **Series** (or families) based on workload characteristics and capabilities. Each series contains multiple specific **SKUs** (instance types) with varying vCPU, memory, and storage configurations.

**Understanding the Relationship**:
- **Azure VM Series** (e.g., `Dsv2-series`): Azure's official grouping of VM sizes by generation and capabilities
- **Family Name** (e.g., `standardDSv2Family`): Internal reference name used by OpenShift
- **VM Size Name** (e.g., `Standard_DS4_v2`): Individual instance types you specify in install-config.yaml

**Naming Convention**:
- Azure Series `Dsv2-series` → Family Name `standardDSv2Family`
- Pattern: `{Series}-series` → `standard{Series}Family`

**Selecting an Instance Type**:
When configuring your cluster in `install-config.yaml`, you specify a specific VM size from a tested Azure VM Series. For example:

```yaml
controlPlane:
  platform:
    azure:
      type: Standard_D8s_v3  # From Azure Dsv3-series (standardDSv3Family)
compute:
- platform:
    azure:
      type: Standard_D4s_v3  # From Azure Dsv3-series (standardDSv3Family)
```

**Minimum Requirements**:
- **Control Plane**: Minimum 8 vCPUs, 32GB RAM (e.g., Standard_D8s_v3 or larger)
- **Worker Nodes**: Minimum 2 vCPUs, 8GB RAM for x86_64 (e.g., Standard_D2s_v3 or larger); 4 vCPUs, 16GB RAM for ARM64 (e.g., Standard_D4ps_v5 or larger)

See [limits.md](limits.md) for detailed resource requirements and [customization.md](customization.md) for complete configuration examples.

## Tested VM Families by Category

The following families have been tested with OpenShift. Availability varies by Azure region.

### General Purpose

Balanced CPU-to-memory ratio. Ideal for testing, development, small to medium databases, and low to medium traffic web servers.

| Azure VM Series | Family Name | Example VM Size Name |
|-----------------|-------------|----------------------|
| [Basv2-series](https://learn.microsoft.com/en-us/azure/virtual-machines/basv2) | `standardBasv2Family` | Standard_B4als_v2, Standard_B8als_v2 |
| [BS-series](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes-b-series-burstable) | `standardBSFamily` | Standard_B4s, Standard_B8s |
| [Bsv2-series](https://learn.microsoft.com/en-us/azure/virtual-machines/bsv2-series) | `standardBsv2Family` | Standard_B4ts_v2, Standard_B8ts_v2 |
| [DSv2-series](https://learn.microsoft.com/en-us/azure/virtual-machines/dv2-dsv2-series) | `standardDSv2Family` | Standard_DS3_v2, Standard_DS4_v2 |
| [Dsv3-series](https://learn.microsoft.com/en-us/azure/virtual-machines/dv3-dsv3-series) | `standardDSv3Family` | Standard_D2s_v3, Standard_D4s_v3, Standard_D8s_v3 |
| [Dsv4-series](https://learn.microsoft.com/en-us/azure/virtual-machines/dv4-dsv4-series) | `standardDSv4Family` | Standard_D2s_v4, Standard_D4s_v4, Standard_D8s_v4 |
| [Dsv5-series](https://learn.microsoft.com/en-us/azure/virtual-machines/dv5-dsv5-series) | `standardDSv5Family` | Standard_D2s_v5, Standard_D4s_v5, Standard_D8s_v5 |
| [Dsv6-series](https://learn.microsoft.com/en-us/azure/virtual-machines/dsv6-series) | `StandardDsv6Family` | Standard_D2s_v6, Standard_D4s_v6, Standard_D8s_v6 |
| ... | ... | ... |

### Compute Optimized

High CPU-to-memory ratio. Good for medium traffic web servers, network appliances, batch processes, and application servers.

| Azure VM Series | Family Name | Example VM Size Name |
|-----------------|-------------|----------------------|
| [FS-series](https://learn.microsoft.com/en-us/azure/virtual-machines/f-series) | `standardFSFamily` | Standard_F4s, Standard_F8s |
| [Fsv2-series](https://learn.microsoft.com/en-us/azure/virtual-machines/fsv2-series) | `standardFSv2Family` | Standard_F4s_v2, Standard_F8s_v2 |
| ... | ... | ... |

### Memory Optimized

High memory-to-CPU ratio. Great for relational database servers, medium to large caches, and in-memory analytics.

### Storage Optimized

High disk throughput and IO. Ideal for Big Data, SQL, NoSQL databases.

### GPU Accelerated

### High Performance Compute

## Complete List by Azure Series Name

For quick reference, here is the complete list sorted by Azure VM Series name:

| Azure VM Series | Family Name | Category |
|-----------------|-------------|----------|
| [Basv2-series](link) | `standardBasv2Family` | General Purpose |
| [BS-series](link) | `standardBSFamily` | General Purpose |
| [Bsv2-series](link) | `standardBsv2Family` | General Purpose |
| [DADSv5-series](link) | `standardDADSv5Family` | General Purpose (AMD) |
| [Dadv6-series](link) | `standardDadv6Family` | General Purpose (AMD) |
| ... | ... | ... |

## Notes

- **Minimum Requirements**: Example VM sizes listed in the tables meet or exceed OpenShift's minimum requirements:
  - Control Plane: 8 vCPUs, 32GB RAM minimum
  - Worker Nodes: 2 vCPUs, 8GB RAM minimum (x86_64); 4 vCPUs, 16GB RAM minimum (ARM64)
  - See [limits.md](limits.md) for complete resource requirements
- **Regional Availability**: Not all VM families are available in all Azure regions. Verify availability in your target region using the [Azure Products by Region](https://azure.microsoft.com/en-us/global-infrastructure/services/) page.
- **Quota Limits**: Your Azure subscription has quota limits for vCPUs per family. Ensure sufficient quota before deployment. Default subscriptions allow 20 vCPUs and should be increased to at least 22 for a default cluster.
- **Premium Storage**: Series ending in 's' (e.g., DSv2, Esv3) support premium SSD storage, which is recommended for production deployments.

## References

- [Azure VM sizes overview](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview)
- [Azure VM size families by type](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview#list-of-vm-size-families-by-type)
- [OpenShift Azure Installation Customization](customization.md)
```

### Phase 3: Implementation Steps

1. **Create Enhanced x86_64 Documentation**:
   - Add header and overview section
   - Create mapping table for major families (20-30 most common)
   - Organize all families by category
   - Add notes section
   - Add references

2. **Create Enhanced aarch64 Documentation**:
   - Apply same structure
   - Focus on ARM64-specific families
   - Link to ARM64 Azure documentation

3. **Update customization.md** (if needed):
   - Add reference to tested instance types
   - Ensure examples use tested families

4. **Testing**:
   - Verify all links work
   - Ensure Markdown renders correctly
   - Check for spelling/grammar

### Phase 4: Review and Refinement

1. **Technical Review**:
   - Verify accuracy of Azure Series mappings
   - Ensure all tested families are included
   - Validate categorization against Azure's official docs

2. **User Experience Review**:
   - Ensure documentation is clear and easy to navigate
   - Verify examples are helpful
   - Check that common questions are answered

---

## Acceptance Criteria

- [ ] Documentation references Azure VM Series families, not just SKU enumerations
- [ ] Clear linkage between OpenShift family names and Azure Series names
- [ ] Header explains what families are and how to use them
- [ ] Organized by Azure's standard categories (General Purpose, Compute, Memory, etc.)
- [ ] Includes mapping table with OpenShift name, Azure Series, example SKUs
- [ ] Links to Azure's official VM size family documentation
- [ ] Notes section covers regional availability, quotas, instance selection
- [ ] Both x86_64 and aarch64 files updated with consistent structure
- [ ] All existing tested families preserved in the documentation
- [ ] Documentation renders correctly in Markdown viewers
- [ ] Clear examples showing how to use instance types in install-config.yaml

---

## Implementation Approach

### Commit Strategy

1. **Commit 1**: `docs(azure): Enhance tested instance types documentation structure`
   - Add header, overview, and mapping guidance to x86_64 file
   - Add categorization and table format for key families
   - Body: Explain this improves clarity by aligning with Azure's official VM Series documentation

2. **Commit 2**: `docs(azure): Update ARM64 tested instance types documentation`
   - Apply same structure to aarch64 file
   - Body: Maintain consistency with x86_64 documentation improvements

3. **Commit 3** (if needed): `docs(azure): Update customization examples to reference tested families`
   - Add references to tested instance types documentation
   - Body: Help users understand relationship between families and specific SKUs

### Testing

- Verify Markdown rendering
- Check all Azure documentation links
- Ensure mapping accuracy

---

## Timeline

**Estimated Effort**: 2-3 hours
- Research and mapping: 45 minutes
- Documentation writing: 60 minutes
- Review and refinement: 30 minutes
- Testing and PR creation: 15 minutes

---

## Questions for Clarification

None at this time. The JIRA provides clear requirements and the current state of the documentation is well-understood.

---

## References

- JIRA: https://issues.redhat.com/browse/CORS-4288
- Current x86_64 doc: `docs/user/azure/tested_instance_types_x86_64.md`
- Current aarch64 doc: `docs/user/azure/tested_instance_types_aarch64.md`
- Azure VM sizes: https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview
- Azure VM families by type: https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview#list-of-vm-size-families-by-type
