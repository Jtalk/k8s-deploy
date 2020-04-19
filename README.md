# Deploy manifests action for Kubernetes

This action can be used to deploy manifests to Kubernetes clusters.

This action requires that the cluster context be set in the appropriate KUBECONFIG file. 

## Action capabilities

Following are the key capabilities of this action:

- **Artifact substitution**: The deploy action takes as input a list of container images which can be specified along with their tags or digests. The same is substituted into the non-templatized version of manifest files before applying to the cluster to ensure that the right version of the image is pulled by the cluster nodes.

- **Object stability checks**: Rollout status is checked for the Kubernetes objects deployed. This is done to incorporate stability checks while computing the action status as success/failure.

- **Secret handling**: The secret names specified as inputs in the action are used to augment the input manifest files with imagePullSecrets values before deploying to the cluster.

- **Deployment strategy** Choosing canary strategy with this action leads to creation of workloads suffixed with '-baseline' and '-canary'. There are two methods of traffic splitting supported in the action:
    - **Service Mesh Interface**: Service Mesh Interface abstraction allows for plug-and-play configuration with service mesh providers such as Linkerd and Istio. Meanwhile, this action takes away the hard work of mapping SMI's TrafficSplit objects to the stable, baseline and canary services during the lifecycle of the deployment strategy. Service mesh based canary deployments using this action are more accurate as service mesh providers enable granular percentage traffic split (via service registry and sidecar containers injected into pods alongside application containers).
    - **Only Kubernetes (no service mesh)**: In the absence of service mesh, while it may not be possible to achieve exact percentage split at the request level, it is still possible to perform canary deployments by deploying -baseline and -canary workload variants next to the stable variant. The service routes requests to pods of all three workload variants as the selector-label constraints are met (KubernetesManifest will honor these when creating -baseline and -canary variants). This achieves the intended effect of routing only a portion of total requests to the canary.


## Action inputs

<table>
  <thead>
    <tr>
      <th>Action inputs</th>
      <th>Description</th>
    </tr>
  </thead>

  <tr>
    <td><code>namespace</code><br/>Namespace</td>
    <td>(Optional) Namespace within the cluster to deploy to.</td>
  </tr>
  <tr>
    <td><code>manifests</code><br/>Manifests</td>
    <td>(Required) Path to the manifest files to be used for deployment</td>
  </tr>
  <tr>
    <td><code>images</code><br/>Images</td>
    <td>(Optional) Fully qualified resource URL of the image(s) to be used for substitutions on the manifest files. This multiline input accepts specifying multiple artifact substitutions in newline separated form. For example - <br>images: |<br>&nbsp&nbsprepo/foo:test1<br>&nbsp&nbsprepo/bar:test2<br>In this example, all references to repo/foo and repo/bar are searched for in the image field of the input manifest files. For the matches found, the tags test1 and test2 are substituted.</td>
  </tr>
  <tr>
    <td><code>imagepullsecrets</code><br/>Image pull secrets</td>
    <td>(Optional) Multiline input where each line contains the name of a docker-registry secret that has already been setup within the cluster. Each of these secret names are added under imagePullSecrets field for the workloads found in the input manifest files</td>
  </tr>
  <tr>
    <td><code>strategy</code><br/>Strategy</td>
    <td>(Optional) Deployment strategy to be used while applying manifest files on the cluster. Acceptable values: none/canary. none - No deployment strategy is used when deploying. canary - Canary deployment strategy is used when deploying to the cluster</td>
  </tr>
  <tr>
    <td><code>traffic-split-method</code><br/>Traffic split method</td>
    <td>(Optional) Acceptable values: pod/smi; Default value: pod <br>SMI: Percentage traffic split is done at request level using service mesh. Service mesh has to be setup by cluster admin. Orchestration of <a href="https://github.com/deislabs/smi-spec/blob/master/traffic-split.md" data-raw-source="TrafficSplit](https://github.com/deislabs/smi-spec/blob/master/traffic-split.md)">TrafficSplit</a> objects of SMI is handled by this action. <br>Pod: Percentage split not possible at request level in the absence of service mesh. So the percentage input is used to calculate the replicas for baseline and canary as a percentage of replicas specified in the input manifests for the stable variant.</td>
  </tr>
  <tr>
    <td><code>percentage</code><br/>Percentage</td>
    <td>(Required if strategy ==  canary) Percentage used to compute the number of replicas of &#39;-baseline&#39; and &#39;-canary&#39; varaints of the workloads found in manifest files. For the specified percentage input, if (percentage * numberOfDesirerdReplicas)/100 is not a round number, the floor of this number is used while creating &#39;-baseline&#39; and &#39;-canary&#39;<br/>Example: If Deployment hello-world was found in the input manifest file with &#39;replicas: 4&#39; and if &#39;strategy: canary&#39; and &#39;percentage: 25&#39; are given as inputs to the action, then the Deployments hello-world-baseline and hello-world-canary are created with 1 replica each. The &#39;-baseline&#39; variant is created with the same image and tag as the stable version (4 replica variant prior to deployment) while the &#39;-canary&#39; variant is created with the image and tag corresponding to the new changes being deployed</td>
  </tr>
  <tr>
    <td><code>baseline-and-canary-replicas</code><br/>Baseline and canary replicas</td>
    <td>(Optional; Relevant only if trafficSplitMethod ==  smi) When trafficSplitMethod == smi, as percentage traffic split is controlled in the service mesh plane, the actual number of replicas for canary and baseline variants could be controlled independently of the traffic split. For example, assume that the input Deployment manifest desired 30 replicas to be used for stable and that the following inputs were specified for the action - <br>&nbsp;&nbsp;&nbsp;&nbsp;strategy: canary<br>&nbsp;&nbsp;&nbsp;&nbsp;trafficSplitMethod: smi<br>&nbsp;&nbsp;&nbsp;&nbsp;percentage: 20<br>&nbsp;&nbsp;&nbsp;&nbsp;baselineAndCanaryReplicas: 1<br> In this case, stable variant will receive 80% traffic while baseline and canary variants will receive 10% each (20% split equally between baseline and canary). However, instead of creating baseline and canary with 3 replicas, the explicit count of baseline and canary replicas is honored. That is, only 1 replica each is created for baseline and canary variants.</td>
  </tr>
  <tr>
    <td><code>action</code><br/>Action</td>
    <td>(Required) Default value: deploy. Acceptable values: deploy/promote/reject. Promote or reject actions are used to promote or reject canary deployments. Sample YAML snippets are provided below for guidance on how to use the same.</td>
  </tr>
  <tr>
    <td><code>kubectl-version</code><br/>Kubectl version</td>
    <td>(Optional) Version of kubectl client to be used for deploying the manifest to the cluster. If this input is left unspecified, latest version is used.</td>
  </tr>
