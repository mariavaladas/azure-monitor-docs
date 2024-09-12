---
title: Filter log collection in Container insights
description: Options for filtering Container insights data that you don't require.
ms.topic: conceptual
ms.date: 05/14/2024
ms.reviewer: aul
---

# Filter log collection in Container insights

This article describes the different filtering options available in [Container insights](./container-insights-overview.md). Kubernetes clusters generate a large amount of data that's collected by Container insights. Since you're charged for the ingestion and retention of this data, you can significantly reduce your monitoring costs by filtering out data that you don't need.

> [!IMPORTANT]
> This article describes different filtering options that require you to modify the DCR or ConfigMap for a monitored cluster. See [Configure log collection in Container insights](container-insights-data-collection-configure.md) for details on performing this configuration.

## Filter container logs

Container logs are stderr and stdout logs generated by containers in your Kubernetes cluster. These logs are stored in the [ContainerLogV2 table](./container-insights-logs-schema.md) in your Log Analytics workspace. By default all container logs are collected, but you can filter out logs from specific namespaces or disable collection of container logs entirely.

Using the [Data collection rule (DCR)](./container-insights-data-collection-configure.md#configure-data-collection-using-dcr), you can enable or disable stdout and stderr logs and filter specific namespaces from each. Settings for Container logs and namespace filtering are included in the cost presets configured in the Azure portal, and you can set these values individually using the other DCR configuration methods.


Using [ConfigMap](./container-insights-data-collection-configure.md#configure-data-collection-using-configmap), you can configure the collection of `stderr` and `stdout` logs separately for the clustery, so you can choose to enable one and not the other.

The following example shows the ConfigMap settings to collect stdout and stderr excluding the `kube-system` and `gatekeeper-system` namespaces. 

```yml
[log_collection_settings]
    [log_collection_settings.stdout]
        enabled = true
        exclude_namespaces = ["kube-system","gatekeeper-system"]

    [log_collection_settings.stderr]
        enabled = true
        exclude_namespaces = ["kube-system","gatekeeper-system"]

    [log_collection_settings.enrich_container_logs]
        enabled = true
```


## Platform log filtering (System Kubernetes namespaces)
By default, container logs from the system namespace are excluded from collection to minimize the Log Analytics cost. Container logs of system containers can be critical though in specific troubleshooting scenarios. This feature is restricted to the following system namespaces:  `kube-system`, `gatekeeper-system`, `calico-system`, `azure-arc`, `kube-public`, and `kube-node-lease`.

Enable platform logs using [ConfigMap](./container-insights-data-collection-configure.md#configure-data-collection-using-configmap) with the `collect_system_pod_logs` setting. You must also ensure that the system namespace is not in the `exclude_namespaces` setting. 

The following example shows the ConfigMap settings to collect stdout and stderr logs of `coredns` container in the `kube-system` namespace. 

```yaml
[log_collection_settings]
    [log_collection_settings.stdout]
        enabled = true
        exclude_namespaces = ["gatekeeper-system"]
        collect_system_pod_logs = ["kube-system:coredns"]

    [log_collection_settings.stderr]
        enabled = true
        exclude_namespaces = ["kube-system","gatekeeper-system"]
        collect_system_pod_logs = ["kube-system:coredns"]
```

## Annotation based filtering for workloads
Annotation-based filtering enables you to exclude log collection for certain pods and containers by annotating the pod. This can reduce your logs ingestion cost significantly and allow you to focus on relevant information without sifting through noise. 

Enable annotation based filtering using [ConfigMap](./container-insights-data-collection-configure.md#configure-data-collection-using-configmap) with the following settings. 

```yml
[log_collection_settings.filter_using_annotations]
   enabled = true
```

You must also add the required annotations on your workload pod spec. The following table highlights different possible pod annotations.

| Annotation | Description |
| ------------ | ------------- |
| `fluentbit.io/exclude: "true"` | Excludes both stdout & stderr streams on all the containers in the Pod |
| `fluentbit.io/exclude_stdout: "true"` | Excludes only stdout stream on all the containers in the Pod |
| `fluentbit.io/exclude_stderr: "true"` | Excludes only stderr stream on all the containers in the Pod |
| `fluentbit.io/exclude_container1: "true"` | Exclude both stdout & stderr streams only for the container1 in the pod |
| `fluentbit.io/exclude_stdout_container1: "true"` | Exclude only stdout only for the container1 in the pod |

>[!NOTE]
>These annotations are fluent bit based. If you use your own fluent-bit based log collection solution with the Kubernetes plugin filter and annotation based exclusion, it will stop collecting logs from both Container Insights and your solution.

Following is an example of `fluentbit.io/exclude: "true"` annotation in a Pod spec:

```
apiVersion: v1 
kind: Pod 
metadata: 
 name: apache-logs 
 labels: 
  app: apache-logs 
 annotations: 
  fluentbit.io/exclude: "true" 
spec: 
 containers: 
 - name: apache 
  image: edsiper/apache_logs 
```

## Filter environment variables
Enable collection of environment variables across all pods and nodes in the cluster using [ConfigMap](./container-insights-data-collection-configure.md#configure-data-collection-using-configmap) with the following settings. 

```yaml
[log_collection_settings.env_var]
    enabled = true
```

If collection of environment variables is globally enabled, you can disable it for a specific container by setting the environment variable `AZMON_COLLECT_ENV` to `False` either with a Dockerfile setting or in the [configuration file for the Pod](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/) under the `env:` section. If collection of environment variables is globally disabled, you can't enable collection for a specific container. The only override that can be applied at the container level is to disable collection when it's already enabled globally.



## Impact on visualizations and alerts

If you have any custom alerts or workbooks using Container insights data, then modifying your data collection settings might degrade those experiences. If you're excluding namespaces or reducing data collection frequency, review your existing alerts, dashboards, and workbooks using this data.

To scan for alerts that reference these tables, run the following Azure Resource Graph query:

```Kusto
resources
| where type in~ ('microsoft.insights/scheduledqueryrules') and ['kind'] !in~ ('LogToMetric')
| extend severity = strcat("Sev", properties["severity"])
| extend enabled = tobool(properties["enabled"])
| where enabled in~ ('true')
| where tolower(properties["targetResourceTypes"]) matches regex 'microsoft.operationalinsights/workspaces($|/.*)?' or tolower(properties["targetResourceType"]) matches regex 'microsoft.operationalinsights/workspaces($|/.*)?' or tolower(properties["scopes"]) matches regex 'providers/microsoft.operationalinsights/workspaces($|/.*)?'
| where properties contains "Perf" or properties  contains "InsightsMetrics" or properties  contains "ContainerInventory" or properties  contains "ContainerNodeInventory" or properties  contains "KubeNodeInventory" or properties  contains"KubePodInventory" or properties  contains "KubePVInventory" or properties  contains "KubeServices" or properties  contains "KubeEvents" 
| project id,name,type,properties,enabled,severity,subscriptionId
| order by tolower(name) asc
```


## Next steps

- See [Data transformations in Container insights](./container-insights-transformations.md) to add transformations to the DCR that will further filter data based on detailed criteria.
