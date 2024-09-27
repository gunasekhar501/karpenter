# *AWS Karpenter Installation Guide*

# **Overview:**
AWS Karpenter is a powerful, flexible, and efficient Kubernetes cluster autoscaler designed to provide just-in-time compute resources to meet your application's needs. It automatically optimizes your cluster’s compute resource footprint to reduce costs and improve performance. Karpenter dynamically provisions new nodes in response to unschedulable pods by observing events within the Kubernetes cluster and sending commands to the underlying cloud provider.

# **Key Features:**
1. **Dynamic Scaling:** Automatically provisions nodes based on current workload demands.
2. **Cost Optimization:** Selects the most cost-effective instance types and sizes.
3. **Performance Improvement:** Responds quickly to changing workloads, ensuring optimal performance.
4. **Direct Node Provisioning:** Provides fine-grained control over instance types, sizes, and configurations.

# **Prerequisites:**
Before installing Karpenter, ensure the following prerequisites are met:

1. **Amazon EKS Cluster:** An existing EKS cluster with nodes. If you don't have one, create a new EKS cluster.
2. **IAM Permissions:** The AWS CLI user/role must have the necessary permissions to create IAM roles, policies, and service-linked roles.
3. **AWS CLI:** AWS CLI should be installed, configured, and connected to your AWS account.
4. **kubectl:** kubectl should be installed and configured to interact with your EKS cluster.
5. **Helm:** Helm should be installed to manage Kubernetes packages.
# **Karpenter vs. Cluster Autoscaler (CAS)**
Both Karpenter and Cluster Autoscaler are designed to automate node scaling in Kubernetes clusters, but they differ in their approach and capabilities:

# **Cluster Autoscaler (CAS):**

Adjusts the number of nodes based on pending pods that can't be scheduled due to insufficient resources.
Works with predefined node groups (e.g., AWS Auto Scaling Groups).
Can only scale nodes up, relying on AWS ASG termination policies to scale down.
Less optimized for cost, as it scales entire node groups.
Karpenter:

Provides flexible, next-generation autoscaling that provisions and configures individual instances directly.
Can scale nodes down by terminating instances when no longer needed, leading to more efficient scaling.
Optimizes for cost, availability, and performance by dynamically selecting the most appropriate instance types and sizes based on the current workload.
Potentially leads to significant cost savings by provisioning nodes that best suit the current workload.
IAM Role and Policy Configuration
**Create IAM Role for Worker Nodes:**
Create a new role for managing Worker Nodes and attach the following policies. Name this role KarpenterNodeRole:

1. **AmazonEBSCSIDriverPolicy**
2. **AmazonEKSWorkerNodePolicy**
3. **AmazonEKS_CNI_Policy**
4. **AmazonEC2ContainerRegistryReadOnly**
5. **AmazonSSMManagedInstanceCore**

**Trust Relationship Policy:**

Update the trust relationship policy for the role as follows:

```python
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```
**Create IAM Policy for Karpenter Controller:**
Create a new IAM policy for the Karpenter Controller and add the following code in the JSON section. Save this policy as KarpenterControllerPolicy.

