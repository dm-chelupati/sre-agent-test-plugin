# Cost Management Analysis

Analyze Azure resource costs and provide actionable optimization recommendations.

## Step 1: Get Advisor Cost Recommendations

Query Azure Advisor for cost recommendations scoped to the target resource groups:

```
az advisor recommendation list --category Cost --query "[?resourceGroup=="<RG>"]" -o table
```

Report each recommendation with: resource name, recommendation text, estimated annual savings.

## Step 2: Identify Idle and Underutilized Resources

Check for common cost waste patterns:

### Stopped but allocated VMs
```
az vm list -g <RG> --query "[?powerState!="VM running"].{name:name, size:hardwareProfile.vmSize, state:powerState}" -o table
```

### Unattached disks
```
az disk list -g <RG> --query "[?managedBy==null].{name:name, size:diskSizeGb, sku:sku.name}" -o table
```

### Idle public IPs
```
az network public-ip list -g <RG> --query "[?ipConfiguration==null].{name:name, ip:ipAddress}" -o table
```

### Underutilized App Service Plans / Container App environments
```
az containerapp list -g <RG> --query "[].{name:name, activeRevisions:properties.latestRevisionName}" -o table
```

### PostgreSQL Flexible Servers — check if stopped or oversized
```
az postgres flexible-server list -g <RG> --query "[].{name:name, state:state, sku:sku.name, tier:sku.tier, storage:storage.storageSizeGb}" -o table
```

## Step 3: Cost Breakdown (if Cost Management API is accessible)

Try to query recent costs:

```
az costmanagement query --type ActualCost --scope /subscriptions/<sub>/resourceGroups/<RG> --timeframe MonthToDate --dataset-grouping name=ResourceType type=Dimension --query "[].rows" -o table
```

If this fails (permissions may vary), skip and note that Cost Management Reader role is needed.

## Step 4: Right-Sizing Recommendations

For each resource, compare current SKU against usage:
- Container Apps: check replica count vs actual CPU/memory usage
- PostgreSQL: check SKU tier vs connection count and storage used
- App Service Plans: check tier vs traffic volume

## Step 5: Generate Report

Summarize findings in a table:

| Resource | Type | Issue | Recommendation | Est. Monthly Savings |
|---|---|---|---|---|
| (resource name) | (type) | (idle/oversized/etc) | (action) | ($ if known) |

Include:
- Total identified savings
- Quick wins (can act now)
- Items needing further analysis

## Rules

- Never delete resources without explicit user approval
- Never resize or stop production resources without approval
- Cost data may be delayed 24-48 hours
- Advisor recommendations refresh every 24 hours
