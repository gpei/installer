# Test Plan: OCPBUGS-56846

## JIRA Summary

**Issue Key**: OCPBUGS-56846
**Summary**: Azure: missing check for existence of user-assigned identity when installing cluster with identity type: UserAssigned
**Component**: Installer / openshift-installer
**Status**: POST
**Priority**: Undefined

### Description

When users specify a user-assigned identity in the install-config that does not exist in Azure, the installer proceeds with cluster creation and only fails during machine provisioning with a `FailedIdentityOperation` error. This wastes time and creates orphaned resources in Azure.

### Expected Behavior

The installer should validate that user-assigned identities exist before beginning cluster creation and provide clear error messages if they are missing or misconfigured.

---

## PR Summary

**Primary PR**: https://github.com/openshift/installer/pull/10220 (WIP - Not yet merged)
**Title**: WIP: OCPBUGS-56846: validate azure user-assigned identity existence

### What the PR Does

The PR adds validation to check if user-assigned identities exist in Azure before starting cluster installation:

1. **New API Method**: `GetUserAssignedIdentity()` - Calls Azure MSI API to verify identity exists
2. **New Validation Function**: `ValidateUserAssignedIdentities()` - Validates all user-assigned identities specified in:
   - Default machine platform (applies to all machines)
   - Control plane specific configuration
   - Compute pool specific configuration
3. **Early Validation**: Runs during install-config validation phase (before any resources are created)

### Changed Files

- `pkg/asset/installconfig/azure/client.go` - Added GetUserAssignedIdentity API method
- `pkg/asset/installconfig/azure/validation.go` - Added validation logic
- `pkg/asset/installconfig/azure/validation_test.go` - Added comprehensive tests

---

## Prerequisites

### Required Access

- Azure subscription with permissions to:
  - Create and delete user-assigned identities
  - Create resource groups
  - View subscription resources
- OpenShift installer binary (with the fix applied)

### Required Tools

- `az` CLI - Azure command-line tool
- `openshift-install` - OpenShift installer (built from PR branch)
- `jq` - JSON processor for parsing output

### Environment Setup

1. **Azure Authentication**:
   ```bash
   az login
   az account set --subscription <your-subscription-id>
   ```

2. **Create Test Resource Group**:
   ```bash
   export TEST_RG="test-identity-validation-rg"
   export LOCATION="eastus"
   az group create --name $TEST_RG --location $LOCATION
   ```

3. **Build Installer from PR**:
   ```bash
   git clone https://github.com/openshift/installer.git
   cd installer
   gh pr checkout 10220
   hack/build.sh
   export INSTALLER_BIN=$(pwd)/bin/openshift-install
   ```

---

## Test Scenarios

### Scenario 1: Non-Existent User-Assigned Identity in defaultMachinePlatform

**Acceptance Criteria**: Installer rejects install-config with non-existent identity before creating resources

**Test Steps**:

1. **Create install-config with non-existent identity**:
   ```bash
   mkdir test-scenario-1
   cd test-scenario-1

   cat > install-config.yaml <<EOF
   apiVersion: v1
   baseDomain: example.com
   metadata:
     name: test-cluster-1
   platform:
     azure:
       region: eastus
       baseDomainResourceGroupName: os4-common
       defaultMachinePlatform:
         identity:
           type: UserAssigned
           userAssignedIdentities:
           - name: non-existent-identity
             resourceGroup: $TEST_RG
             subscription: $(az account show --query id -o tsv)
   pullSecret: '<your-pull-secret>'
   sshKey: '<your-ssh-key>'
   EOF
   ```

2. **Attempt to create manifests**:
   ```bash
   $INSTALLER_BIN create manifests --dir .
   ```

3. **Expected Result**:
   - Command should **fail** with validation error
   - Error message should contain:
     - `user-assigned identity 'non-existent-identity' was not found in resource group '<TEST_RG>'`
     - Field path: `platform.azure.defaultMachinePlatform.identity.userAssignedIdentities[0]`
   - No Azure resources should be created
   - No manifest files should be generated

4. **Verification**:
   ```bash
   # Verify no resources created
   az resource list --resource-group $TEST_RG --output table
   # Should show: No resources

   # Verify no manifests created
   ls manifests/
   # Should show: directory not found or empty
   ```