```markdown
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowScopedEC2InstanceAccessActions",
            "Effect": "Allow",
            "Action": [
                "ec2:RunInstances",
                "ec2:CreateFleet"
            ],
            "Resource": [
                "arn:aws:ec2:<region-name>::image/*",
                "arn:aws:ec2:<region-name>:<aws_account_id>:image/*",
                "arn:aws:ec2:<region-name>:<aws_account_id>:snapshot/*",
                "arn:aws:ec2:<region-name>:<aws_account_id>:security-group/*",
                "arn:aws:ec2:<region-name>:<aws_account_id>:subnet/*"
            ]
        },
        {
            "Sid": "AllowScopedEC2LaunchTemplateAccessActions",
            "Effect": "Allow",
            "Action": [
                "ec2:RunInstances",
                "ec2:CreateFleet"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:ResourceTag/kubernetes.io/cluster/<cluster-name>": "owned"
                },
                "StringLike": {
                    "aws:ResourceTag/karpenter.sh/nodepool": "*"
                }
            },
            "Resource": "arn:aws:ec2:<region-name>:<aws_account_id>:launch-template/*"
        },
        {
            "Sid": "AllowScopedEC2InstanceActionsWithTags",
            "Effect": "Allow",
            "Action": [
                "ec2:RunInstances",
                "ec2:CreateFleet",
                "ec2:CreateLaunchTemplate"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:RequestTag/kubernetes.io/cluster/<cluster-name>": "owned"
                },
                "StringLike": {
                    "aws:RequestTag/karpenter.sh/nodepool": "*"
                }
            },
            "Resource": [
                "arn:aws:ec2:<region-name>:<aws_account_id>:fleet/*",
                "arn:aws:ec2:<region-name>:<aws_account_id>:instance/*",
                "arn:aws:ec2:<region-name>:<aws_account_id>:volume/*",
                "arn:aws:ec2:<region-name>:<aws_account_id>:network-interface/*",
                "arn:aws:ec2:<region-name>:<aws_account_id>:launch-template/*",
                "arn:aws:ec2:<region-name>:<aws_account_id>:spot-instances-request/*"
            ]
        },
        {
            "Sid": "AllowScopedResourceCreationTagging",
            "Effect": "Allow",
            "Action": "ec2:CreateTags",
            "Condition": {
                "StringEquals": {
                    "aws:RequestTag/kubernetes.io/cluster/<cluster-name>": "owned",
                    "ec2:CreateAction": [
                        "RunInstances",
                        "CreateFleet",
                        "CreateLaunchTemplate"
                    ]
                },
                "StringLike": {
                    "aws:RequestTag/karpenter.sh/nodepool": "*"
                }
            },
            "Resource": [
                "arn:aws:ec2:<region-name>:<aws_account_id>:fleet/*",
                "arn:aws:ec2:<region-name>:<aws_account_id>:instance/*",
                "arn:aws:ec2:<region-name>:<aws_account_id>:volume/*",
                "arn:aws:ec2:<region-name>:<aws_account_id>:network-interface/*",
                "arn:aws:ec2:<region-name>:<aws_account_id>:launch-template/*",
                "arn:aws:ec2:<region-name>:<aws_account_id>:spot-instances-request/*"
            ]
        },
        {
            "Sid": "AllowScopedResourceTagging",
            "Effect": "Allow",
            "Action": "ec2:CreateTags",
            "Condition": {
                "ForAllValues:StringEquals": {
                    "aws:TagKeys": [
                        "karpenter.sh/nodeclaim",
                        "Name"
                    ]
                },
                "StringEquals": {
                    "aws:ResourceTag/kubernetes.io/cluster/<cluster-name>": "owned"
                },
                "StringLike": {
                    "aws:ResourceTag/karpenter.sh/nodepool": "*"
                }
            },
            "Resource": "arn:aws:ec2:<region-name>:<aws_account_id>:instance/*"
        },
        {
            "Sid": "AllowScopedDeletion",
            "Effect": "Allow",
            "Action": [
                "ec2:TerminateInstances",
                "ec2:DeleteLaunchTemplate"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:ResourceTag/kubernetes.io/cluster/<cluster-name>": "owned"
                },
                "StringLike": {
                    "aws:ResourceTag/karpenter.sh/nodepool": "*"
                }
            },
            "Resource": [
                "arn:aws:ec2:<region-name>:<aws_account_id>:instance/*",
                "arn:aws:ec2:<region-name>:<aws_account_id>:launch-template/*"
            ]
        },
        {
            "Sid": "AllowRegionalReadActions",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeAvailabilityZones",
                "ec2:DescribeImages",
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceTypeOfferings",
                "ec2:DescribeInstanceTypes",
                "ec2:DescribeLaunchTemplates",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeSpotPriceHistory",
                "ec2:DescribeSubnets"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:RequestedRegion": "<region-name>"
                }
            },
            "Resource": "*"
        },
        {
            "Sid": "AllowSSMReadActions",
            "Effect": "Allow",
            "Action": "ssm:GetParameter",
            "Resource": "arn:aws:ssm:<region-name>::parameter/aws/service/*"
        },
        {
            "Sid": "AllowPricingReadActions",
            "Effect": "Allow",
            "Action": "pricing:GetProducts",
            "Resource": "*"
        },
        {
            "Sid": "AllowSTSAssumeRole",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::<aws_account_id>:role/KarpenterNodeRole"
        },
        {
            "Sid": "AllowIAMPassRole",
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::<aws_account_id>:role/KarpenterNodeRole"
        }
    ]
}
```
**Policy Creation:**
Trust Policy
First, we need to set up a trust policy to allow our IAM role to be assumed by Karpenter. The trust policy should look like the following JSON:


