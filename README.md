# CrossAccountIAMConnector

How to access Cross Account MSK Connect with AWS MSK through IAM Authentication

Amazon Managed Streaming for Apache Kafka (MSK) Connect is a fully managed, scalable, and highly available service that enables the streaming of data between Apache Kafka and other data systems. MSK Connect is built on top of Kafka Connect, an open-source framework that provides a standard way to connect Kafka with external data systems. Kafka Connect supports a variety of connectors, which are used to stream data in and out of Kafka. MSK Connect extends the capabilities of Kafka Connect by providing a managed service with added security features, easier configuration, and automatic scaling capabilities, enabling businesses to focus on their data streaming needs without the overhead of managing the underlying infrastructure.

We have observed an increasing number of customers who require the capability to use an MSK cluster in one AWS account, while the MSK Connector is located in a separate account. In this blog post, we demonstrate how to create a connector to enable this use case. The connector can be configured for a variety of purposes, such as sink data to an S3 bucket or tracking the source database changes, or serving as a migration tool, such as MirrorMaker2 on MSK Connect, to transfer data from a source cluster to a target cluster which are located in different accounts.

<img width="1109" alt="cross" src="https://github.com/maheshsai84/CrossAccountIAMConnector/assets/68698107/8b9818fd-5340-410f-99b4-6a8332aaedcf">

Currently MSK connectors can be created only for MSK clusters which have IAM role-based authentication or no authentication. Establishing a simple VPC peering between two accounts with an unauthenticated cluster can easily achieve the desired result. However, the process becomes more complex when it comes to an IAM Authentication-enabled cluster which provides the secure communication between the resources. In a normal scenario, cross-account IAM connections are established through the assume role method, which requires the creation of multiple IAM roles. Unfortunately, this method is not suitable between MSK Connect and cross account MSK Cluster, as resource-based policies are not supported by MSK Clusters to accept/trust the cross account connections.

Fortunately, Amazon Managed Streaming for Apache Kafka (Amazon MSK) now provides multi-VPC private connectivity (powered by AWS PrivateLink) and cluster policy support for MSK clusters, simplifying the connectivity of Kafka clients to brokers. By enabling this Private Link feature on our source MSK Cluster, we can leverage the cluster-based policy to achieve our goals. This blog post will cover the process of enabling this feature on our source MSK Cluster.

 Please note that we are not utilizing the complete AWS Private Link feature in this scenario, as it generates multi-VPC Private endpoint brokers with ports 14003/02/01, which are not currently supported by the MSK Connect Service.

 Instead, we are using Amazon VPC Peering for network connectivity purposes. Amazon VPC peering is a straightforward networking construct that enables bidirectional connectivity between two VPCs. In this approach, the network administrator is responsible for updating each VPC with the IP addresses of each broker in the routing tables of all subnets. However, this connectivity pattern cannot be used when there are overlapping IPv4 or IPv6 CIDR blocks in the VPCs.

**Overview of solution**
-

Connecting to a cross account MSK Cluster from a MSK Connector involves the below high level steps.

 The following are the steps to configure the MSK cluster in Account A:

1. Enable the multi-VPC private  connectivity(Private Link) feature for IAM authentication scheme that is  enabled for your MSK cluster.
2. Configure cluster policy to allow cross  account connector.
3. Create VPC Peering request to Account B VPC  and make network changes accordingly.

The following are the steps to configure the MSK connector in Account B:

1. Create MSK Connector in Private subnets via  the CLI.
2. Accept the VPC Peering request from Account A  and make network changes accordingly.
3. Check the destination Service(S3, RDS, MSK  Cluster, etc) to verify the incoming data.

**Walkthrough**
-

MSK Cluster setup in Account A:

In this post, we only show the important steps that are required to enable the multi-VPC feature on a MSK cluster.

1. Create a provisioned MSK Cluster in Account A’s VPC with the below considerations which are required for multi-VPC feature.
    1. Cluster version must be 2.7.1 or higher
    2. Instance type must be m5.large or higher
    3. Authentication should be IAM(must not enable unauthenticated access for this cluster)
2. Once the cluster got created, go to the Networking settings of your cluster and then click on Edit. Then, it should be showing Turn on multi-VPC connectivity option. 

<img width="1095" alt="2 1" src="https://github.com/maheshsai84/CrossAccountIAMConnector/assets/68698107/30cdbb2a-919e-4940-a27b-8ae9b4a19713">

3. Click on it and choose IAM role-based authentication option and Turn on selection. It might take around 30 minutes to enable.