**Expected Error Format**:
```
FATAL failed to fetch Install Config: failed to create install config: [platform.azure.defaultMachinePlatform.identity.userAssignedIdentities[0]: Invalid value: "non-existent-identity": user-assigned identity 'non-existent-identity' was not found in resource group 'test-identity-validation-rg']
```

---

### Scenario 2: Non-Existent Identity in Control Plane Configuration

**Acceptance Criteria**: Installer validates control plane specific identity configuration

**Test Steps**:

1. **Create install-config with invalid control plane identity**:
   ```bash
   mkdir test-scenario-2
   cd test-scenario-2

   cat > install-config.yaml <<EOF
   apiVersion: v1
   baseDomain: example.com
   metadata:
     name: test-cluster-2
   controlPlane:
     name: master
     platform:
       azure:
         type: Standard_D4s_v3
         identity:
           type: UserAssigned
           userAssignedIdentities:
           - name: fake-master-identity
             resourceGroup: $TEST_RG
             subscription: $(az account show --query id -o tsv)
   platform:
     azure:
       region: eastus
       baseDomainResourceGroupName: os4-common
   pullSecret: '<your-pull-secret>'
   sshKey: '<your-ssh-key>'
   EOF
   ```

2. **Attempt to create manifests**:
   ```bash
   $INSTALLER_BIN create manifests --dir .
   ```

3. **Expected Result**:
   - Command should **fail** with validation error
   - Error message should reference: `controlPlane.platform.azure.identity.userAssignedIdentities[0]`
   - Error should mention `fake-master-identity` not found
   - No resources created

4. **Verification**:
   ```bash
   az resource list --resource-group $TEST_RG --output table
   # Should show: No resources
   ```

---

### Scenario 3: Non-Existent Identity in Compute Pool Configuration

**Acceptance Criteria**: Installer validates compute pool specific identity configuration

**Test Steps**:

1. **Create install-config with invalid worker identity**:
   ```bash
   mkdir test-scenario-3
   cd test-scenario-3

   cat > install-config.yaml <<EOF
   apiVersion: v1
   baseDomain: example.com
   metadata:
     name: test-cluster-3
   compute:
   - name: worker
     replicas: 3
     platform:
       azure:
         type: Standard_D2s_v3
         identity:
           type: UserAssigned
           userAssignedIdentities:
           - name: invalid-worker-identity
             resourceGroup: $TEST_RG
             subscription: $(az account show --query id -o tsv)
   platform:
     azure:
       region: eastus
       baseDomainResourceGroupName: os4-common
   pullSecret: '<your-pull-secret>'
   sshKey: '<your-ssh-key>'
   EOF
   ```

2. **Attempt to create manifests**:
   ```bash
   $INSTALLER_BIN create manifests --dir .
   ```

3. **Expected Result**:
   - Command should **fail** with validation error
   - Error message should reference: `compute[0].platform.azure.identity.userAssignedIdentities[0]`
   - Error should mention `invalid-worker-identity` not found
   - No resources created

---

### Scenario 4: Valid User-Assigned Identity (Happy Path)

**Acceptance Criteria**: Installer accepts install-config with existing, valid user-assigned identity

**Test Steps**:

1. **Create a valid user-assigned identity**:
   ```bash
   VALID_IDENTITY_NAME="test-valid-identity-$(date +%s)"
   az identity create \
     --name $VALID_IDENTITY_NAME \
     --resource-group $TEST_RG \
     --location $LOCATION

   # Verify identity created
   az identity show \
     --name $VALID_IDENTITY_NAME \
     --resource-group $TEST_RG
   ```

2. **Create install-config with valid identity**:
   ```bash
   mkdir test-scenario-4
   cd test-scenario-4

   cat > install-config.yaml <<EOF
   apiVersion: v1
   baseDomain: example.com
   metadata:
     name: test-cluster-4
   platform:
     azure:
       region: eastus
       baseDomainResourceGroupName: os4-common
       defaultMachinePlatform:
         identity:
           type: UserAssigned
           userAssignedIdentities:
           - name: $VALID_IDENTITY_NAME
             resourceGroup: $TEST_RG
             subscription: $(az account show --query id -o tsv)
   pullSecret: '<your-pull-secret>'
   sshKey: '<your-ssh-key>'
   EOF
   ```