</table>

## Examples YAML snippets

### Basic deployment (without any deployment strategy)

```yaml
- uses: jtalk/k8s-deploy@v1
  with:
    namespace: 'myapp'
    manifests: |
        deployment.yaml
        service.yaml
    images: 'jtalk/myapp:${{ event.run_id }}'
    imagepullsecrets: |
      image-pull-secret1
      image-pull-secret2
    kubectl-version: 'latest'
```

### Deployment Strategies - Canary deployment without service mesh

```yaml
- uses: jtalk/k8s-deploy@v1
  with:
    namespace: 'myapp'
    images: 'jtalk/myapp:${{ event.run_id }}'
    imagepullsecrets: |
      image-pull-secret1
      image-pull-secret2
    manifests: |
        deployment.yaml
        service.yaml
    strategy: canary
    percentage: 20
```

To promote/reject the canary created by the above snippet, the following YAML snippet could be used:

```yaml
- uses: jtalk/k8s-deploy@v1
  with:
    namespace: 'myapp'
    images: 'jtalk/myapp:${{ event.run_id }}'
    imagepullsecrets: |
      image-pull-secret1
      image-pull-secret2
    manifests: |
        deployment.yaml
        service.yaml
    strategy: canary
    action: promote # substitute reject if you want to reject
```

### Deployment Strategies - Canary deployment based on Service Mesh Interface

```yaml
- uses: jtalk/k8s-deploy@v1
  with:
    namespace: 'myapp'
    images: 'jtalk/myapp:${{ event.run_id }}'
    imagepullsecrets: |
      image-pull-secret1
      image-pull-secret2
    manifests: |
        deployment.yaml
        service.yaml
    strategy: canary
    traffic-split-method: smi
    percentage: 20
    baseline-and-canary-replicas: 1
```

To promote/reject the canary created by the above snippet, the following YAML snippet could be used:

```yaml
- uses: jtalk/k8s-deploy@v1
  with:
    namespace: 'myapp'
    images: 'jtalk/myapp:${{ event.run_id }} '
    imagepullsecrets: |
      image-pull-secret1
      image-pull-secret2
    manifests: |
        deployment.yaml
        service.yaml
    strategy: canary
    traffic-split-method: smi
    action: reject # substitute reject if you want to reject
```