<img width="676" alt="3 1" src="https://github.com/maheshsai84/CrossAccountIAMConnector/assets/68698107/4e588600-9305-4678-bb56-3e79db452def">

4. The above step is required to enable the cluster policy feature which allows our cross account connector to access our MSK Cluster.
5. After it has been enabled, scroll down to Security settings and click on Edit cluster policy. Define your cluster policy and Save changes.

<img width="1096" alt="5 1" src="https://github.com/maheshsai84/CrossAccountIAMConnector/assets/68698107/a08d8281-8be5-4fbc-b195-c9d99383d9a3">

6. The new cluster policy allows for defining a Basic or Advanced cluster policy. With the Basic option, only allows CreateVPCConnection, GetBootstrapBrokers, DescribeCluster, and DescribeClusterV2 actions that are required for creating the cross-VPC connectivity to your cluster. However, we have to select Advanced to allow more actions which are required by the MSK Connector. The policy should be having the actions as below:

```
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {
            "AWS": "arn:aws:iam::284380556447:role/mahesh-msk-role"
        },
        "Action": [
            "kafka:CreateVpcConnection",
            "kafka:GetBootstrapBrokers",
            "kafka:DescribeCluster",
            "kafka:DescribeClusterV2",
            "kafka-cluster:Connect",
            "kafka-cluster:DescribeCluster",
            "kafka-cluster:ReadData",
            "kafka-cluster:DescribeTopic",
            "kafka-cluster:WriteData",
            "kafka-cluster:CreateTopic",
            "kafka-cluster:AlterGroup",
            "kafka-cluster:DescribeGroup"
        ],
        "Resource": [
            "arn:aws:kafka:<region>:<Cluster-AccountId>:cluster/<cluster-name>/<uuid>",
            "arn:aws:kafka:<region>:<Cluster-AccountId>:topic/<cluster-name>/*",
            "arn:aws:kafka:<region>:<Cluster-AccountId>:group/<cluster-name>/*"
        ]
    }]
}
```

Now the cluster is ready. However, we need the connection between the cross Account Connector VPC and the MSK Cluster VPC.


7. For that, create a VPC Peering connection request and add the connector’s account id and the VPC ID. Make sure that the both VPC CIDRs doesn’t overlap. Also, the cross region works here. Account B’s VPC should be created with the mentioned constraints in the Account B section below.

Note: While using Amazon VPC peering or Transit Gateway with Amazon MSK Connect, do not configure your connector for reaching the peered VPC resources with IPs in the CIDR ranges:

        * "10.99.0.0/16"
        * "192.168.0.0/16"
        * "172.21.0.0/16"

8. Then in, MSK Security group, allow Account B’s VPC CIDR with the required port (9098).
9. Then, in all MSK Subnet route tables, add a route with Account B’s VPC CIDR as destination and VPC-Peering connection id as target.

**Connector setup in Account B:**
-

Here, I am demonstrating the S3 Sink connector for simple understanding. However, you may use different type of connectors as per your use case and make the changes accordingly.

1. Firstly, Create a S3 bucket or you can use existing bucket as  well.
2. The VPC which you are using in this Account should have a  Security Group, Public Subnets and Private subnets. 
    1. If your connector for Amazon MSK Connect  needs access to the internet, we recommend that you use the Enabling  internet Access documentation. 
3. Then, accept the peering connection request from Account A.
4. Then in a Security group which you will be using for your  connector, allow Account A’s VPC CIDR with required post if applicable for  your use case. 
5. Then, in all Private Subnet route tables that you will be using  for your connector, add a route with Account A VPC CIDR as destination and  VPC-Peering connection id as target.
6. Then, you should create a S3  VPC Endpoint. Because, the the Connector and MSK are in a VPC and  Amazon S3 is not. This VPC Endpoint makes it possible to send data from  the Amazon VPC that has the cluster and the connector to Amazon S3. This  VPC Endpoint should be added to the route tables of your private subnets.
7. Then you should have a Connector Plugin as per your connector  type(confluent or lenses) by following documentation[3]. Please make a  note of the custom plugin, we will be using it later.
8. Then, create an IAM role for your connector to allow access to  your S3 bucket and the MSK Cluster.

The IAM role’s trust relationship should be as follows:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "kafkaconnect.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

**S3 Access policy:**

