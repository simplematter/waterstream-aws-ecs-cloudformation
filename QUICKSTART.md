Waterstream Quickstart with AWS ECS CloudFormation scripts
==========================================================

Take the following steps to deploy Kafka (optionally) and Waterstream to your AWS account on Elastic Cloud Service.
There are 3 CloudFormation templates for this:

- `commons_template.yml` creates basic network infrastructure 
- `kafka_template.yml` starts Kafka cluster in AWS MSK (if you don't already have Kafka) 
- `waterstream_template.yml` actually starts Waterstream and its auxilary infrastructure (testbox, monitoring).

![Waterstream AWS ECS architecture diagram](img/waterstream_aws_architecture.png "Waterstream AWS ECS architecture diagram")

Subscribe to Waterstream on AWS Marketplace 
-------------------------------------------

Go to [Waterstream Container product page](https://aws.amazon.com/marketplace/pp/B08ZDMBQY5) in AWS marketplace
and subscribe to it.

Create DockerHub secret
-----------------------

You're going to need a [DockerHub](https://hub.docker.com/) account so that you wouldn't hit the pull
limits when downloading auxiliary containers (e.g. Prometheus and Grafana). Free account is sufficient.
Details on configuring DockerHub credentials in AWS are described here: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/private-auth.html#private-auth-enable
In short words, you'll need to go to the [Secrets Manager](https://console.aws.amazon.com/secretsmanager/),
create a new secret of "Other type of secrets" type and plaintext content like this:

    {
      "username" : "yourDockerHubUsername",
      "password" : "yourDockerHubPassword"
    }

Then write down the name you've given to the secret - you'll need it later.

Make sure you have the necessary permissions 
--------------------------------------------

The user that runs the CloudFormation templates should have the following policies: 

- AWS-managed `arn:aws:iam::aws:policy/AmazonMSKFullAccess` (if you're going to deploy MSK Kafka)
- AWS-managed `arn:aws:iam::aws:policy/AmazonECS_FullAccess`
- Custom policy that grants the necessary access for managing VPC, IAM roles, CloudWatch logs, testbox EC2 instance, etc.
can be found in [WaterstreamCfEcsDeploy.json](WaterstreamCfEcsDeploy.json). 
You can [add this policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_manage-attach-detach.html#add-policies-console)
directly to the user that is going to deploy Waterstream (as inline policy), or 
[create the policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_create-console.html)
first and then attach it.
- Custom policy that grants access to the DockerHub credentials secret. Like the previous one, it can be in-line 
  (directly in the user) or created separately and then attached.
  Here's an example:

      {
        "Version": "2012-10-17",
        "Statement": [{
          "Sid": "VisualEditor0",
          "Effect": "Allow",
          "Action": [
            "secretsmanager:GetSecretValue",
            "secretsmanager:DescribeSecret"
          ],
          "Resource": "arn:aws:secretsmanager:*:<your AWS account>:secret:<your secret>"
        }]
      }

![User policies](img/user_policies.png "User policies")

Deploy Commons
----------------

[Create VPC and subnets](https://console.aws.amazon.com/cloudformation/home#/stacks/quickcreate?templateUrl=https://waterstream-public-resources.s3.eu-central-1.amazonaws.com/cloud_formation_ecs/v1/templates/commons_template.yml).
You may customize the subnet CIDRs.

![Commons stack create](img/ws_commons_create.png "Commons stack create")

Launch Kafka
------------

If you already have a Kafka cluster and you want to use it for Waterstream you can skip this step.
For example, you may [use ConfluentCloud as a KafkaCluster](#running-with-ConfluentCloud)

If you want a new Kafka cluster specially for Waterstream,
you can [launch AWS MSK cluster](https://console.aws.amazon.com/cloudformation/home#/stacks/quickcreate?templateUrl=https://waterstream-public-resources.s3.eu-central-1.amazonaws.com/cloud_formation_ecs/v1/templates/kafka_template.yml)

![Kafka stack create](img/ws_kafka_create.png "Kafka stack create")

Check the cluster creation in [AWS MSK console](https://eu-central-1.console.aws.amazon.com/msk/home#/clusters).
When its creation is complete - write down bootstrap servers, you'll need it in the next step.
You can get bootstrap servers by clicking "View client information" in "Cluster summary" page
or by running the following command (if you have AWS CLI configured on your machine):

    aws --profile <your AWS CLI profile> kafka get-bootstrap-brokers --cluster-arn <your MSK cluster ARN> | jq -r '.BootstrapBrokerString'

![Kafka cluster status](img/kafka_status_focus.png "Kafka cluster status")
![Kafka client properties](img/kafka_client_props_focus.png "Kafka client properties")

Make sure you have a EC2 keypair for the testbox
------------------------------------------------

Testbox is an auxiliary EC2 instance which is used to create Kafka topics, issue SSL/TLS certificates
and run test scripts. You'll need [EC2 key pair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)
to be able to log into it. Private key remains on your machine, and public key is shared with AWS EC2 infrastructure
so that it could grant you an access to the EC2 instance.
Create a new key pair in [EC2 console](https://console.aws.amazon.com/ec2/) or make sure you can use an existing one.

Launch Waterstream
------------------

Finally, you can [launch the Waterstream](https://console.aws.amazon.com/cloudformation/home#/stacks/quickcreate?templateUrl=https://waterstream-public-resources.s3.eu-central-1.amazonaws.com/cloud_formation_ecs/v1/templates/waterstream_template.yml)
Most parameters can be left default. The only parameters you have to specify yourselves are:

- DockerHub credentials secret name (from the previous step)
- Kafka bootstrap servers (from the "Launch Kafka" step or from pre-existing Kafka cluster)
- Testbox EC2 keypair - pick the keypair from the dropdown that you'd like to use for logging into the testbox
  (machine with auxiliary test scripts and SSL/TLS ad-hoc CA)

With the default options SSL/TLS and client authentication are disabled.
You can enable them with `WaterstreamEnableSsl` and `WaterstreamRequireAuthentication` parameters
respecitvely.

![Waterstream stack create 1](img/ws_waterstream_create_1_focus.png "Waterstream stack create 1")

![Waterstream stack create 2](img/ws_waterstream_create_2_focus.png "Waterstream stack create 2")

When the stack creation is complete, you can see in the "Output" tab the following outputs:

- `WaterstreamTestboxHostname` - hostname of the testbox. You can ssh into that machine and do some tests
- `WaterstreamLbHostname` - MQTT load balancer host name. MQTT clients may connect here, to the port 1883
- `WaterstreamGrafanaLbHostname` - monitoring system (Grafana) hostname. Open in your browser this host with port 3000 

![Waterstream stack outputs](img/ws_waterstream_outputs.png "Waterstream stack outputs")

Test it
-------

Open in your browser `<WaterstreamGrafanaLbHostname>:3000`, where `WaterstreamGrafanaLbHostname` is taken from the
Waterstream stack output. Default credentials are "admin/admin", you'll be prompted to change it
first time you log in.
In the left panel click "Dashboards/Manage", then click "Waterstream" dashboard to open it.
You should see some data in the memory graph, but no actual MQTT stats yet because there were no MQTT connections. 

![Grafana dashboard manage](img/grafana_dashboard_manage.png "Grafana: Dashboard/Manage")

![Grafana Waterstream dashboard](img/waterstream_dashboard.png "Grafana: Waterstream dashboard")

To test it using readily available MQTT clients in the testbox - log into it:

    ssh -i <path to identity file> ec2-user@<value from WaterstreamTestboxHostname>

If you're running plain-text Waterstream (i.e. no SSL/TLS), run `plain/mqtt_receive_sample.sh` in one terminal
and `plain/mqtt_send_sample.sh` in another. You should see the sample message in the receiving side.
If you're runnning with SSL/TLS - use `tls/mqtt_receive_sample.sh` and `tls/mqtt_send_sample.sh` instead.
In the dashboard you'll see the number of sent/received messages going up.

![Testbox pub](img/testbox_pub_terminal.png "Testbox pub")
![Testbox sub](img/testbox_sub_terminal.png "Testbox sub")

You can test MQTT connection from any MQTT client on you machine (connecting to `WaterstreamLbHostname:1883`)
or with the testbox. For example, if you have Mosquitto MQTT clients installed locally you can subscribe in one terminal:

    mosquitto_sub -h <value from WaterstreamLbHostname> -p 1883 -t "#" -i mosquitto_l_p1 -q 0 -v

and publish in another:

    mosquitto_pub -h <value from WaterstreamLbHostname> -p 1883 -t "sample_topic" -i mosquitto_l_p2 -q 0 -m "Hello, world!"


![Local pub](img/local_pub_terminal.png "Local pub")
![Local sub](img/local_sub_terminal.png "Local sub")

Undeploy
--------

First delete Waterstream and Kafka stacks. When they're deleted - also delete commons stack.

Running with ConfluentCloud
---------------------------

You can run Waterstream with these scripts using ConfluentCloud as Kafka provider.
In this case you don't need to [create Kafka CloudFormation stack](#launch-kafka).

Here's how you can prepare to it: 

- An account at https://confluent.cloud 
- Create a cluster in ConfluentCloud, write down its ID
- Set up [Confluent CLI](https://docs.confluent.io/confluent-cli/current/install.html#cli-install) locally
- Create topics: `./createTopicsCCloudMinimal.sh <cluster ID>` or `./createTopicsCCloudBig.sh <cluster ID>`.
  You can list clusters with `confluent kafka cluster list` and topics with `confluent kafka topic list --cluster <cluster ID>` 

When [creating the Waterstream CloudFormation stack](#launch-waterstream)
you'll need to customize the following parameters 
(you'll find values for most of them in ConfluentCloud UI Cluster overview/Configure a client):

- `KafkaBootstrapServers`
- `KafkaSslEndpointIdentificationAlgorithm`
- `KafkaSaslJaasConfig`
- `KafkaSaslMechanism`
- `KafkaSecurityProtocol`

After undeploying the Waterstream be sure to delete the ConfluentCloud topics: `./deleteTopicsCCloud.sh <cluster ID>`,
otherwise you'll be charged for the reserved capacity by ConfluentCloud.

Support
-------

You can get the support on [Waterstream Dev forum](https://dev.waterstream.io/).
Feel free to write there if you have any questions.