3. **Attempt to create manifests**:
   ```bash
   $INSTALLER_BIN create manifests --dir .
   ```

4. **Expected Result**:
   - Command should **succeed** without validation errors
   - Manifest files should be created in `manifests/` directory
   - No error messages about missing identity

5. **Verification**:
   ```bash
   # Verify manifests created
   ls manifests/
   # Should show: Various manifest YAML files

   # Check for identity references in manifests
   grep -r $VALID_IDENTITY_NAME manifests/
   # Should show: Identity name referenced in machine manifests
   ```

---

### Scenario 5: Multiple Valid Identities Across Different Pools

**Acceptance Criteria**: Installer validates all identities when specified in multiple machine pools

**Test Steps**:

1. **Create multiple valid identities**:
   ```bash
   DEFAULT_IDENTITY="test-default-identity-$(date +%s)"
   CONTROL_IDENTITY="test-control-identity-$(date +%s)"
   WORKER_IDENTITY="test-worker-identity-$(date +%s)"

   az identity create --name $DEFAULT_IDENTITY --resource-group $TEST_RG --location $LOCATION
   az identity create --name $CONTROL_IDENTITY --resource-group $TEST_RG --location $LOCATION
   az identity create --name $WORKER_IDENTITY --resource-group $TEST_RG --location $LOCATION
   ```

2. **Create install-config with multiple identities**:
   ```bash
   mkdir test-scenario-5
   cd test-scenario-5

   cat > install-config.yaml <<EOF
   apiVersion: v1
   baseDomain: example.com
   metadata:
     name: test-cluster-5
   controlPlane:
     name: master
     platform:
       azure:
         identity:
           type: UserAssigned
           userAssignedIdentities:
           - name: $CONTROL_IDENTITY
             resourceGroup: $TEST_RG
             subscription: $(az account show --query id -o tsv)
   compute:
   - name: worker
     replicas: 3
     platform:
       azure:
         identity:
           type: UserAssigned
           userAssignedIdentities:
           - name: $WORKER_IDENTITY
             resourceGroup: $TEST_RG
             subscription: $(az account show --query id -o tsv)
   platform:
     azure:
       region: eastus
       baseDomainResourceGroupName: os4-common
       defaultMachinePlatform:
         identity:
           type: UserAssigned
           userAssignedIdentities:
           - name: $DEFAULT_IDENTITY
             resourceGroup: $TEST_RG
             subscription: $(az account show --query id -o tsv)
   pullSecret: '<your-pull-secret>'
   sshKey: '<your-ssh-key>'
   EOF
   ```

3. **Attempt to create manifests**:
   ```bash
   $INSTALLER_BIN create manifests --dir .
   ```

4. **Expected Result**:
   - Command should **succeed**
   - All three identities should be validated
   - Manifests should reference appropriate identities for each pool

5. **Verification**:
   ```bash
   # Check control plane manifests
   grep -r $CONTROL_IDENTITY manifests/
   # Should show: Control plane machine references

   # Check worker manifests
   grep -r $WORKER_IDENTITY manifests/
   # Should show: Worker machine references
   ```

---

### Scenario 6: Mix of Valid and Invalid Identities

**Acceptance Criteria**: Installer reports ALL invalid identities in a single validation pass

**Test Steps**:

1. **Create one valid identity**:
   ```bash
   VALID_ID="test-valid-mix-$(date +%s)"
   az identity create --name $VALID_ID --resource-group $TEST_RG --location $LOCATION
   ```

