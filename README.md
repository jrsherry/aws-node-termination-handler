<h1>AWS Node Termination Handler</h1>

<h4>Gracefully handle EC2 instance shutdown within Kubernetes</h4>

<p>
  <a href="https://github.com/kubernetes/kubernetes/releases">
    <img src="https://img.shields.io/badge/Kubernetes-%3E%3D%201.16-brightgreen" alt="kubernetes">
  </a>
  <a href="https://golang.org/doc/go1.16">
    <img src="https://img.shields.io/github/go-mod/go-version/aws/aws-node-termination-handler?color=blueviolet" alt="go-version">
  </a>
  <a href="https://opensource.org/licenses/Apache-2.0">
    <img src="https://img.shields.io/badge/License-Apache%202.0-ff69b4.svg" alt="license">
  </a>
  <a href="https://codecov.io/gh/aws/aws-node-termination-handler">
    <img src="https://img.shields.io/codecov/c/github/aws/aws-node-termination-handler" alt="build-status">
  </a>
  <a href="https://gallery.ecr.aws/aws-ec2/aws-node-termination-handler">
    <img src="https://img.shields.io/docker/pulls/amazon/aws-node-termination-handler" alt="docker-pulls">
  </a>
</p>

![NTH Continuous Integration and Release](https://github.com/aws/aws-node-termination-handler/workflows/NTH%20Continuous%20Integration%20and%20Release/badge.svg)

<div>
<hr>
</div>

## Project Summary

This project ensures that the Kubernetes control plane responds appropriately to events that can cause your EC2 instance to become unavailable, such as [EC2 maintenance events](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-instances-status-check_sched.html), [EC2 Spot interruptions](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-interruptions.html), [ASG Scale-In](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroupLifecycle.html#as-lifecycle-scale-in), [ASG AZ Rebalance](https://docs.aws.amazon.com/autoscaling/ec2/userguide/auto-scaling-benefits.html#AutoScalingBehavior.InstanceUsage), and EC2 Instance Termination via the API or Console.  If not handled, your application code may not stop gracefully, take longer to recover full availability, or accidentally schedule work to nodes that are going down.

The aws-node-termination-handler (NTH) can operate in two different modes: Instance Metadata Service (IMDS) or the Queue Processor.

The aws-node-termination-handler **[Instance Metadata Service](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html) Monitor** will run a small pod on each host to perform monitoring of IMDS paths like `/spot` or `/events` and react accordingly to drain and/or cordon the corresponding node.

The aws-node-termination-handler **Queue Processor** will monitor an SQS queue of events from Amazon EventBridge for ASG lifecycle events, EC2 status change events, Spot Interruption Termination Notice events, and Spot Rebalance Recommendation events. When NTH detects an instance is going down, we use the Kubernetes API to cordon the node to ensure no new work is scheduled there, then drain it, removing any existing work. The termination handler **Queue Processor** requires AWS IAM permissions to monitor and manage the SQS queue and to query the EC2 API.

You can run the termination handler on any Kubernetes cluster running on AWS, including self-managed clusters and those created with Amazon [Elastic Kubernetes Service](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html).

## Major Features

### Instance Metadata Service Processor
- Monitors EC2 Metadata for Scheduled Maintenance Events
- Monitors EC2 Metadata for Spot Instance Termination Notifications
- Monitors EC2 Metadata for Rebalance Recommendation Notifications
- Helm installation and event configuration support
- Webhook feature to send shutdown or restart notification messages
- Unit & Integration Tests

### Queue Processor
- Monitors an SQS Queue for:
   - EC2 Spot Interruption Notifications
   - EC2 Instance Rebalance Recommendation
   - EC2 Auto-Scaling Group Termination Lifecycle Hooks to take care of ASG Scale-In, AZ-Rebalance, Unhealthy Instances, and more!
   - EC2 Status Change Events
- Helm installation and event configuration support
- Webhook feature to send shutdown or restart notification messages
- Unit & Integration Tests

## Which one should I use?
Feature |IMDS Processor | Queue Processor
:---:|:---:|:---:
K8s DaemonSet | ✅ | ❌
K8s Deployment | ❌ | ✅
Spot Instance Interruptions (ITN) | ✅ | ✅
Scheduled Events | ✅ | ✅
EC2 Instance Rebalance Recommendation | ✅ | ✅
ASG Lifecycle Hooks | ❌ | ✅
EC2 Status Changes | ❌ | ✅
Setup Required | ❌ | ✅


## Installation and Configuration

The aws-node-termination-handler can operate in two different modes: IMDS Processor and Queue Processor. The `enableSqsTerminationDraining` helm configuration key or the `ENABLE_SQS_TERMINATION_DRAINING` environment variable are used to enable the Queue Processor mode of operation. If `enableSqsTerminationDraining` is set to true, then IMDS paths will NOT be monitored. If the `enableSqsTerminationDraining` is set to false, then IMDS Processor Mode will be enabled. Queue Processor Mode and IMDS Processor Mode cannot be run at the same time.

IMDS Processor Mode allows for a fine-grained configuration of IMDS paths that are monitored. There are currently 3 paths supported that can be enabled or disabled by using the following helm configuration keys:
 - `enableSpotInterruptionDraining`
 - `enableRebalanceMonitoring`
 - `enableScheduledEventDraining`

By default, IMDS mode will only Cordon in response to a Rebalance Recommendation event (all other events are Cordoned and Drained). Cordon is the default for a rebalance event because it's not known if an ASG is being utilized and if that ASG is configured to replace the instance on a rebalance event. If you are using an ASG w/ rebalance recommendations enabled, then you can set the `enableRebalanceDraining` flag to true to perform a Cordon and Drain when a rebalance event is received.

The `enableSqsTerminationDraining` must be set to false for these configuration values to be considered.

The Queue Processor Mode does not allow for fine-grained configuration of which events are handled through helm configuration keys. Instead, you can modify your Amazon EventBridge rules to not send certain types of events to the SQS Queue so that NTH does not process those events. All events when operating in Queue Processor mode are Cordoned and Drained unless the `cordon-only` flag is set to true.


The `enableSqsTerminationDraining` flag turns on Queue Processor Mode. When Queue Processor Mode is enabled, IMDS mode cannot be active. NTH cannot respond to queue events AND monitor IMDS paths. Queue Processor Mode still queries for node information on startup, but this information is not required for normal operation, so it is safe to disable IMDS for the NTH pod.

<details opened>
<summary>AWS Node Termination Handler - IMDS Processor</summary>
<br>

### Installation and Configuration

The termination handler DaemonSet installs into your cluster a [ServiceAccount](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/), [ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/), [ClusterRoleBinding](https://kubernetes.io/docs/reference/access-authn-authz/rbac/), and a [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/). All four of these Kubernetes constructs are required for the termination handler to run properly.


#### Kubectl Apply

You can use kubectl to directly add all of the above resources with the default configuration into your cluster.

```
kubectl apply -f https://github.com/aws/aws-node-termination-handler/releases/download/v1.13.3/all-resources.yaml
```

For a full list of releases and associated artifacts see our [releases page](https://github.com/aws/aws-node-termination-handler/releases).

#### Helm

The easiest way to configure the various options of the termination handler is via [helm](https://helm.sh/).  The chart for this project is hosted in the [eks-charts](https://github.com/aws/eks-charts) repository.

To get started you need to add the eks-charts repo to helm

```
helm repo add eks https://aws.github.io/eks-charts
```

Once that is complete you can install the termination handler. We've provided some sample setup options below.

Zero Config:

```sh
helm upgrade --install aws-node-termination-handler \
  --namespace kube-system \
  eks/aws-node-termination-handler
```

Enabling Features:

```
helm upgrade --install aws-node-termination-handler \
  --namespace kube-system \
  --set enableSpotInterruptionDraining="true" \
  --set enableRebalanceMonitoring="true" \
  --set enableScheduledEventDraining="false" \
  eks/aws-node-termination-handler
```

The `enable*` configuration flags above enable or disable IMDS monitoring paths.

Running Only On Specific Nodes:

```
helm upgrade --install aws-node-termination-handler \
  --namespace kube-system \
  --set nodeSelector.lifecycle=spot \
  eks/aws-node-termination-handler
```

Webhook Configuration:

```
helm upgrade --install aws-node-termination-handler \
  --namespace kube-system \
  --set webhookURL=https://hooks.slack.com/services/YOUR/SLACK/URL \
  eks/aws-node-termination-handler
```

Alternatively, pass Webhook URL as a Secret:

```
WEBHOOKURL_LITERAL="webhookurl=https://hooks.slack.com/services/YOUR/SLACK/URL"

kubectl create secret -n kube-system generic webhooksecret --from-literal=$WEBHOOKURL_LITERAL
```
```
helm upgrade --install aws-node-termination-handler \
  --namespace kube-system \
  --set webhookURLSecretName=webhooksecret \
  eks/aws-node-termination-handler
```

For a full list of configuration options see our [Helm readme](https://github.com/aws/eks-charts/tree/master/stable/aws-node-termination-handler).

</details>


<details closed>
<summary>AWS Node Termination Handler - Queue Processor (requires AWS IAM Permissions)</summary>

<br>

### Infrastructure Setup

The termination handler deployment requires some infrastructure to be setup before deploying the application. You'll need the following AWS infrastructure components:

1. AutoScaling Group Termination Lifecycle Hook
2. Amazon Simple Queue Service (SQS) Queue
3. Amazon EventBridge Rule
4. IAM Role for the aws-node-termination-handler Queue Processing Pods

#### 1. Setup a Termination Lifecycle Hook on an ASG:

Here is the AWS CLI command to create a termination lifecycle hook on an existing ASG, although this should really be configured via your favorite infrastructure-as-code tool like CloudFormation or Terraform:

```
$ aws autoscaling put-lifecycle-hook \
  --lifecycle-hook-name=my-k8s-term-hook \
  --auto-scaling-group-name=my-k8s-asg \
  --lifecycle-transition=autoscaling:EC2_INSTANCE_TERMINATING \
  --default-result=CONTINUE \
  --heartbeat-timeout=300
```

#### 2. Tag the ASGs:

By default the aws-node-termination-handler will only manage terminations for ASGs tagged w/ `key=aws-node-termination-handler/managed`

```
$ aws autoscaling create-or-update-tags \
  --tags ResourceId=my-auto-scaling-group,ResourceType=auto-scaling-group,Key=aws-node-termination-handler/managed,Value=,PropagateAtLaunch=true
```

The value of the key does not matter.

This functionality is helpful in accounts where there are ASGs that do not run kubernetes nodes or you do not want aws-node-termination-handler to manage their termination lifecycle.
However, if your account is dedicated to ASGs for your kubernetes cluster, then you can turn off the ASG tag check by setting the flag `--check-asg-tag-before-draining=false` or environment variable `CHECK_ASG_TAG_BEFORE_DRAINING=false`.

You can also control what resources NTH manages by adding the resource ARNs to your Amazon EventBridge rules.

Take a look at the docs on how to create rules that only manage certain ASGs [here](https://docs.aws.amazon.com/autoscaling/ec2/userguide/cloud-watch-events.html).

See all the different events docs [here](https://docs.aws.amazon.com/eventbridge/latest/userguide/event-types.html#auto-scaling-event-types).

#### 3. Create an SQS Queue:

Here is the AWS CLI command to create an SQS queue to hold termination events from ASG and EC2, although this should really be configured via your favorite infrastructure-as-code tool like CloudFormation or Terraform:

```
## Queue Policy
$ QUEUE_POLICY=$(cat <<EOF
{
    "Version": "2012-10-17",
    "Id": "MyQueuePolicy",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {
            "Service": ["events.amazonaws.com", "sqs.amazonaws.com"]
        },
        "Action": "sqs:SendMessage",
        "Resource": [
            "arn:aws:sqs:${AWS_REGION}:${ACCOUNT_ID}:${SQS_QUEUE_NAME}"
        ]
    }]
}
EOF
)

## make sure the queue policy is valid JSON
$ echo "$QUEUE_POLICY" | jq .

## Save queue attributes to a temp file
$ cat << EOF > /tmp/queue-attributes.json
{
  "MessageRetentionPeriod": "300",
  "Policy": "$(echo $QUEUE_POLICY | sed 's/\"/\\"/g' | tr -d -s '\n' " ")"
}
EOF

$ aws sqs create-queue --queue-name "${SQS_QUEUE_NAME}" --attributes file:///tmp/queue-attributes.json
```

#### 4. Create Amazon EventBridge Rules

Here are AWS CLI commands to create Amazon EventBridge rules so that ASG termination events, Spot Interruptions, Instance state changes and Rebalance Recommendations are sent to the SQS queue created in the previous step. This should really be configured via your favorite infrastructure-as-code tool like CloudFormation or Terraform:

```
$ aws events put-rule \
  --name MyK8sASGTermRule \
  --event-pattern "{\"source\":[\"aws.autoscaling\"],\"detail-type\":[\"EC2 Instance-terminate Lifecycle Action\"]}"

$ aws events put-targets --rule MyK8sASGTermRule \
  --targets "Id"="1","Arn"="arn:aws:sqs:us-east-1:123456789012:MyK8sTermQueue"

$ aws events put-rule \
  --name MyK8sSpotTermRule \
  --event-pattern "{\"source\": [\"aws.ec2\"],\"detail-type\": [\"EC2 Spot Instance Interruption Warning\"]}"

$ aws events put-targets --rule MyK8sSpotTermRule \
  --targets "Id"="1","Arn"="arn:aws:sqs:us-east-1:123456789012:MyK8sTermQueue"

$ aws events put-rule \
  --name MyK8sRebalanceRule \
  --event-pattern "{\"source\": [\"aws.ec2\"],\"detail-type\": [\"EC2 Instance Rebalance Recommendation\"]}"

$ aws events put-targets --rule MyK8sRebalanceRule \
  --targets "Id"="1","Arn"="arn:aws:sqs:us-east-1:123456789012:MyK8sTermQueue"

$ aws events put-rule \
  --name MyK8sInstanceStateChangeRule \
  --event-pattern "{\"source\": [\"aws.ec2\"],\"detail-type\": [\"EC2 Instance State-change Notification\"]}"

$ aws events put-targets --rule MyK8sInstanceStateChangeRule \
  --targets "Id"="1","Arn"="arn:aws:sqs:us-east-1:123456789012:MyK8sTermQueue"
```

#### 5. Create an IAM Role for the Pods

There are many different ways to allow the aws-node-termination-handler pods to assume a role:

1. [Amazon EKS IAM Roles for Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
2. [IAM Instance Profiles for EC2](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html)
3. [Kiam](https://github.com/uswitch/kiam)
4. [kube2iam](https://github.com/jtblin/kube2iam)

IAM Policy for aws-node-termination-handler Deployment:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "autoscaling:CompleteLifecycleAction",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeTags",
                "ec2:DescribeInstances",
                "sqs:DeleteMessage",
                "sqs:ReceiveMessage"
            ],
            "Resource": "*"
        }
    ]
}
```

### Installation

#### Helm

The easiest and most commonly used method to configure the termination handler is via [helm](https://helm.sh/).  The chart for this project is hosted in the [eks-charts](https://github.com/aws/eks-charts) repository.

To get started you need to add the eks-charts repo to helm

```
helm repo add eks https://aws.github.io/eks-charts
```

Once that is complete you can install the termination handler. We've provided some sample setup options below.

Minimal Config:

```sh
helm upgrade --install aws-node-termination-handler \
  --namespace kube-system \
  --set enableSqsTerminationDraining=true \
  --set queueURL=https://sqs.us-east-1.amazonaws.com/0123456789/my-term-queue \
  eks/aws-node-termination-handler
```

Webhook Configuration:

```
helm upgrade --install aws-node-termination-handler \
  --namespace kube-system \
  --set enableSqsTerminationDraining=true \
  --set queueURL=https://sqs.us-east-1.amazonaws.com/0123456789/my-term-queue \
  --set webhookURL=https://hooks.slack.com/services/YOUR/SLACK/URL \
  eks/aws-node-termination-handler
```

Alternatively, pass Webhook URL as a Secret:

```
WEBHOOKURL_LITERAL="webhookurl=https://hooks.slack.com/services/YOUR/SLACK/URL"

kubectl create secret -n kube-system generic webhooksecret --from-literal=$WEBHOOKURL_LITERAL
```
```
helm upgrade --install aws-node-termination-handler \
  --namespace kube-system \
  --set enableSqsTerminationDraining=true \
  --set queueURL=https://sqs.us-east-1.amazonaws.com/0123456789/my-term-queue \
  --set webhookURLSecretName=webhooksecret \
  eks/aws-node-termination-handler
```

For a full list of configuration options see our [Helm readme](https://github.com/aws/eks-charts/tree/master/stable/aws-node-termination-handler).

#### Kubectl Apply

Queue Processor needs an **sqs queue url** to function; therefore, manifest changes are **REQUIRED** before using kubectl to directly add all of the above resources into your cluster.

Minimal Config:

```
curl -L https://github.com/aws/aws-node-termination-handler/releases/download/v1.13.3/all-resources-queue-processor.yaml -o all-resources-queue-processor.yaml
<open all-resources-queue-processor.yaml and update QUEUE_URL value>
kubectl apply -f ./all-resources-queue-processor.yaml
```

For a full list of releases and associated artifacts see our [releases page](https://github.com/aws/aws-node-termination-handler/releases).

</details>


<details close>
<summary>Use with Kiam</summary>
<br>

## Use with Kiam

If you are using IMDS mode which defaults to `hostNetworking: true`, or if you are using queue-processor mode, then this section does not apply. The configuration below only needs to be used if you are explicitly changing NTH IMDS mode to `hostNetworking: false` .

To use the termination handler alongside [Kiam](https://github.com/uswitch/kiam) requires some extra configuration on Kiam's end.
By default Kiam will block all access to the metadata address, so you need to make sure it passes through the requests the termination handler relies on.

To add a whitelist configuration, use the following fields in the Kiam Helm chart values:

```
agent.whiteListRouteRegexp: '^\/latest\/meta-data\/(spot\/instance-action|events\/maintenance\/scheduled|instance-(id|type)|public-(hostname|ipv4)|local-(hostname|ipv4)|placement\/availability-zone)|\/latest\/dynamic\/instance-identity\/document$'
```
Or just pass it as an argument to the kiam agents:

```
kiam agent --whitelist-route-regexp='^\/latest\/meta-data\/(spot\/instance-action|events\/maintenance\/scheduled|instance-(id|type)|public-(hostname|ipv4)|local-(hostname|ipv4)|placement\/availability-zone)|\/latest\/dynamic\/instance-identity\/document$'
```

## Metadata endpoints
The termination handler relies on the following metadata endpoints to function properly:

```
/latest/dynamic/instance-identity/document
/latest/meta-data/spot/instance-action
/latest/meta-data/events/recommendations/rebalance
/latest/meta-data/events/maintenance/scheduled
/latest/meta-data/instance-id
/latest/meta-data/instance-life-cycle
/latest/meta-data/instance-type
/latest/meta-data/public-hostname
/latest/meta-data/public-ipv4
/latest/meta-data/local-hostname
/latest/meta-data/local-ipv4
/latest/meta-data/placement/availability-zone
```

</details>

## Building
For build instructions please consult [BUILD.md](./BUILD.md).

## Communication
* If you've run into a bug or have a new feature request, please open an [issue](https://github.com/aws/aws-node-termination-handler/issues/new).
* You can also chat with us in the [Kubernetes Slack](https://kubernetes.slack.com) in the `#provider-aws` channel
* Check out the open source [Amazon EC2 Spot Instances Integrations Roadmap](https://github.com/aws/ec2-spot-instances-integrations-roadmap) to see what we're working on and give us feedback!

##  Contributing
Contributions are welcome! Please read our [guidelines](https://github.com/aws/aws-node-termination-handler/blob/main/CONTRIBUTING.md) and our [Code of Conduct](https://github.com/aws/aws-node-termination-handler/blob/main/CODE_OF_CONDUCT.md)

## License
This project is licensed under the Apache-2.0 License.

