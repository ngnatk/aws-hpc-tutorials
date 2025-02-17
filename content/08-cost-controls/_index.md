---
title: "Cost controls"
date: 2019-01-24T09:05:54Z
weight: 80
pre: "<b>VIII ⁃ </b>"
tags: ["HPC", "Introduction", "EC2", "Optional"]
---

{{% notice info %}}If you have not yet deployed a cluster, you can use the instructions for either the [ParallelCluster UI](02-deploy-pcm.html) or the [CLI](03-parallel-cluster-cli.html) to create one. Please note the cleanup instructions at the end and include cleanup of any accounting artefacts created within this chapter - particularly if working within your own account.
{{% /notice %}}

![hpc_logo](/images/hpc-aws-parallelcluster-workshop/aws-parallelclusterlogo.png)

In this lab, you will use Slurm Accounting within AWS ParallelCluster to create cost controls and monitor costs at the cluster level. 

This lab includes the following steps:

1. Configure Slurm Accounting and prerequisites
2. Create cost controls using Slurm Accounting resource limits
3. Test cost controls by running sample jobs
4. View cluster cost data in Amazon CloudWatch

### Design Overview

Cloud HPC resources have ephemeral components, such as compute nodes, that only incur cost to the user when the resources are running. Users of cloud HPC are able to leverage the cloud's variable-cost model to cost optimize their clusters. However, the cloud's "pay for what you use" cost model can present challenges for HPC user groups that are used to operating in a fixed-cost model. In this lab, you will implement a solution that tracks
the cluster's compute node costs and prevents jobs from running if a budget threshold is reached.

The cost control solution contains the following steps:

  1. Execute the cost conversion python script with the desired budget (in US dollars) as an input parameter.
  2. Convert the input budget to compute node CPU minutes using the AWS Price List API.
  3. Apply CPU minutes as a Slurm Accounting resource limit.
  4. Prevent Slurm job execution if the CPU minutes resource limit is reached.

Separately, you will publish cost metric data to Amazon CloudWatch to provide cost visibility in the ParallelCluster CloudWatch dashboard.

You will create the below architecture during this lab:

![Lab4 Architecture Diagram](/images/cost-controls/lab4_arch_diagram.png)

The architecture presented in this lab has the following components:
- **Slurm Accounting** collects accounting information for each job and job step executed. [Slurm Accounting](https://slurm.schedmd.com/accounting.html#Overview) tracks resources and allows users to apply resource limits through a configuration called Group Trackable
Resource Minutes (GrpTRESMins). In this lab, you will use Slurm Accounting to track compute node usage within ParallelCluster and translate usage to cost. 
- **Amazon Aurora** is a MySQL- and PostgreSQL-compatible database built for the cloud. If taking this lab as part of a workshop an [Amazon Aurora](https://aws.amazon.com/rds/aurora/) MySQL-compatible database has been pre-provisioned within your AWS sandbox account and associated with your Slurm instance via AWS ParallelCluster configuration for use with Slurm Accounting. If this is being done outside a workshop event, a database can be deployed in US East 1 Region by clicking on this button:
{{% button href="https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?stackName=pcluster-slurm-db&templateURL=https://us-east-1-aws-parallelcluster.s3.amazonaws.com/templates/1-click/serverless-database.yaml" icon="fas fa-rocket" %}}Deploy Amazon Aurora database{{% /button %}}
- **AWS Price List API** provides a catalog of the products and prices for AWS services. The [AWS Price List API](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/price-changes.html) is used in this lab to look up the dollar cost of EC2 compute node resources used by ParallelCluster.
- **Cost Conversion Python script** to define a cost limit in dollars, covert dollars to CPU minutes (CPUMins) using the AWS Price List API ,and apply the CPUMins value as a GrpTRESMins limit within Slurm Accounting. 
- **Cost Data CloudWatch Python script** to publish cluster cost data to the [Amazon CloudWatch](https://aws.amazon.com/cloudwatch/) service. This script extracts the total cluster compute usage from Slurm
Accounting, converts usage into dollars via Price List API, and publishes the cost as a custom metric to CloudWatch. 
- **Amazon CloudWatch** collects and visualizes real-time logs, metrics, and event data in dashboards to streamline your infrastructure and application maintenance.
In this lab, you will use the [Amazon CloudWatch ParallelCluster Dashboard](https://docs.aws.amazon.com/parallelcluster/latest/ug/cloudwatch-dashboard-v3.html) that is created during ParallelCluster cluster creation.

### Considerations and Limitations
The solution presented in this lab demonstrates the art of the possible for AWS ParallelCluster cost controls.
This solution, provided as-is under the MIT license, is **not intended for use in production environments**.
Those intending to use the concepts presented in this lab in their own environments are encouraged to build upon the capabilities
demonstrated in this lab to meet their own requirements.

Capabilities that are not currently supported:
- multiple Slurm partitions (multiple ParallelCluster compute queues)
- varied compute pricing models: Spot, On-Demand, Reserved Instances, Savings Plans
- incorporating "always-on" costs including, but not limited to: head node, shared storage, networking
- handling partially used or idle compute nodes
- time-based (monthly, weekly, etc.) budgets


### Documentation/Links
Some supplemental documentation is available in the AWS Docs that were used to build and develop this solution:
- [Slurm accounting with AWS ParallelCluster](https://docs.aws.amazon.com/parallelcluster/latest/ug/slurm-accounting-v3.html) guides users through the processes to carry out SLURM accounting; including the changes brought with AWS ParallelCluster 3.10.0 where support was included for accounting with an external Slurmdbd.
- [Creating a cluster with an external Slurmdbd accounting](https://docs.aws.amazon.com/parallelcluster/latest/ug/external-slurmdb-accounting.html) is documentation detailing the external Slurmdbd approach. This includes a CloudFormation process to create a Slurmdbd stack.
- [AWS ParallelCluster tutorial for Creating a cluster with Slurm accounting](https://docs.aws.amazon.com/parallelcluster/latest/ug/tutorials_07_slurm-accounting-v3.html)