````console
{
   "Version": "2012-10-17",
   "Statement": [
       {
           "Effect": "Allow",
           "Principal": {
               "Federated": "arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_ENDPOINT#*//}"
           },
           "Action": "sts:AssumeRoleWithWebIdentity",
           "Condition": {
               "StringEquals": {
                   "${OIDC_ENDPOINT#*//}:aud": "sts.amazonaws.com",
                   "${OIDC_ENDPOINT#*//}:sub": "system:serviceaccount:${KARPENTER_NAMESPACE}:${KARPENTER_SERVICE_ACCOUNT}"
               }
           }
       }
   ]
}
````
**IAM Role Creation:**
Create the IAM role using the above trust policy. The resulting role in the AWS Console will have the trust policy applied.

**Permissions Policy:**
Attach the following permissions to the IAM role created above:

```console
{
   "Version": "2012-10-17",
   "Statement": [
       {
           "Sid": "AllowScopedEC2InstanceAccessActions",
           "Effect": "Allow",
           "Resource": [
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}::image/*",
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:image/*",
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:snapshot/*",
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:security-group/*",
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:subnet/*"
           ],
           "Action": [
               "ec2:RunInstances",
               "ec2:CreateFleet"
           ]
       },
       {
           "Sid": "AllowScopedEC2LaunchTemplateAccessActions",
           "Effect": "Allow",
           "Resource": "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:launch-template/*",
           "Action": [
               "ec2:RunInstances",
               "ec2:CreateFleet"
           ],
           "Condition": {
               "StringEquals": {
                   "aws:ResourceTag/kubernetes.io/cluster/${var.cluster_name}": "owned"
               },
               "StringLike": {
                   "aws:ResourceTag/karpenter.sh/nodepool": "*"
               }
           }
       },
       {
           "Sid": "AllowScopedEC2InstanceActionsWithTags",
           "Effect": "Allow",
           "Resource": [
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:fleet/*",
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:instance/*",
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:volume/*",
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:network-interface/*",
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:launch-template/*",
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:spot-instances-request/*"
           ],
           "Action": [
               "ec2:RunInstances",
               "ec2:CreateFleet",
               "ec2:CreateLaunchTemplate"
           ],
           "Condition": {
               "StringEquals": {
                   "aws:RequestTag/kubernetes.io/cluster/${var.cluster_name}": "owned"
               },
               "StringLike": {
                   "aws:RequestTag/karpenter.sh/nodepool": "*"
               }
           }
       },
       {
           "Sid": "AllowScopedResourceCreationTagging",
           "Effect": "Allow",
           "Resource": [
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:fleet/*",
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:instance/*",
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:volume/*",
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:network-interface/*",
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:launch-template/*",
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:spot-instances-request/*"
           ],
           "Action": "ec2:CreateTags",
           "Condition": {
               "StringEquals": {
                   "aws:RequestTag/kubernetes.io/cluster/${var.cluster_name}": "owned",
                   "ec2:CreateAction": [
                       "RunInstances",
                       "CreateFleet",
                       "CreateLaunchTemplate"
                   ]
               },
               "StringLike": {
                   "aws:RequestTag/karpenter.sh/nodepool": "*"
               }
           }
       },
       {
           "Sid": "AllowScopedResourceTagging",
           "Effect": "Allow",
           "Resource": "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:instance/*",
           "Action": "ec2:CreateTags",
           "Condition": {
               "StringEquals": {
                   "aws:ResourceTag/kubernetes.io/cluster/${var.cluster_name}": "owned"
               },
               "StringLike": {
                   "aws:ResourceTag/karpenter.sh/nodepool": "*"
               },
               "ForAllValues:StringEquals": {
                   "aws:TagKeys": [
                       "karpenter.sh/nodeclaim",
                       "Name"
                   ]
               }
           }
       },
       {
           "Sid": "AllowScopedDeletion",
           "Effect": "Allow",
           "Resource": [
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:instance/*",
               "arn:${data.aws_partition.current.partition}:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:launch-template/*"
           ],
           "Action": [
               "ec2:TerminateInstances",
               "ec2:DeleteLaunchTemplate"
           ],
           "Condition": {
               "StringEquals": {
                   "aws:ResourceTag/kubernetes.io/cluster/${var.cluster_name}": "owned"
               },
               "StringLike": {
                   "aws:ResourceTag/karpenter.sh/nodepool": "*"
               }
           }
       },
       {
           "Sid": "AllowRegionalReadActions",
           "Effect": "Allow",
           "Resource": "*",
           "Action": [
               "ec2:DescribeAvailabilityZones",
               "ec2:DescribeImages",
               "ec2:DescribeInstances",
               "ec2:DescribeInstanceTypeOfferings",
               "ec2:DescribeInstanceTypes",
               "ec2:DescribeLaunchTemplates",
               "ec2:DescribeSecurityGroups",
               "ec2:DescribeSpotPriceHistory",
               "ec2:DescribeSubnets"
           ],
           "Condition": {
               "StringEquals": {
                   "aws:RequestedRegion": "${var.region}"
               }
           }
       },
       {
           "Sid": "AllowSSMReadActions",
           "Effect": "Allow",
           "Resource": "arn:${data.aws_partition.current.partition}:ssm:${data.aws_region.current.name}::parameter/aws/service/*",
           "Action": "ssm:GetParameter"
       },
       {
           "Sid": "AllowPricingReadActions",
           "Effect": "Allow",
           "Resource": "*",
           "Action": "pricing:GetProducts"
       },
       {
           "Sid": "AllowInterruptionQueueActions",
           "Effect": "Allow",
           "Resource": "${module.karpenterinterruptionqueue.sqs_queue_arn}",
           "Action": [
               "sqs:DeleteMessage",
               "sqs:GetQueueUrl",
               "sqs:ReceiveMessage"
           ]
       },
       {
           "Sid": "AllowPassingInstanceRole",
           "Effect": "Allow",
           "Resource": "${module.KarpenterNodeRole.iam_role_arn}",
           "Action": "iam:PassRole",
           "Condition": {
               "StringEquals": {
                   "iam:PassedToService": "ec2.amazonaws.com"
               }
           }
       },
       {
           "Sid": "AllowScopedInstanceProfileCreationActions",
           "Effect": "Allow",
           "Resource": "*",
           "Action": [
               "iam:CreateInstanceProfile"
           ],
           "Condition": {
               "StringEquals": {
                   "aws:RequestTag/kubernetes.io/cluster/${var.cluster_name}": "owned",
                   "aws:RequestTag/topology.kubernetes.io/region": "${var.region}"
               },
               "StringLike": {
                   "aws:RequestTag/karpenter.k8s.aws/ec2nodeclass": "*"
               }
           }
       },
       {
           "Sid": "AllowScopedInstanceProfileTagActions",
           "Effect": "Allow",
           "Resource": "*",
           "Action": [
               "iam:TagInstanceProfile"
           ],
           "Condition": {
               "StringEquals": {
                   "aws:ResourceTag/kubernetes.io/cluster/${var.cluster_name}": "owned",
                   "aws:ResourceTag/topology.kubernetes.io/region": "${var.region}",
                   "aws:RequestTag/kubernetes.io/cluster/${var.cluster_name}": "owned",
                   "aws:RequestTag/topology.kubernetes.io/region": "${var.region}"
               },
               "StringLike": {
                   "aws:ResourceTag/karpenter.k8s.aws/ec2nodeclass": "*",
                   "aws:RequestTag/karpenter.k8s.aws/ec2nodeclass": "*"
               }
           }
       },
       {
           "Sid": "AllowScopedInstanceProfileActions",
           "Effect": "Allow",
           "Resource": "*",
           "Action": [
               "iam:AddRoleToInstanceProfile",
               "iam:RemoveRoleFromInstanceProfile",
               "iam:DeleteInstanceProfile"
           ],
           "Condition": {
               "StringEquals": {
                   "aws:ResourceTag/kubernetes.io/cluster/${var.cluster_name}": "owned",
                   "aws:ResourceTag/topology.kubernetes.io/region": "${var.region}"
               },
               "StringLike": {
                   "aws:ResourceTag/karpenter.k8s.aws/ec2nodeclass": "*"
               }
           }
       },
       {
           "Sid": "AllowInstanceProfileReadActions",
           "Effect": "Allow",
           "Resource": "*",
           "Action": "iam:GetInstanceProfile"
       },
       {
           "Sid": "AllowAPIServerEndpointDiscovery",
           "Effect": "Allow",
           "Resource": "arn:${data.aws_partition.current.partition}:eks:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:cluster/${var.cluster_name}",
           "Action": "eks:DescribeCluster"
       }
   ]
}
```
**Tagging Resources:**
Tags for Security Groups and Subnets
To ensure Karpenter can manage worker nodes and attach the appropriate security groups, add the following tag to all relevant security groups and subnets:

