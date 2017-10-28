# Kibana cluster on Ubuntu
This is a tutorial on how to deploy an autoscaling Kibana cluster on Ubuntu in AWS using CloudFormation.

The CloudFormation template and explanation is also posted on the [NETBEARS](https://netbears.com/blog/kibana-cluster-ubuntu/) company blog. You might want to check the website out for more tutorials like this.

## Prior to deployment notes

This CloudFormation stack assumes that you already have an ElasticSearch cluster deployed in your infrastructure in which Zen discovery via security group is already in place.

If you don't have one already, then all you have to do is deploy it using our previous tutorial -> [Deploy ElasticSearch Cluster on Ubuntu using AWS CloudFormation](https://netbears.com/blog/elasticsearch-cluster-ubuntu/).

Reason for this is that this template uses the security group of the auto-discovery ElasticSearch cluster to create a proxy node (with no-data) that the Kibana service uses in order to limit the load on the actual ElasticSearch cluster.

This method ensures that your Kibana service is able to talk properly with all master and non-master nodes in your ElasticSearch cluster and can hence scale easily if your user adoption for this service is high (aka high traffic).

## Run the CloudFormation template with AWS CLI

```
git clone https://github.com/NETBEARS/kibana-cluster-ubuntu.git

aws cloudformation create-stack \
  --stack-name kibana-cluster-ubuntu \
  --template-body file://cloudformation-template.yaml \
  --parameters \
    ParameterKey=Ami,ParameterValue=ami-6e1a0117 \
    ParameterKey=AsgMaxSize,ParameterValue=8 \
    ParameterKey=AsgMinSize,ParameterValue=1 \
    ParameterKey=EmailAlerts,ParameterValue=email_for_alerts@domain.com \
    ParameterKey=InstanceType,ParameterValue=m4.large \
    ParameterKey=KeyName,ParameterValue=YOUR_INSTANCE_KEY \
    ParameterKey=VpcId,ParameterValue=VPC_ID \
    ParameterKey=SubnetID1,ParameterValue=SUBNET_IN_VPC_ID_1 \
    ParameterKey=SubnetID2,ParameterValue=SUBNET_IN_VPC_ID_2 \
  --capabilities CAPABILITY_IAM

```

## Run the CloudFormation template in the AWS Console
* Login to the AWS console and browse to the CloudFormation section
* Select the "cloudformation-template.yaml” file
* Before clicking "Create", make sure that you scroll down and tick the “I acknowledge that AWS CloudFormation might create IAM resources” checkbox
* ...drink coffee...
* Go to the URL in the output section for the environment that you want to access

## Resources created
* 1 AutoScaling Group
* 1 Elastic Load Balancer
* 1 S3 bucket (for data backup)
* 1 SNS topic (send monitoring alerts)

## Autoscaling
The autoscaling groups uses the CpuUtilization alarm to autoscale automatically.

Because of this, you wouldn't have to bother making sure that your hosts can sustain the load.

## Alarms
In order to be sure that you have set up the proper limits for your containers, the following alerts have been but into place:
* NetworkInAlarm
* RAMAlarmHigh
* NetworkOutAlarm
* IOWaitAlarmHigh
* StatusAlarm
  
These CloudWatch alarms will send an email each time the limits are hit so that you will always be in control of what happens with your stack.

## Monitoring
The stack launches [NodeExporter](https://github.com/prometheus/node_exporter) <> `Prometheus exporter for hardware and OS metrics exposed by *NIX kernels, written in Go with pluggable metric collectors`, on each host inside the cluster.

To view the monitoring data, all you need to setup is a Prometheus host and a Grafana dashboard and you're all set.

## Data persistency
Due to the mechanics behind Kibana, you don't need to set up any sort of data persistency, as the application queries continously the ElasticSearch cluster in order to acquire the data that is being displayed. Hence, the data persistency that we're currently handling is the actual configuration of Kibana, which is stored for backup and reference purposes on an attached EBS volume.
      
## Backup

A cronjob has been set up to run every 3 days on the ASG hosts that dump the data in an S3 bucket that is created inside the template.
        
## Final notes
Need help implementing this?

Feel free to contact us using [this form](https://netbears.com/#contact-form).