2. **Create install-config with mix of valid and invalid**:
   ```bash
   mkdir test-scenario-6
   cd test-scenario-6

   cat > install-config.yaml <<EOF
   apiVersion: v1
   baseDomain: example.com
   metadata:
     name: test-cluster-6
   controlPlane:
     name: master
     platform:
       azure:
         identity:
           type: UserAssigned
           userAssignedIdentities:
           - name: $VALID_ID
             resourceGroup: $TEST_RG
             subscription: $(az account show --query id -o tsv)
   compute:
   - name: worker
     replicas: 3
     platform:
       azure:
         identity:
           type: UserAssigned
           userAssignedIdentities:
           - name: invalid-worker-identity-mix
             resourceGroup: $TEST_RG
             subscription: $(az account show --query id -o tsv)
   platform:
     azure:
       region: eastus
       baseDomainResourceGroupName: os4-common
   pullSecret: '<your-pull-secret>'
   sshKey: '<your-ssh-key>'
   EOF
   ```

3. **Attempt to create manifests**:
   ```bash
   $INSTALLER_BIN create manifests --dir . 2>&1 | tee validation-output.log
   ```

4. **Expected Result**:
   - Command should **fail** with validation error
   - Error should mention `invalid-worker-identity-mix` not found
   - Error should NOT complain about the valid control plane identity
   - No resources created

5. **Verification**:
   ```bash
   # Check error mentions worker identity
   grep "invalid-worker-identity-mix" validation-output.log
   # Should show: Error message

   # Verify does NOT mention valid identity
   grep "$VALID_ID.*not found" validation-output.log
   # Should show: No matches
   ```

---

### Scenario 7: Identity in Different Subscription

**Acceptance Criteria**: Installer validates identities across different subscriptions

**Test Steps**:

1. **Prerequisites**: Requires access to multiple Azure subscriptions

2. **Create identity in different subscription** (if available):
   ```bash
   # Switch to different subscription
   SECOND_SUBSCRIPTION="<second-subscription-id>"
   az account set --subscription $SECOND_SUBSCRIPTION

   # Create identity
   CROSS_SUB_IDENTITY="test-cross-sub-$(date +%s)"
   az identity create --name $CROSS_SUB_IDENTITY --resource-group $TEST_RG --location $LOCATION

   # Switch back to original subscription
   az account set --subscription $(az account list --query "[?isDefault].id" -o tsv)
   ```

3. **Create install-config referencing cross-subscription identity**:
   ```bash
   mkdir test-scenario-7
   cd test-scenario-7

   cat > install-config.yaml <<EOF
   apiVersion: v1
   baseDomain: example.com
   metadata:
     name: test-cluster-7
   platform:
     azure:
       region: eastus
       baseDomainResourceGroupName: os4-common
       defaultMachinePlatform:
         identity:
           type: UserAssigned
           userAssignedIdentities:
           - name: $CROSS_SUB_IDENTITY
             resourceGroup: $TEST_RG
             subscription: $SECOND_SUBSCRIPTION
   pullSecret: '<your-pull-secret>'
   sshKey: '<your-ssh-key>'
   EOF
   ```

4. **Attempt to create manifests**:
   ```bash
   $INSTALLER_BIN create manifests --dir .
   ```

5. **Expected Result**:
   - Command should **succeed** if identity exists in specified subscription
   - Validation should check the correct subscription (not default)

**Note**: Skip this test if only one subscription is available

---

### Scenario 8: Wrong Resource Group Specified

**Acceptance Criteria**: Installer detects when identity exists but in different resource group

**Test Steps**:

1. **Create identity in different resource group**:
   ```bash
   SECOND_RG="test-identity-second-rg"
   az group create --name $SECOND_RG --location $LOCATION

   IDENTITY_IN_SECOND_RG="test-identity-wrong-rg-$(date +%s)"
   az identity create --name $IDENTITY_IN_SECOND_RG --resource-group $SECOND_RG --location $LOCATION
   ```

2. **Create install-config with wrong resource group**:
   ```bash
   mkdir test-scenario-8
   cd test-scenario-8

   cat > install-config.yaml <<EOF
   apiVersion: v1
   baseDomain: example.com
   metadata:
     name: test-cluster-8
   platform:
     azure:
       region: eastus
       baseDomainResourceGroupName: os4-common
       defaultMachinePlatform:
         identity:
           type: UserAssigned
           userAssignedIdentities:
           - name: $IDENTITY_IN_SECOND_RG
             resourceGroup: $TEST_RG  # Wrong RG - should be $SECOND_RG
             subscription: $(az account show --query id -o tsv)
   pullSecret: '<your-pull-secret>'
   sshKey: '<your-ssh-key>'
   EOF
   ```

