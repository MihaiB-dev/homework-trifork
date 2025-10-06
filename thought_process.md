1. First things first, I will set up a local Kubernetes cluster using Minikube. https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download
```bash
minikube start -p trifork-homework
```

I'm using Windows with docker container.

2. The research part:
Research about helm chart. 
Some resources:
- https://helm.sh/docs/intro/quickstart/
- https://helm.sh/docs/topics/charts/

Research about tenants: 
- https://kubernetes.io/docs/concepts/security/multi-tenancy/ 
(found about namespaces, network policies, resource quotas, limit ranges, RBAC)
(think about data plane isolation - network, storage)
(found about namespace per tenant at the implementation tab)   

<details>
 <summary>About tenant for implementation</summary>
    As previously mentioned, you should consider isolating each workload in its own namespace, even if you are using dedicated clusters or virtualized control planes. This ensures that each workload only has access to its own resources, such as ConfigMaps and Secrets, and allows you to tailor dedicated security policies for each workload. In addition, it is a best practice to give each namespace names that are unique across your entire fleet (that is, even if they are in separate clusters), as this gives you the flexibility to switch between dedicated and shared clusters in the future, or to use multi-cluster tooling such as service meshes.

    Conversely, there are also advantages to assigning namespaces at the tenant level, not just the workload level, since there are often policies that apply to all workloads owned by a single tenant. However, this raises its own problems. Firstly, this makes it difficult or impossible to customize policies to individual workloads, and secondly, it may be challenging to come up with a single level of "tenancy" that should be given a namespace. For example, an organization may have divisions, teams, and subteams - which should be assigned a namespace?

    One possible approach is to organize your namespaces into hierarchies, and share certain policies and resources between them. This could include managing namespace labels, namespace lifecycles, delegated access, and shared resource quotas across related namespaces. These capabilities can be useful in both multi-team and multi-customer scenarios.
</details>

- See how others solved the problem: https://medium.com/@owumifestus/implementing-a-multi-tenancy-solution-in-a-kubernetes-cluster-f90241b28c29
- implement multitenant SaaS on K8s: https://developers.redhat.com/articles/2022/08/12/implement-multitenant-saas-kubernetes#the_role_of_kubernetes_namespaces_in_saas
- https://medium.com/@tarantarantino/tenark-architecting-a-scalable-saas-multi-tenant-platform-with-gitops-822f56bb4580
(Found 2 gitops tools: ArgoCD and FluxCD)

- Learn about GitOps: https://spacelift.io/blog/gitops-tools 

Chose ArgoCD because :
- Natively supports Helm charts and multi-tenancy (via Projects + RBAC)
- Has a nice UI - easier to present for interview
- Features automatic syncs, drift detection, and RBAC for namespace isolation.

For now I'll start with Argo CD (https://spacelift.io/blog/argocd)

Thinking about adding monitoring tool. If I will still have time. 

Understanding the problem:
- Is the problem about multiple customers? (SaaS tenancy - this is the concept that I found on the internet)   
**Response:** yes, it is about creating an environment for customers to deploy their application(s)   

- For the GitOps part, should a tenant correspond to one Git repository that manages all its namespaces, or should each namespace (e.g., tenant1-dev, tenant1-prod) be tied to its own repository?      
**Response:**  normally we make one repo per tenant and let their deploy to different environments (namespaces) from there    

- Do you expect the GitOps setup to manage multiple clusters eventually, or just the local Minikube environment? Approximately how many tenants/customers should I design for?    
**Response:** A single cluster is enough for this exercise and the number of tenants is arbitrary, you don't have to worry about that  

- Should Helm chart support dynamic creation and deletion of tenants and namespaces, or is it a one-time setup?    
**Response:** the creation and deletion (install and uninstall of the chart) should be managed by the GitOps tool


- Are there differences between tenants? (Basic, premium) with different resource allocations?    
**Response:** that is up to you 😊 if you implement it, there will be  

- Should I consider secrets management in this assignment or is it that out of scope?    
**Response:** it is beyond the scope of the assignment, but if you have the time, feel free. It will only make us happy 😃   

1. helm create tenant-bootstrap
2. I first tried to create a tenant and namespaces hardcoded to see how it works
3. I switched to helm upgrade if we see the tenant already exists and helm install for new release name

<details>
<summary>Commands that I used</summary>

    {{- $tenant := default .Release.Name .Values.tenantName -}} 
    {{- if not (kindIs "slice" .Values.namespaces) }}{{ fail "values.namespaces must be a list" }}{{- end }}

    {{- range $ns := .Values.namespaces }}
    apiVersion: v1
    kind: Namespace
    metadata:
    name: {{ printf "%s-%s" $tenant $ns }}
    labels:
        tenant: {{ $tenant }}
        env: {{ $ns }}
    ---
    {{- end }}
</details>

4. Find how to restrict tenants for network, cpu and memory:    
- https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/?utm_source=chatgpt.com 

    <details>
    <summary>Example of a ResourceQuota</summary>

        apiVersion: v1   
    kind: ResourceQuota    
    metadata:    
    name: mem-cpu-demo    
    spec:   
    hard:    
        requests.cpu: "1"    
        requests.memory: 1Gi   
        limits.cpu: "2"    
        limits.memory: 2Gi   


        For every Pod in the namespace, each container must have a memory request, memory limit, cpu request, and cpu limit.
        The memory request total for all Pods in that namespace must not exceed 1 GiB.
        The memory limit total for all Pods in that namespace must not exceed 2 GiB.
        The CPU request total for all Pods in that namespace must not exceed 1 cpu.
        The CPU limit total for all Pods in that namespace must not exceed 2 cpu.
    </details>

5. I realised that ResourceQuota works only for a namespace and not for tenant-level, so I need to create a way to limit the tenant. I'm looking for Hierarchical Namespace Controller (HNC) and Hierarchical Resource Quota (HRQ). Found those links: 
- https://www.redhat.com/en/blog/kubernetes-hierarchical-namespaces 
- https://github.com/kubernetes-retired/hierarchical-namespaces/blob/master/docs/user-guide/how-to.md#use-hrq


Download HNC:
```bash 
kubectl apply -f https://github.com/kubernetes-sigs/hierarchical-namespaces/releases/download/v1.1.0/hrq.yaml    
```

When creating a tenant, first I create the parent, then the namespaces as children and then I apply the HRQ to the parent.

Run the k8s: minikube start -p trifork-homework --cni=cilium 

Ingress = incoming traffic to the pod
Egress = outgoing traffic from the pod

First I blocked all the traffic (netpol-default-deny)
https://kubernetes.io/docs/concepts/services-networking/network-policies/#default-deny-all-ingress-and-all-egress-traffic


Then I allowed DNS traffic (netpol-allow-dns)
Then I allowed egress to the internet (netpol-allow-internet-egress)

Saw a solution to the problem here: https://blog.nashtechglobal.com/how-to-automate-tenant-provisioning-in-kubernetes-with-capsule-and-argocd/ (but it feels cheating)