Then, add the below S3 Access policy to your IAM role.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListAllMyBuckets"
      ],
      "Resource": "arn:aws:s3:::*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::<destination-bucket>"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts",
        "s3:ListBucketMultipartUploads"
      ],
      "Resource": "*"
    }
  ]
}
```

**MSK Cluster access policy:**

Here, the below policy is having all the required actions by the connector.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "kafka-cluster:Connect",
                "kafka-cluster:DescribeCluster",
                "kafka-cluster:ReadData",
                "kafka-cluster:DescribeTopic",
                "kafka-cluster:WriteData",
                "kafka-cluster:CreateTopic",
                "kafka-cluster:AlterGroup",
                "kafka-cluster:DescribeGroup"
            ],
            "Resource": [
                "arn:aws:kafka:<region>:<Account-A-id>:cluster/<cluster-name>/<uuid>",
                "arn:aws:kafka:<region>:<Account-A-id>:topic/<cluster-name>/*",
                "arn:aws:kafka:<region>:<Account-A-id>:group/<cluster-name>/*"
            ]
        }
    ]
}
```

Finally, it’s time to create the MSK Connector. Kindly note, it is not possible to create the connector in AWS Console as we cannot select the MSK Cluster broker endpoints which are located in different account. Thus, we should be creating the connector via CLI/API. Also, I have used basic S3 configuration for testing purposes. You may need to modify the configuration according to your use case. 
Then, create a connector via AWS CLI using the below command with all the required parameters of connector, along with the Account A’s MSK Cluster broker endpoints.

```
aws kafkaconnect create-connector \
--capacity "autoScaling={maxWorkerCount=2,mcuCount=1,minWorkerCount=1,scaleInPolicy={cpuUtilizationPercentage=10},scaleOutPolicy={cpuUtilizationPercentage=80}}" \
--connector-configuration \
"connector.class=io.confluent.connect.s3.S3SinkConnector, \
s3.region=<region>, \
schema.compatibility=NONE, \
flush.size=2, \
tasks.max=1, \
topics=<MSK-Cluster-topic>, \
confluent.license=, \
security.protocol=SASL_SSL, \
s3.compression.type=gzip, \
format.class=io.confluent.connect.s3.format.json.JsonFormat, \
sasl.mechanism=AWS_MSK_IAM, \
sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required, \
sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler, \
value.converter=org.apache.kafka.connect.storage.StringConverter, \
storage.class=io.confluent.connect.s3.storage.S3Storage, \
s3.bucket.name=<s3-bucket-name>, \
timestamp.extractor=Record, \
key.converter=org.apache.kafka.connect.storage.StringConverter" \
--connector-name "Connector-name" \
--kafka-cluster '{"apacheKafkaCluster": {"bootstrapServers": "bootstrap:9098,"vpc": {"securityGroups": ["sg-0b36axxxxxxf859a3"],"subnets": ["subnet-07950xxxxx8be6d8","subnet-026a729xxxxxf9728"]}}}' \
--kafka-cluster-client-authentication "authenticationType=IAM" \
--kafka-cluster-encryption-in-transit "encryptionType=TLS" \
--kafka-connect-version "2.7.1" \
--log-delivery workerLogDelivery='{cloudWatchLogs={enabled=true,logGroup="<MSKConnect-log-group-name>"}}' \
--plugins "customPlugin={customPluginArn=<Custom-Plugin-ARN>,revision=1}" \
--service-execution-role-arn "<IAM-role-ARN>"
```

After deploying the connector, if it is in the CREATING state on the connector console, access the CloudWatch log group specified in your Connector creation request. Review the logs for any errors. If no errors are found, wait for the connector to complete it’s creation process.

Once the connector is created, use a Kafka client to insert data into your MSK topic.

```
bin/kafka-console-producer.sh --broker-list b-1.msk.abc124.c14.kafka.us-east-1.amazonaws.com:9098,b-2.abc.abc124.c14.kafka.us-east-1.amazonaws.com:9098,b-3.msk.abc124.c14.kafka.us-east-1.amazonaws.com:9098 —producer.config client.properties —topic <topic-name>
```

<img width="232" alt="6 1" src="https://github.com/maheshsai84/CrossAccountIAMConnector/assets/68698107/9c205ce3-ad3c-491d-8e1d-d351972946f9">

You can check the output at your destination S3 bucket.

<img width="1340" alt="7 1" src="https://github.com/maheshsai84/CrossAccountIAMConnector/assets/68698107/1a8eabdf-3180-4b6e-bcb1-c2107450586c">

**Conclusion:**
-

This architecture includes an S3 Sink connector for demonstration purposes, but it can accommodate any type of sink and source connectors. Additionally, this architecture focuses solely on IAM-authenticated connectors. If an unauthenticated connector is desired, the Multi-VPC Connectivity (Private Link) and cluster policy components can be ignored. The remaining process, which involves creating a VPC Peering connection between the account VPCs, remains the same.