Key: karpenter.sh/discovery
Value: ${CLUSTER_NAME}
Ensure this tag is applied to all subnets and security groups associated with your cluster.

**Installing Karpenter:**
Create Namespace
Create a namespace for Karpenter in your Kubernetes cluster. You can use the following command:


```console
kubectl create namespace karpenter
```
**Install Karpenter:**
Follow the official Karpenter installation guide to complete the installation process. Ensure you specify the created namespace as needed.

Configure values.yaml
Create a values.yaml file with the following configuration. This file will override default settings and define specific parameters for your Karpenter installation.

yaml
Copy code
```console
# -- Overrides the chart's name.
nameOverride: ""
# -- Overrides the chart's computed fullname.
fullnameOverride: ""
# -- Additional labels to add into metadata.
additionalLabels: {}
# app: karpenter

# -- Additional annotations to add into metadata.
additionalAnnotations: {}
# -- Image pull policy for Docker images.
imagePullPolicy: IfNotPresent
# -- Image pull secrets for Docker images.
imagePullSecrets: []
serviceAccount:
  # -- Specifies if a ServiceAccount should be created.
  create: true
  # -- The name of the ServiceAccount to use.
  # If not set and create is true, a name is generated using the fullname template.
  name: ""
  # -- Additional annotations for the ServiceAccount.
  annotations: {}
# -- Specifies additional rules for the core ClusterRole.
additionalClusterRoleRules: []
serviceMonitor:
  # -- Specifies whether a ServiceMonitor should be created.
  enabled: false
  # -- Additional labels for the ServiceMonitor.
  additionalLabels: {}
  # -- Configuration on `http-metrics` endpoint for the ServiceMonitor.
  # Not to be used to add additional endpoints.
  # See the Prometheus operator documentation for configurable fields https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/api.md#endpoint
  endpointConfig: {}
# -- Number of replicas.
replicas: 2
# -- The number of old ReplicaSets to retain to allow rollback.
revisionHistoryLimit: 10
# -- Strategy for updating the pod.
strategy:
  rollingUpdate:
    maxUnavailable: 1
# -- Additional labels for the pod.
podLabels: {}
# -- Additional annotations for the pod.
podAnnotations: {}
podDisruptionBudget:
  name: karpenter
  maxUnavailable: 1
# -- SecurityContext for the pod.
podSecurityContext:
  fsGroup: 65532
# -- PriorityClass name for the pod.
priorityClassName: system-cluster-critical
# -- Override the default termination grace period for the pod.
terminationGracePeriodSeconds:
# -- Bind the pod to the host network.
# This is required when using a custom CNI.
hostNetwork: false
# -- Configure the DNS Policy for the pod
dnsPolicy: ClusterFirst
# -- Configure DNS Config for the pod
dnsConfig: {}
#  options:
#    - name: ndots
#      value: "1"
# -- Node selectors to schedule the pod to nodes with labels.
nodeSelector:
  kubernetes.io/os: linux
# -- Affinity rules for scheduling the pod. If an explicit label selector is not provided for pod affinity or pod anti-affinity one will be created from the pod selector labels.
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: karpenter.sh/nodepool
              operator: DoesNotExist
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - topologyKey: "kubernetes.io/hostname"
# -- Topology spread constraints to increase the controller resilience by distributing pods across the cluster zones. If an explicit label selector is not provided one will be created from the pod selector labels.
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
# -- Tolerations to allow the pod to be scheduled to nodes with taints.
tolerations:
  - key: CriticalAddonsOnly
    operator: Exists
# -- Additional volumes for the pod.
extraVolumes: []
# - name: aws-iam-token
#   projected:
#     defaultMode: 420
#     sources:
#     - serviceAccountToken:
#         audience: sts.amazonaws.com
#         expirationSeconds: 86400
#         path: token
controller:
  image:
    # -- Repository path to the controller image.
    repository: public.ecr.aws/karpenter/controller
    # -- Tag of the controller image.
    tag: 0.37.0
    # -- SHA256 digest of the controller image.
    digest: sha256:157f478f5db1fe999f5e2d27badcc742bf51cc470508b3cebe78224d0947674f
  # -- Additional environment variables for the controller pod.
  env: []
  # - name: AWS_REGION
  #   value: eu-west-1
  envFrom: []
  # -- Resources for the controller pod.
  resources: {}
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  #  requests:
  #    cpu: 1
  #    memory: 1Gi
  #  limits:
  #    cpu: 1
  #    memory: 1Gi

  # -- Additional volumeMounts for the controller pod.
  extraVolumeMounts: []
  # - name: aws-iam-token
  #   mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
  #   readOnly: true
  # -- Additional sidecarContainer config
  sidecarContainer: []
  # -- Additional volumeMounts for the sidecar - this will be added to the volume mounts on top of extraVolumeMounts
  sidecarVolumeMounts: []
  metrics:
    # -- The container port to use for metrics.
    port: 8080
  healthProbe:
    # -- The container port to use for http health probe.
    port: 8081
webhook:
  # -- Whether to enable the webhooks and webhook permissions.
  enabled: true
  # -- The container port to use for the webhook.
  port: 8443
  metrics:
    # -- The container port to use for webhook metrics.
    port: 8001
# -- Global log level, defaults to 'info'
logLevel: info
# -- Log outputPaths - defaults to stdout only
logOutputPaths:
  - stdout
# -- Log errorOutputPaths - defaults to stderr only
logErrorOutputPaths:
  - stderr
# -- Global Settings to configure Karpenter
settings:
  # -- The maximum length of a batch window. The longer this is, the more pods we can consider for provisioning at one
  # time which usually results in fewer but larger nodes.
  batchMaxDuration: 10s
  # -- The maximum amount of time with no new ending pods that if exceeded ends the current batching window. If pods arrive
  # faster than this time, the batching window will be extended up to the maxDuration. If they arrive slower, the pods
  # will be batched separately.
  batchIdleDuration: 1s
  # -- Cluster CA bundle for TLS configuration of provisioned nodes. If not set, this is taken from the controller's TLS configuration for the API server.
  clusterCABundle: ""
  # -- Cluster name.
  clusterName: ""
  # -- Cluster endpoint. If not set, will be discovered during startup (EKS only)
  clusterEndpoint: ""
  # -- If true then assume we can't reach AWS services which don't have a VPC endpoint
  # This also has the effect of disabling look-ups to the AWS pricing endpoint
  isolatedVPC: false
  # -- The VM memory overhead as a percent that will be subtracted from the total memory for all instance types
  vmMemoryOverheadPercent: 0.075
  # -- Interruption queue is the name of the SQS queue used for processing interruption events from EC2
  # Interruption handling is disabled if not specified. Enabling interruption handling may
  # require additional permissions on the controller service account. Additional permissions are outlined in the docs.
  interruptionQueue: ""
  # -- Reserved ENIs are not included in the calculations for max-pods or kube-reserved
  # This is most often used in the VPC CNI custom networking setup https://docs.aws.amazon.com/eks/latest/userguide/cni-custom-network.html
  reservedENIs: "0"
  # -- Feature Gate configuration values. Feature Gates will follow the same graduation process and requirements as feature gates
  # in Kubernetes. More information here https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#feature-gates-for-alpha-or-beta-features
  featureGates:
    # -- spotToSpotConsolidation is ALPHA and is disabled by default.
    # Setting this to true will enable spot replacement consolidation for both single and multi-node consolidation.
    spotToSpotConsolidation: false
```
Update values.yaml File
Role ARN: Add the role ARN in the annotations section:


```console
annotations:
  eks.amazonaws.com/role-arn: "arn:aws:iam::<account-id>>:role/<role-name>"
```
Cluster Name and Interruption Queue: Update these values as per your requirements.

**Install Karpenter:**
Run one of the following commands to install Karpenter:


```console
helm upgrade --install --namespace karpenter --create-namespace -f values.yaml karpenter oci://public.ecr.aws/karpenter/karpenter --version v0.30.0-rc.0 --debug
```
Or, if you have specific values:


```console
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version "${KARPENTER_VERSION}" --namespace "${KARPENTER_NAMESPACE}" --create-namespace \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.interruptionQueue=${CLUSTER_NAME}" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait
```
**Verify Installation:**
Check the status of Karpenter pods:

```console
kubectl get pods -n karpenter
```
# **Create NodePool:**
Create a NodePool configuration file (e.g., nodepool.yaml) with the following content:

```console
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["2"]
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h # 30 * 24h = 720h
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2 # Amazon Linux 2
  role: "KarpenterNodeRole-${CLUSTER_NAME}" # replace with your cluster name
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}" # replace with your cluster name
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}" # replace with your cluster name
  amiSelectorTerms:
    - id: "${ARM_AMI_ID}"
    - id: "${AMD_AMI_ID}"
#   - id: "${GPU_AMI_ID}" # <- GPU Optimized AMD AMI 
#   - name: "amazon-eks-node-${K8S_VERSION}-*" # <- automatically upgrade when a new AL2 EKS Optimized AMI is released. This is unsafe for production workloads. Validate AMIs in lower environments before deploying them to production.
```
Make sure to replace placeholders such as ${CLUSTER_NAME}, ${ARM_AMI_ID}, ${AMD_AMI_ID}, etc., with appropriate values.

**Apply the NodePool configuration:**


```console
kubectl apply -f nodepool.yaml
```
This README provides comprehensive steps for installing Karpenter and configuring it for your Kubernetes cluster. Adjust any placeholders as needed for your specific environment and use cases.