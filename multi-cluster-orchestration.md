# Best Practices for OpenKruise Multi-Cluster Orchestration (Karmada/OCM)

**Last Updated:** June 23, 2025

## Table of Contents
- [Introduction](#introduction)
- [Project Scope and Terms](#project-scope-and-terms)
- [OpenKruise Workloads Overview](#openkruise-workloads-overview)
- [Karmada Integration](#karmada-integration)
  - [Karmada Installation and Setup](#karmada-installation-and-setup)
    - [Prerequisites](#prerequisites)
    - [Installing Karmada](#installing-karmada)
    - [Installing OpenKruise](#installing-openkruise)
    - [Registering Clusters](#registering-clusters)
  - [Resource Interpreter Framework](#resource-interpreter-framework)
  - [CloneSet Integration with Karmada](#cloneset-integration-with-karmada)
  - [Advanced StatefulSet Integration with Karmada](#advanced-statefulset-integration-with-karmada)
  - [SidecarSet Integration with Karmada](#sidecarset-integration-with-karmada)
  - [UnitedDeployment Integration with Karmada](#uniteddeployment-integration-with-karmada)
- [Open Cluster Management (OCM) Integration](#open-cluster-management-ocm-integration)
  - [Architecture Overview for OCM](#architecture-overview-for-ocm)
  - [Installing and Configuring OCM](#installing-and-configuring-ocm)
  - [ManifestWork for OpenKruise Workloads](#manifestwork-for-openkruise-workloads)
  - [Best Practices for OCM Integration](#best-practices-for-ocm-integration)
- [Troubleshooting Guide](#troubleshooting-guide)
- [Common Challenges and Solutions](#common-challenges-and-solutions)
- [FAQ](#faq)
- [References](#references)

---

## Introduction

[OpenKruise](https://openkruise.io) is a Kubernetes extension that provides advanced workload management, such as CloneSet, Advanced StatefulSet, SidecarSet, and UnitedDeployment. For multi-cluster scenarios, [Karmada](https://karmada.io) and [Open Cluster Management (OCM)](https://open-cluster-management.io) are widely used orchestration systems. This document provides best practices and official guidelines for integrating OpenKruise workloads with Karmada and OCM.

## Project Scope and Terms

- **Project**: CNCF - OpenKruise: Best Practice for Karmada/OCM Integration (2025 Term 2)
- **Term 2**: Jun - Aug 2025
- **Goal**: Provide official guidelines for using OpenKruise workloads in Karmada and OCM, including supported features, limitations, and interpreter scripts.

## OpenKruise Workloads Overview

- **CloneSet**: Enhanced Deployment with in-place update, partitioned rolling update, and advanced scaling strategies.
- **Advanced StatefulSet**: StatefulSet with extra features (e.g., in-place update, pod management policies, persistent volume management).
- **SidecarSet**: Sidecar container management for multiple pods, enabling dynamic injection and update of sidecars.
- **UnitedDeployment**: Multi-domain workload management, allowing deployment of workloads across multiple zones or clusters with fine-grained control.

## Karmada Integration

### Karmada Installation and Setup

#### Prerequisites

* At least one Karmada control plane cluster
* Two or more Kubernetes clusters as Karmada members
* OpenKruise installed on each member cluster
* `kubectl` access to all clusters
* Familiarity with YAML and basic Lua scripting

#### Installing Karmada

```bash
# Clone and install Karmada
git clone https://github.com/karmada-io/karmada
cd karmada
hack/local-up-karmada.sh
export KUBECONFIG="$HOME/.kube/karmada.config"
```

#### Installing OpenKruise

> **Note:** Always check for the latest stable version of OpenKruise at [OpenKruise Releases](https://openkruise.io/docs/release/). Update the version in the command below as needed to match your environment.

```bash
# Replace v1.8.0 with the latest stable version if available
kubectl apply -f https://openkruise.io/manifests/latest/kruise.yaml
```

#### Registering Clusters

```bash
# Register member clusters
kubectl karmada join <CLUSTER_NAME> --cluster-kubeconfig=<MEMBER_CLUSTER_KUBECONFIG>

# Verify registration
kubectl get clusters --kubeconfig=$HOME/.kube/karmada.config
```

### Resource Interpreter Framework

- **Built-in Interpreter**: Handles standard Kubernetes resources.
- **Customized Interpreter**: For CRDs (like OpenKruise), users can define custom logic using Lua scripts via the `ResourceInterpreterCustomization` CRD.
- **Modes**:
  - **Declarative**: Use JSONPath for simple field mapping.
  - **Lua Script**: For advanced logic, write Lua scripts for operations like replica calculation, status aggregation, retention, and health checks.

**Reference:** [Karmada Resource Interpreter Guide](https://karmada.io/docs/userguide/globalview/customizing-resource-interpreter/)

### CloneSet Integration with Karmada

To enable Karmada to manage OpenKruise CloneSet resources, you must define a `ResourceInterpreterCustomization` for CloneSet. This can be done using Lua scripts for advanced behaviors.

#### Example: ResourceInterpreterCustomization for CloneSet

```yaml
apiVersion: config.karmada.io/v1alpha1
kind: ResourceInterpreterCustomization
metadata:
  name: openkruise-cloneset-interpreter
spec:
  target:
    apiVersion: apps.kruise.io/v1alpha1
    kind: CloneSet
  customizations:
    replicas:
      luaScript: |
        local kube = require("kube")
        function GetReplicas(obj)
          local replica = obj.spec.replicas
          local requirement = kube.accuratePodRequirements(obj.spec.template)
          return replica, requirement
        end
    reviseReplica:
      luaScript: |
        function ReviseReplica(obj, desiredReplica)
          obj.spec.replicas = desiredReplica
          return obj
        end
    statusAggregation:
      luaScript: |
        function AggregateStatus(desiredObj, statusItems)
          local replicas = 0
          local updatedReplicas = 0
          local readyReplicas = 0
          local availableReplicas = 0
          local updatedReadyReplicas = 0
          local expectedUpdatedReplicas = 0
          for i = 1, #statusItems do
            local status = statusItems[i].status or {}
            replicas = replicas + (status.replicas or 0)
            updatedReplicas = updatedReplicas + (status.updatedReplicas or 0)
            readyReplicas = readyReplicas + (status.readyReplicas or 0)
            availableReplicas = availableReplicas + (status.availableReplicas or 0)
            updatedReadyReplicas = updatedReadyReplicas + (status.updatedReadyReplicas or 0)
            expectedUpdatedReplicas = expectedUpdatedReplicas + (status.expectedUpdatedReplicas or 0)
          end
          return {
            replicas = replicas,
            updatedReplicas = updatedReplicas,
            readyReplicas = readyReplicas,
            availableReplicas = availableReplicas,
            updatedReadyReplicas = updatedReadyReplicas,
            expectedUpdatedReplicas = expectedUpdatedReplicas
          }
        end
    retention:
      luaScript: |
        function Retain(desiredObj, observedObj)
          if observedObj.spec.suspend ~= nil then
            desiredObj.spec.suspend = observedObj.spec.suspend
          end
          if observedObj.status ~= nil then
            desiredObj.status = observedObj.status
          end
          return desiredObj
        end
    healthInterpretation:
      luaScript: |
        function InterpretHealth(observedObj)
          if observedObj.status == nil then
            return false
          end
          -- Check if we have the expected number of ready replicas
          local specReplicas = (observedObj.spec and observedObj.spec.replicas) or 0
          local readyReplicas = observedObj.status.readyReplicas or 0
          local availableReplicas = observedObj.status.availableReplicas or 0
          
          -- Healthy if all replicas are ready and available
          return readyReplicas == specReplicas and availableReplicas == specReplicas and specReplicas > 0
        end
```

**Key Points:**
- Use Lua scripts for advanced field handling and status aggregation.
- Adjust field names as needed for other OpenKruise workloads.
- Test interpreter scripts in a staging environment before production.

### Advanced StatefulSet Integration with Karmada

Advanced StatefulSet provides more features than the native StatefulSet. You should create a similar `ResourceInterpreterCustomization` for Advanced StatefulSet, adjusting field names as needed.

> **Note on Naming:** OpenKruise's Advanced StatefulSet is identified by `apiVersion: apps.kruise.io/v1beta1` and `kind: StatefulSet`. This distinguishes it from the native Kubernetes StatefulSet (`apiVersion: apps/v1`, `kind: StatefulSet`) and is important to specify correctly in policies and interpreters.

#### Example: ResourceInterpreterCustomization for Advanced StatefulSet

```yaml
apiVersion: config.karmada.io/v1alpha1
kind: ResourceInterpreterCustomization
metadata:
  name: openkruise-advancedstatefulset-interpreter
spec:
  target:
    apiVersion: apps.kruise.io/v1beta1
    kind: StatefulSet
  customizations:
    replicas:
      luaScript: |
        function GetReplicas(obj)
          return obj.spec.replicas
        end
    reviseReplica:
      luaScript: |
        function ReviseReplica(obj, desiredReplica)
          obj.spec.replicas = desiredReplica
          return obj
        end
    statusAggregation:
      luaScript: |
        function AggregateStatus(desiredObj, statusItems)
          local replicas = 0
          local readyReplicas = 0
          local currentReplicas = 0
          local updatedReplicas = 0
          for i = 1, #statusItems do
            local status = statusItems[i].status or {}
            replicas = replicas + (status.replicas or 0)
            readyReplicas = readyReplicas + (status.readyReplicas or 0)
            currentReplicas = currentReplicas + (status.currentReplicas or 0)
            updatedReplicas = updatedReplicas + (status.updatedReplicas or 0)
          end
          return {
            replicas = replicas,
            readyReplicas = readyReplicas,
            currentReplicas = currentReplicas,
            updatedReplicas = updatedReplicas
          }
        end
    retention:
      luaScript: |
        function Retain(desiredObj, observedObj)
          if observedObj.spec.podManagementPolicy ~= nil then
            desiredObj.spec.podManagementPolicy = observedObj.spec.podManagementPolicy
          end
          return desiredObj
        end
    healthInterpretation:
      luaScript: |
        function InterpretHealth(observedObj)
          if observedObj.status == nil or observedObj.status.readyReplicas == nil or observedObj.status.replicas == nil then
            return false
          end
          return observedObj.status.readyReplicas == observedObj.status.replicas
        end
```

### SidecarSet Integration with Karmada

SidecarSet is used to manage sidecar containers across pods. You can define a custom interpreter for SidecarSet as follows:

#### Example: ResourceInterpreterCustomization for SidecarSet

```yaml
apiVersion: config.karmada.io/v1alpha1
kind: ResourceInterpreterCustomization
metadata:
  name: openkruise-sidecarset-interpreter
spec:
  target:
    apiVersion: apps.kruise.io/v1alpha1
    kind: SidecarSet
  customizations:
    replicas:
      luaScript: |
        function GetReplicas(obj)
          -- SidecarSet doesn't manage replicas directly, return 0
          return 0
        end
    statusAggregation:
      luaScript: |
        function AggregateStatus(desiredObj, statusItems)
          local matchedPods = 0
          local updatedPods = 0
          for i = 1, #statusItems do
            local status = statusItems[i].status or {}
            matchedPods = matchedPods + (status.matchedPods or 0)
            updatedPods = updatedPods + (status.updatedPods or 0)
          end
          return {
            matchedPods = matchedPods,
            updatedPods = updatedPods
          }
        end
    retention:
      luaScript: |
        function Retain(desiredObj, observedObj)
          -- Preserve injectionStrategy if it exists
          if observedObj.spec and observedObj.spec.injectionStrategy then
            if not desiredObj.spec then
              desiredObj.spec = {}
            end
            desiredObj.spec.injectionStrategy = observedObj.spec.injectionStrategy
          end
          -- Preserve status
          if observedObj.status ~= nil then
            desiredObj.status = observedObj.status
          end
          return desiredObj
        end
    healthInterpretation:
      luaScript: |
        function InterpretHealth(observedObj)
          if observedObj.status == nil or observedObj.status.updatedPods == nil or observedObj.status.matchedPods == nil then
            return false
          end
          -- A SidecarSet is healthy if all matched pods have been updated with the sidecar.
          return observedObj.status.updatedPods == observedObj.status.matchedPods
        end
```

### UnitedDeployment Integration with Karmada

UnitedDeployment manages multiple workloads across different domains (e.g., zones, clusters). Its interpreter should aggregate status and handle domain-specific logic.

> **Strategy Note:** `UnitedDeployment` provides its own native topology management for distributing workloads across different pools (e.g., zones or clusters). This is useful for managing domain-specific configurations from a single resource. Alternatively, you can use a simpler workload (like a `CloneSet`) and rely on Karmada's `PropagationPolicy` and `OverridePolicy` for multi-cluster distribution.
> 
> - **Use `UnitedDeployment` when:** You need fine-grained control over workload subsets within a single manifest, such as different resource requests or image versions per zone.
> - **Use Karmada policies when:** You prefer to manage distribution and configuration overrides at the orchestration layer, keeping the base workload manifest simpler.

#### Example: ResourceInterpreterCustomization for UnitedDeployment

```yaml
apiVersion: config.karmada.io/v1alpha1
kind: ResourceInterpreterCustomization
metadata:
  name: openkruise-uniteddeployment-interpreter
spec:
  target:
    apiVersion: apps.kruise.io/v1alpha1
    kind: UnitedDeployment
  customizations:
    statusAggregation:
      luaScript: |
        function AggregateStatus(desiredObj, statusItems)
          local replicas = 0
          local readyReplicas = 0
          for i = 1, #statusItems do
            local status = statusItems[i].status or {}
            replicas = replicas + (status.replicas or 0)
            readyReplicas = readyReplicas + (status.readyReplicas or 0)
          end
          return {
            replicas = replicas,
            readyReplicas = readyReplicas
          }
        end
    healthInterpretation:
      luaScript: |
        function InterpretHealth(observedObj)
          if observedObj.status == nil or observedObj.spec == nil or observedObj.spec.replicas == nil or observedObj.status.readyReplicas == nil then
            return false
          end
          -- A UnitedDeployment is healthy if all its replicas are ready.
          return observedObj.spec.replicas == observedObj.status.readyReplicas
        end
```

### Best Practices for Karmada Integration

1.  **CRD Synchronization**:
    -   Ensure OpenKruise CRDs are installed on all member clusters *before* propagating workloads. Use Karmada's `ClusterPropagationPolicy` to distribute the CRD definitions themselves.
    -   Keep CRD versions consistent across the fleet to avoid compatibility issues.

2.  **Interpreter Script Management**:
    -   Store your `ResourceInterpreterCustomization` scripts in version control.
    -   **Test scripts thoroughly in a staging environment before applying them to the Karmada control plane. A faulty script can impact workload orchestration.**
    -   **Cross-check all status fields referenced in your Lua scripts with the latest OpenKruise CRD specifications to ensure completeness and accuracy.**

3.  **Use Policies for Granularity**:
    -   Rely on `PropagationPolicy` to control which clusters receive a workload.
    -   Use `OverridePolicy` to apply cluster-specific modifications, such as different resource limits, image tags, or replica counts.

4.  **Status and Health Checks**:
    -   Write robust `statusAggregation` scripts to get a meaningful overview of your multi-cluster workload.
    -   Define precise `healthInterpretation` logic. A workload isn't healthy just because it exists; it's healthy when it's ready and available.
    -   **Add robust nil-checking and error handling in all Lua scripts to prevent runtime errors.**

5.  **Version Alignment**:
    -   **Ensure that OpenKruise, Karmada, and Kubernetes versions are compatible and up-to-date. Always refer to the official documentation for version compatibility matrices.**

6.  **SidecarSet Evolution**:
    -   **Note:** OpenKruise v1.7+ introduced support for native Kubernetes sidecar containers. If you are using these features, review and update your interpreter logic as needed to accommodate any changes in the SidecarSet CRD or status fields.