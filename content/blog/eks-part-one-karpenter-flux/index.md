---
title: EKS Series Part 1. Operationalizing EKS with Karpenter and Flux
date: "2022-11-29"
description: Starting with the basics. Spinning up EKS clusters the DevOps way with Karpenter and Flux GitOps.
---

## Introduction

A good chunk of my work revolves around Kubernetes clusters on AWS. Recently, I've been planning to create a series of posts on EKS clusters with different network and security components available in the CNCF ecosystem - a no-thrills overview of service meshes and CNI solutions aimed at operations teams - in the format of ready-made infrastructure as code examples powered by Pulumi. 

I'm inquisitive about what Cilium will bring us, especially after Release 1.12 and further down the road after CNCF graduation. We may also find a demo use case or two for Linkerd. And so on.

I'm going to start small with the first post of what hopefully will become a series - we will operationalize a barebones EKS cluster with [Karpenter](https://karpenter.sh/v0.19.2/) and [Flux](https://fluxcd.io/). 

Pulumi IaC will help us bring up our infrastructure on the AWS Cloud. Check out pulumi.com if you still need to become familiar with it. You can deploy this demo stack using the Pulumi button below.

[![Deploy](https://get.pulumi.com/new/button.svg)](https://app.pulumi.com/new?template=https://github.com/svodwood/pulumi-eks-karpenter-flux)

You may find the source code for this demo in [this Github repo](https://github.com/svodwood/pulumi-eks-karpenter-flux). A sample Flux configuration repository for this project [is here](https://github.com/svodwood/eks-1-flux-config-repo).

## What We Are Going To Build

We will spin up a demo VPC and a Fargate-enabled EKS cluster. We will additionally create the necessary IAM roles for service accounts to run Karpenter in the cluster. Finally, we will initialize a demo workload to check cluster autoscaling using Flux as our GitOps engine.

To start building, we are going to need to meet the prerequisites:
1. An AWS account with a named AWS CLI profile configured.
2. A working Pulumi environment.
3. A GitHub repo to set up the GitOps flow powered by Flux.
4. A GitHub personal access token.
5. [FluxCLI](https://fluxcd.io/flux/cmd/) installed locally.

## Flux Configuration Repository Set-up
Before launching our Pulumi template, we must set up a Flux repository. This repository will hold all definitions of things running inside our EKS cluster. The demo template assumes we are using a personal public repo on GitHub. When created, clone it locally. Because of an unresolved issue with the [Flux bootstrap process on Fargate](https://github.com/fluxcd/flux2/issues/3138), we will define some kustomizations to the default resource definitions before running the Pulumi stack.

First, create the file structure Flux expects, assuming the EKS cluster name "demo-k8s" (this is the name of the cluster to be provisioned by the stack by default).

```bash
mkdir -p clusters/demo-k8s/flux-system
touch clusters/demo-k8s/flux-system/gotk-components.yaml \
    clusters/demo-k8s/flux-system/gotk-sync.yaml \
    clusters/demo-k8s/flux-system/kustomization.yaml
```
Paste the following into kustomization.yaml, commit and push.
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- gotk-components.yaml
- gotk-sync.yaml
patches:
  - target:
      kind: Deployment
      labelSelector: app.kubernetes.io/part-of=flux
    patch: |
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/cpu
        value: "0.25"
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/cpu
        value: "0.25"
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/memory
        value: 256M
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/memory
        value: 256M
  - patch: |
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: all
      spec:
        template:
          spec:
            nodeSelector:
              $patch: delete
    target:
      kind: Deployment
      labelSelector: app.kubernetes.io/part-of=flux
```

## Pulumi Stack: Basic Infrastructure
First, we will construct a VPC consisting of public subnets, private subnets for the Kubernetes worker nodes and finally, dedicated subnets for the EKS Control Plane network interfaces:
```python
"""
vpc.py
"""
# Create a VPC and Internet Gateway:
demo_vpc = ec2.Vpc("demo-vpc",
    cidr_block=demo_vpc_cidr,
    enable_dns_hostnames=True,
    enable_dns_support=True,
    tags={**general_tags, "Name": f"demo-vpc-{config.region}"}
)

demo_igw = ec2.InternetGateway("demo-igw",
    vpc_id=demo_vpc.id,
    tags={**general_tags, "Name": f"demo-igw-{config.region}"},
    opts=pulumi.ResourceOptions(parent=demo_vpc)
)

# Create a default any-any security group for demo purposes:
demo_sg = ec2.SecurityGroup("demo-security-group",
    description="Allow any-any",
    vpc_id=demo_vpc.id,
    ingress=[ec2.SecurityGroupIngressArgs(
        description="Any",
        from_port=0,
        to_port=0,
        protocol="-1",
        cidr_blocks=["0.0.0.0/0"],
        ipv6_cidr_blocks=["::/0"],
    )],
    egress=[ec2.SecurityGroupEgressArgs(
        from_port=0,
        to_port=0,
        protocol="-1",
        cidr_blocks=["0.0.0.0/0"],
        ipv6_cidr_blocks=["::/0"],
    )],
    tags={**general_tags, "Name": f"demo-sg-{config.region}"},
    opts=pulumi.ResourceOptions(parent=demo_vpc)
)

# Create subnets:
demo_azs = get_availability_zones(state="available").names
demo_public_subnets = []
demo_private_subnets = []
demo_eks_cp_subnets = []

for i in range(2):
    prefix = f"{demo_azs[i]}"
    
    demo_public_subnet = ec2.Subnet(f"demo-public-subnet-{prefix}",
        vpc_id=demo_vpc.id,
        cidr_block=demo_public_subnet_cidrs[i],
        availability_zone=demo_azs[i],
        tags={**general_tags, "Name": f"demo-public-subnet-{prefix}"},
        opts=pulumi.ResourceOptions(parent=demo_vpc)
    )
    
    demo_public_subnets.append(demo_public_subnet)

    demo_public_route_table = ec2.RouteTable(f"demo-public-rt-{prefix}",
        vpc_id=demo_vpc.id,
        tags={**general_tags, "Name": f"demo-public-rt-{prefix}"},
        opts=pulumi.ResourceOptions(parent=demo_public_subnet)
    )
    
    demo_public_route_table_association = ec2.RouteTableAssociation(f"demo-public-rt-association-{prefix}",
        route_table_id=demo_public_route_table.id,
        subnet_id=demo_public_subnet.id,
        opts=pulumi.ResourceOptions(parent=demo_public_subnet)
    )

    demo_public_wan_route = ec2.Route(f"demo-public-wan-route-{prefix}",
        route_table_id=demo_public_route_table.id,
        gateway_id=demo_igw.id,
        destination_cidr_block="0.0.0.0/0",
        opts=pulumi.ResourceOptions(parent=demo_public_subnet)
    )

    demo_eip = ec2.Eip(f"demo-eip-{prefix}",
        tags={**general_tags, "Name": f"demo-eip-{prefix}"},
        opts=pulumi.ResourceOptions(parent=demo_vpc)
    )
    
    demo_nat_gateway = ec2.NatGateway(f"demo-nat-gateway-{prefix}",
        allocation_id=demo_eip.id,
        subnet_id=demo_public_subnet.id,
        tags={**general_tags, "Name": f"demo-nat-{prefix}"},
        opts=pulumi.ResourceOptions(depends_on=[demo_vpc])
    )

    demo_private_subnet = ec2.Subnet(f"demo-private-subnet-{prefix}",
        vpc_id=demo_vpc.id,
        cidr_block=demo_private_subnet_cidrs[i],
        availability_zone=demo_azs[i],
        tags={**general_tags, "Name": f"demo-private-subnet-{prefix}", "karpenter.sh/discovery": f"{cluster_descriptor}"},
        opts=pulumi.ResourceOptions(parent=demo_vpc)
    )
    
    demo_private_subnets.append(demo_private_subnet)

    demo_private_route_table = ec2.RouteTable(f"demo-private-rt-{prefix}",
        vpc_id=demo_vpc.id,
        tags={**general_tags, "Name": f"demo-private-rt-{prefix}"},
        opts=pulumi.ResourceOptions(parent=demo_private_subnet)
    )
    
    demo_private_route_table_association = ec2.RouteTableAssociation(f"demo-private-rt-association-{prefix}",
        route_table_id=demo_private_route_table.id,
        subnet_id=demo_private_subnet.id,
        opts=pulumi.ResourceOptions(parent=demo_private_subnet)
    )

    demo_private_wan_route = ec2.Route(f"demo-private-wan-route-{prefix}",
        route_table_id=demo_private_route_table.id,
        nat_gateway_id=demo_nat_gateway.id,
        destination_cidr_block="0.0.0.0/0",
        opts=pulumi.ResourceOptions(parent=demo_private_subnet)
    )

    demo_eks_cp_subnet = ec2.Subnet(f"demo-eks-cp-subnet-{prefix}",
        vpc_id=demo_vpc.id,
        cidr_block=demo_eks_cp_subnet_cidrs[i],
        availability_zone=demo_azs[i],
        tags={**general_tags, "Name": f"demo-eks-cp-subnet-{prefix}"},
        opts=pulumi.ResourceOptions(parent=demo_vpc)
    )
    
    demo_eks_cp_subnets.append(demo_eks_cp_subnet)

    demo_eks_cp_route_table = ec2.RouteTable(f"demo-eks-cp-rt-{prefix}",
        vpc_id=demo_vpc.id,
        tags={**general_tags, "Name": f"demo-eks-cp-rt-{prefix}"},
        opts=pulumi.ResourceOptions(parent=demo_eks_cp_subnet)
    )
    
    demo__eks_cp_route_table_association = ec2.RouteTableAssociation(f"demo-eks-cp-rt-association-{prefix}",
        route_table_id=demo_eks_cp_route_table.id,
        subnet_id=demo_eks_cp_subnet.id,
        opts=pulumi.ResourceOptions(parent=demo_eks_cp_subnet)
    )

    demo_eks_cp_wan_route = ec2.Route(f"demo-eks-cp-wan-route-{prefix}",
        route_table_id=demo_eks_cp_route_table.id,
        nat_gateway_id=demo_nat_gateway.id,
        destination_cidr_block="0.0.0.0/0",
        opts=pulumi.ResourceOptions(parent=demo_eks_cp_subnet)
    )
```
We will additionally create all VPC endpoints required to operate a private EKS cluster:
```python
"""
vpc_endpoints.py
"""
# Create a shared security group for all AWS services VPC endpoints:
vpc_endpoints_sg = ec2.SecurityGroup("aws-vpc-endpoints-sg",
    description="Shared security groups for AWS services VPC endpoints",
    vpc_id=demo_vpc.id,
    tags={**general_tags, "Name": "aws-vpc-endpoints-sg"}
)

inbound_endpoints_cidrs = ec2.SecurityGroupRule("inbound-vpc-endpoint-cidrs",
    type="ingress",
    from_port=443,
    to_port=443,
    protocol="tcp",
    cidr_blocks=["0.0.0.0/0"],
    security_group_id=vpc_endpoints_sg.id
)

outbound_endpoints_cidrs = ec2.SecurityGroupRule("outbound-vpc-endpoint-cidrs",
    type="egress",
    to_port=0,
    protocol="-1",
    from_port=0,
    cidr_blocks=["0.0.0.0/0"],
    security_group_id=vpc_endpoints_sg.id
)

# Create VPC endpoints:
endpoints = []
for service in endpoint_services:
    if service == "s3":
        enable_dns = False
    else:
        enable_dns = True
    endpoints.append(ec2.VpcEndpoint(f"{service.replace('.','-')}-vpc-endpoint",
        vpc_id = demo_vpc.id,
        service_name = f"com.amazonaws.{deployment_region}.{service}",
        vpc_endpoint_type = "Interface",
        subnet_ids = [s.id for s in demo_private_subnets],
        security_group_ids = [vpc_endpoints_sg.id],
        private_dns_enabled = enable_dns,
        tags = {**general_tags, "Name": f"{service.replace('.','-')}-vpc-endpoint"},
        opts = ResourceOptions(
            depends_on=[vpc_endpoints_sg],
            parent=demo_vpc)
        ))
```

With our basic infrastructure completed, we can continue configuring our EKS cluster.

## Pulumi Stack: EKS, Fargate, Karpenter Role, Flux Bootstrap
First, we will take care of several IAM objects necessary to construct our working EKS environment, namely:
1. An EKS service role.
2. A Fargate pod execution role.
3. A Karpenter-managed node instance profile.

Second, we will spin up an EKS cluster with a Fargate profile covering pod provisioning in "karpenter", "kube-system", and "flux-system" namespaces. These are the only namespaces to be created as part of the cluster provisioning routine - bootstrapping the cluster to a minimal working state can be outsourced to Operations and subsequently passed to a development team for consumption. In this example, the development teams, in turn, can adopt the cluster using the established GitOps workflow. In addition, ops teams may use role-based access control to restrict the manipulation of resources in these "operations-managed" namespaces. Finally, we will set up an IAM role for the Karpenter service account.

It is a deliberate choice to run CoreDNS on Fargate. We will leave this approach's pros and cons outside this guide's scope.

```python
"""
eks.py
"""
# Create an EKS cluster role
eks_iam_role_policy_arns = [
    "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy",
    "arn:aws:iam::aws:policy/AmazonEKSVPCResourceController"
]

eks_iam_role = create_iam_role(f"{cluster_descriptor}-eks-role", "Service", "eks.amazonaws.com", eks_iam_role_policy_arns)

# Create a default node role for Karpenter
karpenter_default_nodegroup_role_policy_arns = [
    "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy",
    "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly",
    "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore",
    "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
]

# Create a default Fargate execution role:
default_fargate_pod_execution_role_policy = [
    "arn:aws:iam::aws:policy/AmazonEKSFargatePodExecutionRolePolicy"
]

# VPC CNI service account policies:
cni_service_account_policy_arns = [
    "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
]

# Create a default demo EKS cluster nodegroup security group:
demo_nodegroup_security_group = ec2.SecurityGroup(f"custom-node-attach-{cluster_descriptor}",
    description=f"{cluster_descriptor} custom node security group",
    vpc_id=demo_vpc.id,
    tags={**general_tags, "Name": f"custom-node-attach-{cluster_descriptor}", "karpenter.sh/discovery": f"{cluster_descriptor}"}
)

demo_nodegroup_security_group_inbound_custom_cidrs = ec2.SecurityGroupRule(f"inbound-eks-node-{cluster_descriptor}",
    type="ingress",
    from_port=443,
    to_port=443,
    protocol="tcp",
    cidr_blocks=["0.0.0.0/0"],
    security_group_id=demo_nodegroup_security_group.id
)

demo_nodegroup_security_group_oubound_custom_cidrs = ec2.SecurityGroupRule(f"outbound-eks-node-{cluster_descriptor}",
    type="egress",
    to_port=0,
    protocol="-1",
    from_port=0,
    cidr_blocks=["0.0.0.0/0"],
    security_group_id=demo_nodegroup_security_group.id
)

# Create a default demo EKS cluster security group:
demo_cluster_security_group = ec2.SecurityGroup(f"custom-cluster-attach-{cluster_descriptor}",
    description=f"{cluster_descriptor} custom security group",
    vpc_id=demo_vpc.id,
    tags={**general_tags, "Name": f"custom-cluster-attach-{cluster_descriptor}"}
)

demo_cluster_security_group_inbound_custom_cidrs = ec2.SecurityGroupRule(f"inbound-eks-cp-{cluster_descriptor}",
    type="ingress",
    from_port=443,
    to_port=443,
    protocol="tcp",
    cidr_blocks=["0.0.0.0/0"],
    security_group_id=demo_cluster_security_group.id
)

demo_cluster_security_group_oubound_custom_cidrs = ec2.SecurityGroupRule(f"outbound-eks-cp-{cluster_descriptor}",
    type="egress",
    to_port=0,
    protocol="-1",
    from_port=0,
    cidr_blocks=["0.0.0.0/0"],
    security_group_id=demo_cluster_security_group.id
)

# Create a default Karpenter node role and instance profile:
karpenter_node_role = create_iam_role(f"KarpenterNodeRole-{cluster_descriptor}", "Service", "ec2.amazonaws.com", karpenter_default_nodegroup_role_policy_arns)
karpenter_instance_profile = iam.InstanceProfile(f"KarpenterNodeInstanceProfile-{cluster_descriptor}",
    role=karpenter_node_role.name,
    name=f"KarpenterNodeInstanceProfile-{cluster_descriptor}"
)

# Create a Fargate profile service role:
fargate_profile_service_role = create_iam_role(f"{cluster_descriptor}-fargate-role", "Service", "eks-fargate-pods.amazonaws.com", default_fargate_pod_execution_role_policy)

# Create an EKS log group:
demo_eks_loggroup = cloudwatch.LogGroup("demo-eks-loggroup", 
    name=f"/aws/eks/{cluster_descriptor}/cluster",
    tags=general_tags,
    retention_in_days=1
)

# Create the cluster control plane and Fargate profiles:
demo_eks_cluster = eks_provider.Cluster(f"eks-{cluster_descriptor}",
    name=f"{cluster_descriptor}",
    vpc_id=demo_vpc.id,
    instance_role=karpenter_node_role,
    cluster_security_group=demo_cluster_security_group,
    create_oidc_provider=True,
    version="1.24",
    instance_profile_name=karpenter_instance_profile,
    skip_default_node_group=True,
    service_role=eks_iam_role,
    provider_credential_opts=eks_provider.KubeconfigOptionsArgs(
        profile_name=config.profile,
    ),
    endpoint_private_access=True,
    endpoint_public_access=True,
    enabled_cluster_log_types=["api", "audit", "authenticator", "controllerManager", "scheduler"],
    public_access_cidrs=["0.0.0.0/0"],
    subnet_ids=[s.id for s in demo_eks_cp_subnets],
    tags={**general_tags, "Name": f"{cluster_descriptor}"},
    fargate=eks.FargateProfileArgs(
        cluster_name=f"{cluster_descriptor}",
        subnet_ids=[s.id for s in demo_private_subnets],
        pod_execution_role_arn=fargate_profile_service_role.arn,
        selectors=[{"namespace": "karpenter"}, {"namespace": "flux-system"}, {"namespace": "kube-system"}]),
    opts=ResourceOptions(depends_on=[
            demo_nodegroup_security_group,
            eks_iam_role,
            demo_eks_loggroup
        ]))

demo_eks_cluster_oidc_arn = demo_eks_cluster.core.oidc_provider.arn
demo_eks_cluster_oidc_url = demo_eks_cluster.core.oidc_provider.url

# Create an IAM role for VPC CNI:
iam_role_vpc_cni_service_account = create_oidc_role(f"{cluster_descriptor}-aws-node", "kube-system", demo_eks_cluster_oidc_arn, demo_eks_cluster_oidc_url, "aws-node", cni_service_account_policy_arns)

# Create a Karpenter IAM role scoped to karpenter namespace
iam_role_karpenter_controller_policy = create_policy(f"{cluster_descriptor}-karpenter-policy", "karpenter_oidc_role_policy.json")
iam_role_karpenter_controller_service_account_role = create_oidc_role(f"{cluster_descriptor}-karpenter", "karpenter", demo_eks_cluster_oidc_arn, demo_eks_cluster_oidc_url, "karpenter", [iam_role_karpenter_controller_policy.arn])
export("karpenter-oidc-role-arn", iam_role_karpenter_controller_service_account_role.arn)


# Install VPC CNI addon when the cluster is initialized:
vpc_cni_addon = eks.Addon("vpc-cni-addon",
    cluster_name=f"{cluster_descriptor}",
    addon_name="vpc-cni",
    addon_version="v1.11.4-eksbuild.1",
    resolve_conflicts="OVERWRITE",
    service_account_role_arn=iam_role_vpc_cni_service_account.arn,
    opts=ResourceOptions(
        depends_on=[demo_eks_cluster]
    )
)

# Install kube-proxy addon when the cluster is initialized:
kube_proxy_addon = eks.Addon("kube-proxy-addon",
    cluster_name=f"{cluster_descriptor}",
    addon_name="kube-proxy",
    addon_version="v1.24.7-eksbuild.2",
    resolve_conflicts="OVERWRITE",
    opts=ResourceOptions(
        depends_on=[demo_eks_cluster, vpc_cni_addon]
    )
)

# Install CoreDNS addon when the cluster is initialized:
core_dns_addon = eks.Addon("coredns-addon",
    cluster_name=f"{cluster_descriptor}",
    addon_name="coredns",
    addon_version="v1.8.7-eksbuild.3",
    resolve_conflicts="OVERWRITE",
    opts=ResourceOptions(
        depends_on=[demo_eks_cluster, vpc_cni_addon, kube_proxy_addon]
    )
)

# Create a kubernetes provider
role_provider = k8s.Provider(f"{cluster_descriptor}-kubernetes-provider",
    kubeconfig=demo_eks_cluster.kubeconfig,
    opts=ResourceOptions(depends_on=[demo_eks_cluster])
)

# Create a service account and cluster role binding for flux controller
flux_service_account = k8s.core.v1.ServiceAccount("flux-controller-service-account",
    api_version="v1",
    kind="ServiceAccount",
    metadata=k8s.meta.v1.ObjectMetaArgs(
        name="flux-controller",
        namespace="kube-system"
    ),
    opts=ResourceOptions(
        provider=role_provider,
        depends_on=[demo_eks_cluster]
    )
)

flux_controller_cluster_role_binding = k8s.rbac.v1.ClusterRoleBinding("flux-controller-sa-crb",
    role_ref=k8s.rbac.v1.RoleRefArgs(
        api_group="rbac.authorization.k8s.io",
        kind="ClusterRole",
        name="cluster-admin"
    ),
    subjects=[
        k8s.rbac.v1.SubjectArgs(
            kind="ServiceAccount",
            name="flux-controller",
            namespace="kube-system"
        )
    ],
    opts=ResourceOptions(
        provider=role_provider,
        depends_on=[demo_eks_cluster]
    )
)

# Create the bootstrap job for flux-cli
flux_bootstrap_job = k8s.batch.v1.Job("fluxBootstrapJob",
    opts=ResourceOptions(
        provider=role_provider,
        depends_on=[flux_service_account, flux_controller_cluster_role_binding]
    ),
    metadata=k8s.meta.v1.ObjectMetaArgs(
        name="flux-bootstrap-job",
        namespace="kube-system",
        annotations={"pulumi.com/replaceUnready": "true"}
    ),
    spec=k8s.batch.v1.JobSpecArgs(
        backoff_limit=3,
        template=k8s.core.v1.PodTemplateSpecArgs(
            spec=k8s.core.v1.PodSpecArgs(
                service_account_name="flux-controller",
                containers=[k8s.core.v1.ContainerArgs(
                    env=[k8s.core.v1.outputs.EnvVar(
                        name="GITHUB_TOKEN",
                        value=flux_github_token
                    )],
                    command=[
                        "flux",
                        "bootstrap",
                        "github",
                        f"--owner={flux_github_repo_owner}",
                        f"--repository={flux_github_repo_name}",
                        f"--path=./clusters/{cluster_descriptor}",
                        "--private=false",
                        "--personal=true"
                    ],
                    image=f"fluxcd/flux-cli:v{flux_cli_version}",
                    name="flux-bootstrap",
                )],
                restart_policy="OnFailure",
            ),
        ),
    ))

# Create a karpenter namespace:
karpenter_namespace = k8s.core.v1.Namespace("karpenter-namespace",
    metadata={"name": "karpenter"},
    opts=ResourceOptions(
        provider=role_provider,
        depends_on=[demo_eks_cluster]
    )
)
```
Let's break down the above code into steps. 

First, we created two custom security groups: one for the EKS cluster (```custom-cluster-attach-demo-k8s```) and one for the Karpenter-managed nodes (```custom-node-attach-demo-k8s```). We will keep these "ANY-ANY" for the sake of the demonstration. We created these to provide future fine-grained controls over the Karpenter node to control plane network traffic and traffic coming in from the potential load balancers we might have. Notice that the node security group has a tag ```"karpenter.sh/discovery": "k8s-demo"``` that would allow the Karpenter controller to auto-discover this security group and attach it to a node.

Second, we created a default instance profile for Karpenter-managed nodes. This profile has to have several managed policies attached, as outlined in Karpenter documentation. Additionally, we made a default Fargate profile role to use with Fargate.

Finally, we feed everything to the EKS cluster definition, create the cluster and export its OIDC provider configuration. We will need these to provision an IAM role for the Karpenter service account. Note this role's ARN via the stack's export variable ```karpenter-oidc-role-arn```.

Once our EKS cluster is ready, we will use the ```pulumi-kubernetes``` provider to create our ```flux-cli``` bootstrap job, feeding it the GitHub repository name and authentication token. This way, we effectively use our infrastructure-as-code stack to set up EKS to manage itself via GitOps. 

>We can establish a boundary between the operations/platform team (the enabling infrastructure team or function) and the platform's end users. From here on out, resources may be managed entirely by GitOps, with several considerations. This way, we enable broader team autonomy and less backlog coupling. Exceptions may include IAM roles for additional service accounts, security groups and rules - essentially all AWS objects we might need for the workloads running on Kubernetes to function.

Pulumi up!

## Installing Things the GitOps Way: Karpenter and Demo Workload

Once our Pulumi stack has successfully run, we can add Karpenter installation to our Flux configuration repository. Now go ahead and pull the most recent changes that Flux bootstrap committed to our configuration repo. Let's install Karpenter:

```bash
mkdir -p clusters/demo-k8s/karpenter
touch clusters/demo-k8s/karpenter/release.yaml \
    clusters/demo-k8s/karpenter/source.yaml
```
source.yaml:
```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: karpenter
  namespace: karpenter
spec:
  interval: 1m
  url: oci://public.ecr.aws/karpenter
  type: oci
```
release.yaml:
```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: karpenter
  namespace: karpenter
spec:
  targetNamespace: karpenter
  interval: 5m
  chart:
    spec:
      chart: karpenter
      version: v0.19.2
      sourceRef:
        kind: HelmRepository
        name: karpenter
        namespace: karpenter
      interval: 10m
  values:
    replicas: 1
    serviceAccount:
      create: true
      name: "karpenter"
      annotations:
        eks.amazonaws.com/role-arn: INSERT KARPENTER ROLE ARN ("karpenter-oidc-role-arn" stack export)
    settings:
      aws:
        clusterName: "demo-k8s"
        clusterEndpoint: "INSERT HTTPS CLUSTER ENDPOINT OBTAINED FROM EKS"
        defaultInstanceProfile: "KarpenterNodeInstanceProfile-demo-k8s"
    logLevel: info
  install:
    crds: CreateReplace
  upgrade:
    crds: CreateReplace
```
Commit your changes and push. Verify that Flux successfully reconciled both HelmRepository and HelmRelease:
```bash
kubectl -n karpenter get helmrepository
kubectl -n karpenter get helmrelease
```
We should see a "Helm repository is ready" status and "Release reconciliation succeeded", respectively.

Now, let's test our Karpenter in action. For this to work, we have to create a Provisioner:
```bash
mkdir -p clusters/demo-k8s/karpenter-provisioner
touch clusters/demo-k8s/karpenter-provisioner/default.yaml
```
default.yaml:
```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  labels:
    node-type: demo-karpenter
  requirements:
    - key: "karpenter.sh/capacity-type"
      operator: In
      values: ["on-demand"]
    - key: "kubernetes.io/arch"
      operator: In
      values: ["amd64"]
    - key: "node.kubernetes.io/instance-type"
      operator: In
      values: ["t3.small", "t3.medium", "t3a.small", "t3a.medium"]
  limits:
    resources:
      cpu: 10
      memory: 10Gi
  providerRef:
    name: default
  ttlSecondsAfterEmpty: 30
  ttlSecondsUntilExpired: 43200
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default
spec:
  amiFamily: Bottlerocket
  subnetSelector:
    karpenter.sh/discovery: demo-k8s
  securityGroupSelector:
    karpenter.sh/discovery: demo-k8s
  tags:
    Name: "KarpenterWorkerNode"
```
Commit your changes and push. Then, run ```kubectl get provisioner``` to verify successful reconciliation. Finally, our infrastructure set-up is complete.

## Summing It All Up

Let's create a demo workload.

inflate.yaml:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
  namespace: default
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      nodeSelector:
        node-type: demo-karpenter
      terminationGracePeriodSeconds: 0
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
          resources:
            requests:
              cpu: 1
```

Let's quickly install it from the command line, scale out our deployment and observe Karpenter in action:
```bash
kubectl apply -f inflate.yaml
kubectl scale deployment inflate --replicas 5
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller
```
In a matter of seconds, we will observe a scaling-out event. Success!

## Cleaning Up
Before running ```pulumi destroy```, it is best to run ```flux uninstall``` first to remove all finalizers that might otherwise interfere with the stack's resources and delete the EC2 instances that Karpenter spun up for us. Then, after Flux is uninstalled and the instances are terminated, run ```pulumi destroy``` to clean up.