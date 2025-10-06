# Trifork Cloud Team Interview - homework exercise

This is a small exercise for us to see how you approach a technical
challenge with very loose guidance on what tools you should use.

On the interview we will want to hear your thoughts on why you made the choices you did,
what you tried, what worked, what didn't work, and what you learned along the way.

We expect you to spend around 3-6 hours on this exercise including the
research of tools and technologies you want to use.

## A cluster with tenants

Your first task is to create an empty Kubernetes cluster. You are welcome to
spin it up locally on your machine.
Your goal in the tasks below is to make the cluster ready to be used by multiple
different tenants (customers). 

### Helm chart for creating tenant namespaces

Create a Helm chart that bootstraps a new namespace for a tenant. The Helm chart should
allow setting the tenant's name and configuring one or more namespaces with the naming convention
&lt;tenant-name&gt;-&lt;namespace-name&gt;.
With the Helm chart you should be able to specify the maximum resources (CPU and Memory)
the tenant is allowed to use. Any deployments in the created namespaces should be allowed
to access the Internet but connectivity between services in- and in-between namespaces should be
restricted.

### GitOps

At Trifork we use GitOps to manage our clusters. We want you to choose a GitOps tool,
deploy it to your cluster, and use it to deploy empty tenant namespaces via the
Helm chart you made in the previous step.

### Deploy a test application

Customers will want to deploy applications in their namespaces. They also want a git repository
where they can push changes and have them automatically deployed to their namespaces.
Deploy a simple test application (e.g. [nginx](https://github.com/nginx/nginx) or
[podinfo](https://github.com/stefanprodan/podinfo)) to one of the namespaces created by your Helm chart.

## Bonus task

### Limit tenant access

In the Helm chart: try to set up Role-Based Access Control (RBAC) so that a tenant
only has access to their own namespace(s). Demonstrate this by creating a different
user for each tenant.