3. **Attempt to create manifests**:
   ```bash
   $INSTALLER_BIN create manifests --dir .
   ```

4. **Expected Result**:
   - Command should **fail** with validation error
   - Error should state identity not found in specified resource group
   - Even though identity exists, it's in the wrong RG

---

## Regression Testing

### Areas to Verify

1. **Existing Identity Types Still Work**:
   - Test with `identity.type: SystemAssigned` (should not trigger validation)
   - Test without identity configuration (should use defaults)

2. **Non-Azure Platforms Unaffected**:
   - Verify AWS install-configs still work
   - Verify GCP install-configs still work

3. **Azure Stack Hub**:
   - Test that validation works on Azure Stack Hub environments
   - Verify API version compatibility (uses appropriate SDK)

---

## Success Criteria

✅ **All scenarios must pass**:

- [ ] Scenario 1: Non-existent identity in defaultMachinePlatform rejected
- [ ] Scenario 2: Non-existent identity in controlPlane rejected
- [ ] Scenario 3: Non-existent identity in compute pool rejected
- [ ] Scenario 4: Valid identity accepted (happy path)
- [ ] Scenario 5: Multiple valid identities across pools accepted
- [ ] Scenario 6: Mix of valid/invalid identities reports errors correctly
- [ ] Scenario 7: Cross-subscription identity validated (if applicable)
- [ ] Scenario 8: Wrong resource group detected

✅ **Validation occurs early**:
- [ ] Errors appear during `create manifests` phase
- [ ] No Azure resources created when validation fails
- [ ] Clear error messages with field paths

✅ **Regression tests pass**:
- [ ] SystemAssigned identity type still works
- [ ] Other cloud platforms unaffected
- [ ] Azure Stack Hub compatibility verified

---

## Troubleshooting

### Issue: "failed to create user-assigned identities client"

**Possible Causes**:
- Azure credentials not configured
- Invalid subscription ID
- Network connectivity issues

**Debug Steps**:
```bash
# Verify Azure login
az account show

# Test identity access directly
az identity list --resource-group $TEST_RG

# Check installer logs
cat .openshift_install.log | grep -i "identity"
```

### Issue: Validation passes but identity doesn't actually exist

**Possible Causes**:
- Caching issues
- Wrong Azure environment (Gov Cloud vs Public)

**Debug Steps**:
```bash
# Force fresh identity check
az identity show --name <identity-name> --resource-group <rg> --output json

# Verify subscription and cloud environment
az account show --query "{subscription:id, cloud:name}"
```

### Issue: Cross-subscription test fails

**Possible Causes**:
- Insufficient permissions in second subscription
- Service principal doesn't have access

**Debug Steps**:
```bash
# Check permissions in second subscription
az account set --subscription $SECOND_SUBSCRIPTION
az identity list --resource-group $TEST_RG

# Verify service principal access
az role assignment list --assignee <sp-id> --scope /subscriptions/$SECOND_SUBSCRIPTION
```

---

## Cleanup

After testing, clean up resources:

```bash
# Delete test resource groups
az group delete --name $TEST_RG --yes --no-wait
az group delete --name $SECOND_RG --yes --no-wait

# Remove test directories
cd ..
rm -rf test-scenario-*
```

---

## Notes

### Critical Test Cases

🔥 **Must Pass**:
- Scenario 1 (defaultMachinePlatform validation)
- Scenario 4 (happy path with valid identity)
- Scenario 6 (reports all errors in single pass)

### Known Limitations

- **PR Status**: PR #10220 is currently in WIP status and not merged
- **API Version**: Uses Azure MSI SDK which may have version dependencies
- **Timeout**: Identity validation has 30-second timeout per identity

### Related Links

- **JIRA**: https://issues.redhat.com/browse/OCPBUGS-56846
- **PR**: https://github.com/openshift/installer/pull/10220
- **Azure MSI Docs**: https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/

---

🤖 Generated with [Claude Code](https://claude.com/claude-code) via `/jira:generate-test-plan OCPBUGS-56846`
