# Complete MSK workshop

## Blogs that need to be incorporated
* KDA = https://aws.amazon.com/blogs/big-data/build-a-real-time-streaming-application-using-apache-flink-python-api-with-amazon-kinesis-data-analytics/
* KDA = https://streaming-analytics.workshop.aws/flink-on-kda/
* https://aws.amazon.com/blogs/big-data/govern-how-your-clients-interact-with-apache-kafka-using-api-gateway/

## Pre-requisite
* Two AWS Accounts
    * Account A = AWS MSK Cluster, Kafka Consumer and Producer EC2 instances, Cloud9 Bastion host, Prometheus and Grafana EC2 instance 
    * Account B = AWS Glue Schema Registry Account (optional)
## Let's get going
* Login into Account A
    * Go to EC2 console and create a keypair to be able to ssh into Kafka and Prometheus EC2 instances that will be created during the workshop
    * Go to AWS MSK console and create a MSK Cluster Configuration
        
            * auto.create.topics.enable=true
            * delete.topic.enable=true
            * log.retention.hours=8
            * default.replication.factor=3
            * min.insync.replicas=2
            * num.io.threads=8
            * num.network.threads=5
            * num.partitions=1
            * num.replica.fetchers=2
            * replica.lag.time.max.ms=30000
            * socket.receive.buffer.bytes=102400
            * socket.request.max.bytes=104857600
            * socket.send.buffer.bytes=102400
            * unclean.leader.election.enable=true
            * zookeeper.session.timeout.ms=18000
    * Copy Cluster Configuration ARN in a notepad to use it in subsequent steps
    * Go to CloudFormation console
        * Create a VPC stack = [VPC Stack](cloudformation-stacks/MSK-VPC.yml)
            * VPC with 1 public subnet and 3 private subnets + Nat Gateways
        * Create a Kafka clients stack = [Kafka Clients Stack](cloudformation-stacks/MSK-Clients.yaml)
            * 3 EC2 instances for a Kafka Producer, Consumer and Prometheus & Grafana
            * Kafka Producer & Consumer instances in separate Private subnets
            * Prometheus & Grafana instance in a Public subnet
            * Cloud9 environment to be used as a Bastion Host
            * CloudFormation parameters
               *  GlueSchemaRegistryAccountId = __you can use Account A id, if you don't have the second AWS Account__
               *  KeyName = __EC2 keypair from Account A__
               *  VPCStackNAme = __VPC Stack Name that you created prior to this stack__
               *  YouIPAddress = __You can use https://checkip.amazonaws.com/ to find your laptop IP address__
        * Create a MSK Cluster stack = [AWS MSK Cluster Stack](cloudformation-stacks/MSK-Cluster.yaml)
            * CloudFormation parameters
                * ClusterConfigARN = __AWS MSK Cluster Configuration ARN that you created in the previous step__
                * ClusterConfigRevisionNumber = 1
                * KafkaClientStack = __Kakfa Client CloudFormation Stack Name, stack that you created prior to this__
                * MSKKafkaVersion = 2.7
                * PCAARN = __Leave it blank__
                * TLSMutualAuthentication = __false__
                * VPCStack = __VPC CloudFormation Stack Name, the first stack that you created__
    * Once the cluster is up and running proceed with the next steps
    <hr> 
  
## Cloud9 Bastion Host steps   
* Open Cloud 9 Terminal and upload ssh keypair file in cloud9 (upload local files), that you created in Account A
* Change keypair.pem file permissions
      
        chmod 0400 <keypair.pem>
* Create ssh scripts to be able to ssh into Prometheus, Producer and Consumer instances
    * producer.sh
          
            ssh -i <ec2 keypair.pem> ec2-user@<private ip of Producer EC2 instance>
    * consumer.sh

            ssh -i <ec2 keypair.pem> ec2-user@<private ip of Consumer EC2 instance>
    * prometheus.sh

            ssh -i <ec2 keypair.pem> ec2-user@<private ip of prometheus EC2 instance>
    * Make shell script executable
            
            chmod +x producer.sh prometheus.sh consumer.sh
   
    
__Note:__ _Wait for MSK Cluster stack to complete before proceeding to the next steps_
<hr>

## Prometheus EC2 instance Basic Setup
* From Cloud9 terminal ssh into Prometheus EC2 instance
     
        ./prometheus.sh
* Once Sshed into Prometheus instance, configure region to `ap-southeast-2`
        
        aws configure        
        
* Set the environment variables 
    * CLUSTER_ARN = AWS MSK Cluster ARN
    * BS = AWS MSK Cluster Broker nodes endpoint
    * ZK = AWS MSK Cluster Zookeeper nodes endpoint

            ~/ vim .bash_profile
            
            export CLUSTER_ARN=`aws kafka list-clusters|grep ClusterArn|cut -d ':' -f 2-|cut -d ',' -f 1 | sed -e 's/\"//g'`
            
            export BS=`aws kafka get-bootstrap-brokers --cluster-arn $CLUSTER_ARN|grep BootstrapBrokerString|grep 9092| cut -d ':' -f 2- | sed -e 's/\"//g' | sed -e 's/,$//'`
            
            export ZK=`aws kafka describe-cluster --cluster-arn $CLUSTER_ARN|grep ZookeeperConnectString|grep -v Tls|cut -d ':' -f 2-|sed 's/,$//g'|sed -e 's/\"//g'`
            
            # save changes and exit .bash_profile
            
            # load environment variables in profile
            ~/ source .bash_profile
            
            # verify environment variables values
      
            echo $CLUSTER_ARN
            
            echo $BS
            
            echo $ZK
<hr>
  
## Producer EC2 instance Basic Setup
* From Cloud9 terminal ssh into Prometheus EC2 instance

        ./producer.sh
* Once Sshed into Prometheus instance, configure region to `ap-southeast-2`

        aws configure        

* Set the environment variables, follow the commands/instructions given below bash_profile
    * CLUSTER_ARN = AWS MSK Cluster ARN        
    * BS = AWS MSK Cluster Broker nodes endpoint
    * ZK = AWS MSK Cluster Zookeeper nodes endpoint

            ~/ vim .bash_profile
        
            #append ~/kafka/bin to PATH
            #PATH variable should look something like
            
            PATH=$PATH:$HOME/.local/bin:$HOME/bin:~/kafka/bin
            export PATH
      
            export CLUSTER_ARN=`aws kafka list-clusters|grep ClusterArn|cut -d ':' -f 2-|cut -d ',' -f 1 | sed -e 's/\"//g'`
        
            export BS=`aws kafka get-bootstrap-brokers --cluster-arn $CLUSTER_ARN|grep BootstrapBrokerString|grep 9092| cut -d ':' -f 2- | sed -e 's/\"//g' | sed -e 's/,$//'`
        
            export ZK=`aws kafka describe-cluster --cluster-arn $CLUSTER_ARN|grep ZookeeperConnectString|grep -v Tls|cut -d ':' -f 2-|sed 's/,$//g'|sed -e 's/\"//g'`
        
            # save changes and exit .bash_profile
        
            # load environment variables in profile
            ~/ source .bash_profile
        
            # verify environment variables values
            
            echo $CLUSTER_ARN
        
            echo $BS
        
            echo $ZK
<hr>

## Consumer EC2 instance Basic Setup

* From Cloud9 terminal ssh into Prometheus EC2 instance

        ./consumer.sh
* Once Sshed into Prometheus instance, configure region to `ap-southeast-2`

        aws configure        

* Set the environment variables, follow the commands/instructions given below bash_profile
    * CLUSTER_ARN = AWS MSK Cluster ARN
    * BS = AWS MSK Cluster Broker nodes endpoint
    * ZK = AWS MSK Cluster Zookeeper nodes endpoint
    
            ~/ vim .bash_profile
            
            #append ~/kafka/bin to PATH
            #PATH variable should look something like
            
            PATH=$PATH:$HOME/.local/bin:$HOME/bin:~/kafka/bin
            export PATH
      
            export CLUSTER_ARN=`aws kafka list-clusters|grep ClusterArn|cut -d ':' -f 2-|cut -d ',' -f 1 | sed -e 's/\"//g'`

            export BS=`aws kafka get-bootstrap-brokers --cluster-arn $CLUSTER_ARN|grep BootstrapBrokerString|grep 9092| cut -d ':' -f 2- | sed -e 's/\"//g' | sed -e 's/,$//'`

            export ZK=`aws kafka describe-cluster --cluster-arn $CLUSTER_ARN|grep ZookeeperConnectString|grep -v Tls|cut -d ':' -f 2-|sed 's/,$//g'|sed -e 's/\"//g'`

            # save changes and exit .bash_profile

            # load environment variables in profile
            ~/ source .bash_profile

            # verify environment variables values
             
            echo $CLUSTER_ARN

            echo $BS

            echo $ZK
<hr>

## Let's Produce & Consume Message to Kafka Topic for further use cases
__Note:__ Start consumer first

* __Consume messages__
    * ssh into Consumer EC2 instance from Cloud9 terminal

            ./consumer.sh
    * List existing topics in your MSK Cluster

          kafka-topics.sh --bootstrap-server $BS --list
    * Create topic __workshop-topic__

          kafka-topics.sh --bootstrap-server $BS --topic workshop-topic \
          --create --partitions 3 \
          --replication-factor 2

          kafka-topics.sh --bootstrap-server $BS \
          --topic workshop-topic --describe

    * Consume message

            kafka-console-consumer.sh --bootstrap-server $BS \
            --topic workshop-topic --group workshop-app \
            --from-beginning

* __Produce messages__
    * ssh into Producer EC2 instance from Cloud9 terminal
    
            ./producer.sh
    
    * Generate / Produce traffic for __workshop-topic__ topic

            echo $BS
        
            #copy output from the previous echo command which is list of brokers
  
            kafka-producer-perf-test.sh \
            --producer-props bootstrap.servers=<BROKERS LIST COPIED IN PREVIOUS STEP> \
            acks=all --throughput 100 \
            --num-records 9999 \
            --topic workshop-topic \
            --record-size 1000

* __Kill Consumer that you started earlier before Producer finishes sending all the messages. We would like to keep the lag as we want to check the Consumer lag metrics later__
    * In a consumer terminal
    
            CTRL + C
    * Get list of consumer groups
    
            kafka-consumer-groups.sh --bootstrap-server $BS --list
    * Describe __workshop-app__ consumer group
    
            kafka-consumer-groups.sh --bootstrap-server $BS --describe --group workshop-app
    * Note the following to understand the consumer lag
        * CURRENT-OFFSET
        * LOG-END-OFFSET
        * LAG
    
<hr>

## Open Monitoring
### Install Prometheus ==> on Prometheus EC2 Instance  
* Create user
    
            sudo useradd -M -r -s /bin/false prometheus

* Create required directories
            
            sudo mkdir /etc/prometheus /var/lib/prometheus

* Download prometheus tar
    
            wget https://github.com/prometheus/prometheus/releases/download/v2.29.1/prometheus-2.29.1.linux-amd64.tar.gz

            tar xvfz prometheus-2.29.1.linux-amd64.tar.gz

            rm prometheus-2.29.1.linux-amd64.tar.gz

* Copy binary files to /usr/local/bin
        
            sudo cp prometheus-2.29.1.linux-amd64/{prometheus,promtool} /usr/local/bin/

* Change ownership of Prometheus binary files to user prometheus

            sudo chown prometheus:prometheus /usr/local/bin/{prometheus,promtool}

            sudo cp -r prometheus-2.29.1.linux-amd64/{consoles,console_libraries} /etc/prometheus/

            sudo cp prometheus-2.29.1.linux-amd64/prometheus.yml /etc/prometheus/prometheus.yml

            sudo chown -R prometheus:prometheus /etc/prometheus

            sudo chown prometheus:prometheus /var/lib/prometheus

* Run Prometheus and verify if it works

            prometheus --config.file=/etc/prometheus/prometheus.yml
* Last line in the logs should be
            
            msg="Server is ready to receive web requests."
* Stop prometheus server  
            
            CTL+C
* Configure Prometheus service in systemd

            sudo vi /etc/systemd/system/prometheus.service
* Add the following content in __prometheus.service__
    
            [Unit]
            Description=Prometheus Time Series Collection and Processing Server
            Wants=network-online.target
            After=network-online.target
            
            [Service]
            User=prometheus
            Group=prometheus
            Type=Simple
            ExecStart=/usr/local/bin/prometheus \
            --config.file /etc/prometheus/prometheus.yml \
            --storage.tsdb.path /var/lib/prometheus/ \
            --web.console.templates=/etc/prometheus/consoles \
            --web.console.libraries=/etc/prometheus/console_libraries
            
            [Install]
            WantedBy=multi-user.target

* Refresh the systemd content

            sudo systemctl daemon-reload

* Start Prometheus service on the Prometheus EC2 instance
    
            sudo systemctl start prometheus

            sudo systemctl status prometheus
                
            ## Look for "Active: active (running)" to ensure service is configured and started correctly

* Enable Prometheus service so that it starts automatically everytime Prometheus EC2 instance restarts
        
            sudo systemctl enable prometheus

* Lets __curl__ to check if prometheus service that we configured in the previous step is responding to curl commands
* From the same terminal in Prometheus EC2 instance run the curl command

            curl localhost:9090

            You should see the following response <a href="/graph">Found</a>

* Lets check from browser as well. Open browser on your Laptop and enter the following URL to see if you get Prometheus Expression Browser

              <public ip address of prometheus server>:9090

* Enter __up__ in the expression bar and click __Execute__. You should see

                up{instance="localhost:9090", job="prometheus"}    1
* Let's change the configuration of Prometheus server and see if changes take effect
* Update __prometheus.yml__

            sudo vi /etc/prometheus/prometheus.yml

* Change the scrape frequency to __10s__

* Reloading the systemd configuration

            sudo killall -HUP prometheus
* check prometheus config if scrape frequency changes are in place. From your Prometheus EC2 terminal window, run the following

            curl localhost:9090/api/v1/status/config
                
            you should see the following "scrape_timeout: 10s\n"
    
### Install Node Exporter ==> on Prometheus EC2 instance
* We are going to configure node exporter on this Prometheus EC2 instance to explore working of Prometheus set up on this EC2 instance.
* We are still in __Prometheus EC2 instance (sshed)__
* Add user node_exporter
    
            sudo useradd -M -r -s /bin/false node_exporter
* Download node_exporter, move files to their appropriate locations and change ownership of directories/files
        
            wget https://github.com/prometheus/node_exporter/releases/download/v1.2.2/node_exporter-1.2.2.linux-amd64.tar.gz
      
            tar xvfz node_exporter-1.2.2.linux-amd64.tar.gz
      
            rm node_exporter-1.2.2.linux-amd64.tar.gz
        
            sudo cp node_exporter-1.2.2.linux-amd64/node_exporter /usr/local/bin/
            
            sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
* Configure Node Exporter service in systemd
    
            sudo vi /etc/systemd/system/node_exporter.service
* Add the following content in __node_exporter.service__
    
            [Unit]
            Description=Prometheus Time Series Collection and Processing Server
            Wants=network-online.target
            After=network-online.target

            [Service]
            User=node_exporter
            Group=node_exporter
            Type=Simple
            ExecStart=/usr/local/bin/node_exporter
            
            [Install]
            WantedBy=multi-user.target
* Refresh the systemd content
      
            sudo systemctl daemon-reload
* Start Node Exporter service on the Prometheus EC2 instance
    
            sudo systemctl start node_exporter
    
            sudo systemctl status node_exporter
    
            ## Look for "Active: active (running)" to ensure service is configured and started correctly
* Enable Node Exporter service so that it starts automatically everytime Prometheus EC2 instance restarts
    
            sudo systemctl enable node_exporter
* Lets curl to check if Node Exporter service that we configured in the previous step is responding to curl commands
* From the same terminal in Prometheus EC2 instance run the curl command
    
            curl localhost:9100/metrics
        
### Configure Prometheus to get Metrics from Node Exporter
* Update __prometheus.yml__ file
    
            sudo vi /etc/prometheus/prometheus.yml

* Add the following in __prometheus.yml__ under __scrape_configs:__
      
            - job_name: "Prometheus Linux Server"
                static_configs:
                - targets: ["<private ip of prometheus server>:9100"]
* Reload the Config changes
      
            sudo killall -HUP prometheus
* Let's verify if Prometheus Server is being monitored. Open browser on your laptop and access Prometheus expression browser

            http://<public ip address of Prometheus Server>:9090/
    
            You should see the following results - First one shows Prometheus Application is up and running, Second one shows Node_exporter is up and running which is getting metrics for Prometheus EC2 instance
            
            up{instance="10.0.0.237:9090", job="prometheus"}                     1
            up{instance="10.0.0.237:9100", job="Prometheus Linux Server"}        1 
    
* __Awesome!!__ you have configured Prometheus instance and verified it's working with __node_exporter__ installed locally on __Prometheus EC2 instance__
    
### Let's get real!! 
## Get Node metrics and JMX metrics from Your MSK Cluster

* We are still Prometheus EC2 instance (sshed)
* Get Your MSK Cluster Brokers details
    
        echo $BS 
* Copy Broker details and have them ready in the following format
    * 11001 is a port where __JMX Exporter__ is running on MSK Brokers
    * 11002 is a port where __Node Exporter__ is running on MSK brokers

            "b-1.mskcluster-msk.xxxxxxxxxxx.ap-southeast-2.amazonaws.com:11001",
            "b-2.mskcluster-msk.xxxxxxxxxxx.ap-southeast-2.amazonaws.com:11001",
            "b-3.mskcluster-msk.xxxxxxxxxxx.ap-southeast-2.amazonaws.com:11001"
    
            "b-1.mskcluster-msk.xxxxxxxxxxx.ap-southeast-2.amazonaws.com:11002",
            "b-2.mskcluster-msk.xxxxxxxxxxx.ap-southeast-2.amazonaws.com:11002",
            "b-3.mskcluster-msk.xxxxxxxxxxx.ap-southeast-2.amazonaws.com:11002"

* Configure __prometheus.yml__ file with MSK Brokers JMX and Node exporter details 

            sudo vi /etc/prometheus/prometheus.yml
    
* Add the following under __scrape_configs__
    
__Note:__ replace broker details with your broker details (strings) that you prepared in the previous step
    
    - job_name: "jmx"
      static_configs:
        - targets: [
          "b-1.mskcluster-msk.xxxx.c2.kafka.ap-southeast-2.amazonaws.com:11001",
          "b-2.mskcluster-msk.xxxx.c2.kafka.ap-southeast-2.amazonaws.com:11001",
          "b-3.mskcluster-msk.xxxx.c2.kafka.ap-southeast-2.amazonaws.com:11001"
          ]
    - job_name: "node"
      static_configs:
        - targets: [
          "b-1.mskcluster-msk.xxxx.c2.kafka.ap-southeast-2.amazonaws.com:11002",
          "b-2.mskcluster-msk.xxxx.c2.kafka.ap-southeast-2.amazonaws.com:11002",
          "b-3.mskcluster-msk.xxxx.c2.kafka.ap-southeast-2.amazonaws.com:11002"
          ] 
* Reload the config changes

            sudo killall -HUP prometheus
* Access Prometheus expression browser and type __up__ in the Expression query field and click __Execute__
    
        http://<public ip of prometheus server>:9090
  
        you should see the following results (first two are from the configuration on Prometheus EC2 instance, 
        rest 6 are for MSK Brokers, 3 each for JMX and Node Exporter
  
        up{instance="10.0.0.237:9090", job="prometheus"}                                                          1
        up{instance="10.0.0.237:9100", job="Prometheus Linux Server"}                                             1
        up{instance="b-1.mskcluster-msk.xxxxxx.c2.kafka.ap-southeast-2.amazonaws.com:11001", job="jmx"}           1
        up{instance="b-1.mskcluster-msk.xxxxxx.c2.kafka.ap-southeast-2.amazonaws.com:11002", job="node"}          1
        up{instance="b-2.mskcluster-msk.xxxxxx.c2.kafka.ap-southeast-2.amazonaws.com:11001", job="jmx"}           1
        up{instance="b-2.mskcluster-msk.xxxxxx.c2.kafka.ap-southeast-2.amazonaws.com:11002", job="node"}          1
        up{instance="b-3.mskcluster-msk.xxxxxx.c2.kafka.ap-southeast-2.amazonaws.com:11001", job="jmx"}           1
        up{instance="b-3.mskcluster-msk.xxxxxx.c2.kafka.ap-southeast-2.amazonaws.com:11002", job="node"}          1
      
* Type the following in the Expression query field

        kafka_controller_KafkaController_Value{name="OfflinePartitionsCount"}
        
        OR
  
        kafka_consumer_group_ConsumerLagMetrics_Value
<hr>  
    
## Configure Grafana Service - on Prometheus EC2 Instance
    
* We are still in Prometheus EC2 instance terminal
* Download and Install Grafana 
            
          wget https://dl.grafana.com/oss/release/grafana-8.1.2-1.x86_64.rpm
            
          sudo yum install grafana-8.1.2-1.x86_64.rpm
* Change Grafana config 

          pushd /etc/grafana/
           
          sudo vim grafana.ini
            
          search for [live]
* Add the following 
       
          allowed_origins="*"
* Reload changes and Start Grafana Service by running the following commands

          sudo systemctl daemon-reload
      
          sudo systemctl start grafana-server
      
          sudo systemctl status grafana-server
            # you should see the following Active: active (running)
      
          sudo systemctl enable grafana-server
* Access grafana, open browser on your local machine/laptop and enter the following URL
          
          #Prometheus and Grafana are running on the same EC2 instance which is  
           named as __PrometheusInstance__ hence use PrometheusInstance EC2 instance public IP address
      
          <public-ip address of grafana server>:3000
    
* You should get Grafana login page, use default username and password
        
          username = admin
          password = admin
* On New Password screen, press __Skip__
* Let's configure Grafana to fetch metrics from Prometheus
* Add Prometheus datasource, Click __cogwheel icon__ in left panel and click on __data sources__
* Click __Add data source__ and select __Prometheus__
* In __URL__ add 
      
            http://<PRIVATE IP address of prometheus server>:9090 
* Press Save and Test, you should see the following
            
            Data source is working

* Create Kafka metrics Dashboard, download the following dashboard config json

            https://amazonmsk-labs.workshop.aws/en/openmonitoring/msk_grafana_dashboard.json

* Click on four square icon and click on manage and import __msk_grafana_dashboard.json__
* You should see Grafana Dashboard with nice graphs and metrics number on the top
* __Grafana and Prometheus Configuration is Working!!__ Metrics are being fetched from MSK Cluster.
* Click on __Refresh__ icon in the top right corner and select __5s__. You should see Graphs moving!!
   
### Configure your MSK cluster Cloudwatch Metrics on Grafana Dashboard
* Create Cloudwatch datasource, Click __cogwheel icon__ in left panel and click on __data sources__
* Choose Data Source = __Cloudwatch__
    * __Authentication Provider__ = AWS SDK Default (it will pick up EC2 instance role)
    * __Default region__ = ap-southeast-2
        * Leave rest of the fields blank
    * Click __Save & Test__. You should see the following

            Data source is working
* Add new panel (empty panel) to Kafka Dashboard that you created in the previous step
    * Select
    * __Data source = Cloudwatch__
    * __Query Mode = Cloudwatch Metrics__
    * __Region = ap-southeast-2__
    * __Namespace = AWS/Kafka__
    * __Metric Name = MaxOffsetLag__
    * __Stats = Maximum__
    * Add the following dimensions 
    * __Cluster Name = SELECT YOUR CLUSTER__
        * __Topic = SELECT TOPIC__
        * __Consumer Group = SELECT CONSUMER GROUP__
   * OR
   * you can add the following in __Expression__
        
            SEARCH('{AWS/Kafka,"Cluster Name","Consumer Group",Partition,Topic} MSKCluster-MSK workshop-topic ml OffsetLag', 'Average', 300)

<hr>    

## Configure Cruise Control
* From Cloud9 terminal, ssh into Prometheus EC2 instance
  
        ./prometheus.sh
* check java 8 installation

        sudo alternatives --config java
          
        # if Java 8 is already installed, no need to execute the the next steps (1,2,3,4) install which installs java 8
        1) sudo yum -y install java-1.8.0-openjdk java-1.8.0-openjdk-devel
        2) sudo alternatives --config java
        3) sudo update-alternatives --config javac      
        4) select Java 8
* Run the following to verify Java and Javac installations

        java -version
        javac -version

        both should be Java 8
* Verify MSK kafka environment variables once again
  
        # check your environment variables once again
        echo $CLUSTER_ARN
        echo $BS
        echo $ZK
* Download Cruise control
   
        cd ~
  
        wget https://github.com/linkedin/cruise-control/archive/2.5.22.tar.gz

        tar xvfz 2.5.22.tar.gz
        
        rm 2.5.22.tar.gz    
* To build Cruise Control you must initialize it as a git repo

        cd cruise-control-2.5.22/

        git init && git add . && git commit -m "Init local repo." && git tag -a 2.0.130 -m "Init local version."

        #Build CruiseControl
        ./gradlew jar copyDependantLibs
* Make changes in __cruisecontrol.properties__
     
    __NOTE:__ __9090__ is prometheus server port, __9091__ is cruise control front end server port

        sed -i "s/localhost:9092/${BS}/g" config/cruisecontrol.properties

        sed -i "s/localhost:2181/${ZK}/g" config/cruisecontrol.properties

        sed -i "s/com.linkedin.kafka.cruisecontrol.monitor.sampling.CruiseControlMetricsReporterSampler/com.linkedin.kafka.cruisecontrol.monitor.sampling.prometheus.PrometheusMetricSampler/g" config/cruisecontrol.properties

        sed -i "s/webserver.http.port=9090/webserver.http.port=9091/g" config/cruisecontrol.properties

        sed -i "s/capacity.config.file=config\/capacityJBOD.json/capacity.config.file=config\/capacityCores.json/g" config/cruisecontrol.properties
    
        sed -i "s/two.step.verification.enabled=false/two.step.verification.enabled=true/g" config/cruisecontrol.properties
  
        echo "prometheus.server.endpoint=<PRIVATE IP ADDRESS OF PROMETHEUS EC2 INSTANCE>:9090" >> config/cruisecontrol.properties
        
        mkdir logs; touch logs/kafka-cruise-control.out  
    __NOTE:__ Replace  __PRIVATE IP ADDRESS OF PROMETHEUS EC2 INSTANCE__ with Prometheus EC2 instance private IP address

* Modify __config/capacityCores.json__ for m5 large

        vi config/capacityCores.json
         
        {
            "brokerCapacities":[
                {
                    "brokerId": "-1",
                    "capacity": {
                        "DISK": "10737412742445",
                        "CPU": {"num.cores": "2"},
                        "NW_IN": "1073741824",
                        "NW_OUT": "1073741824"
                    },      
                    "doc": "This is the default capacity. Capacity unit used for disk is in MB, cpu is in number of cores, network throughput is in KB."
                }
            ]
        }
### Install CruiseControl UI
* We are still in Prometheus EC2 instance

        cd ~
        wget https://github.com/linkedin/cruise-control-ui/releases/download/v0.3.4/cruise-control-ui-0.3.4.tar.gz
    
        tar -xvzf cruise-control-ui-0.3.4.tar.gz
        rm cruise-control-ui-0.3.4.tar.gz
        mv cruise-control-ui cruise-control-2.5.22/

* run cruise control

        cd cruise-control-2.5.22
        ./kafka-cruise-control-start.sh -daemon config/cruisecontrol.properties

* check logs

        tail -f logs/kafkacruisecontrol.log

__NOTE:__ Wait for few mins to let CruiseControl gather the data

* Check if CruiseControl has able to create two topics in your MSK cluster
* From another Cloud9 terminal, ssh into Producer EC2 instance

    ./producer.sh
* Run the following to check list of topics in your MSK cluster

        kafka-topics.sh --bootstrap-server $BS --list
  
        You should see the following TWO CruiseControl topics
    
            __KafkaCruiseControlModelTrainingSamples
            __KafkaCruiseControlPartitionMetricSamples
    
* Test Cruise control UI browser 

        http://<PUBLIC IP ADDRESS of Prometheus EC2 Instance>:9091

* Exit Log tailing and Curl few times
    * on Cloud9 terminal window where you sshed into Prometheus instance and started CruiseControl        
        
            CTRL + C

            # Curl commands
  
            curl -X OPTIONS -v http://<PRIVATE IP of Prometheus EC2>:9091/kafkacruisecontrol/kafka_cluster_state?json=true
            curl -v http://<PRIVATE IP of Prometheus EC2>:9091/kafkacruisecontrol/kafka_cluster_state?json=true
            curl -v http://<PRIVATE IP of Prometheus EC2>:9091/kafkacruisecontrol/load?allow_capacity_estimation=true&json=true
            curl -v http://<PRIVATE IP of Prometheus EC2>:9091/kafkacruisecontrol/state?substates=EXECUTOR&verbose=true&json=true
            curl -v http://<PRIVATE IP of Prometheus EC2>:9091/kafkacruisecontrol/proposals?verbose=true&json=true
            curl -v http://<PRIVATE IP of Prometheus EC2>:9091/kafkacruisecontrol/user_tasks?json=true
            curl -v http://<PRIVATE IP of Prometheus EC2>:9091/kafkacruisecontrol/kafka_cluster_state?json=true
            curl -v -X POST http://<PRIVATE IP of Prometheus EC2>:9091/kafkacruisecontrol/rebalance?dryrun=true&goals=RackAwareGoal%2CReplicaCapacityGoal%2CCpuCapacityGoal%2CDiskCapacityGoal%2CNetworkInboundCapacityGoal%2CNetworkOutboundCapacityGoal&json=true

* Shutdown CruiseControl

  ./kafka-cruise-control-stop.sh

<hr>

# Security & Encryption
* Currently, Amazon Managed Streaming for Apache Kafka (Amazon MSK) supports encryption in transit with TLS and TLS mutual authentication with TLS certificates for client authentication

* Amazon MSK utilizes Amazon Certificate Manager Private Certificate Authority (ACM PCA) for TLS mutual authentication.

* In addition, for Amazon MSK to be able to use the ACM PCA, it needs to be in the same AWS account as the Amazon MSK cluster.
* However, the Apache Kafka clients, for example, the producers and consumers, schema registries, Kafka Connect or other Apache Kafka tools that need the end-entity certificates can be in an AWS account different from the AWS account that the ACM PCA is in.
* In that scenario, in order to be able to access the ACM PCA, they need to assume a role in the account the ACM PCA is in and has the required permissions as the ACM PCA does not support resource-based policies, only identity-based policies.

## Key concept
* __In-Transit Encryption__
    * If encryption in-transit is enabled for an Amazon MSK cluster, Public TLS certificates from ACM are installed in the Amazon MSK Apache Kafka brokers keystores.
* __Mutual TLS__
    * __BROKERs Side__
        * If TLS mutual authentication is enabled for the Amazon MSK cluster, you need to provide an ACM PCA Amazon Resource Number (ARN) that the Amazon MSK cluster can utilize. The CA certificate and the certificate chain of the specified PCA are retrieved and installed in the truststores of the Amazon MSK Apache Kafka brokers.
    * __Kafka Clients (Producer/Consumer) Side__
        * On the clients, you need to generate a Private Key and create a CSR (Certificate Signing Request) that are used to get end-entity certificates issued by the ACM PCA specified for an Amazon MSK cluster.
        * These certificates and their certificate chains are installed in the keystores on the client and are trusted by the Amazon MSK Apache Kafka brokers.
## Let's Get Started

## Pre-requisite
* Two AWS Accounts
    * Account A = AWS MSK Cluster, Kafka Consumer and Producer EC2 instances, Cloud9 Bastion host, Prometheus and Grafana EC2 instance
    * Account B = AWS Glue Schema Registry (optional)
## Let's get going
* Login into Account A
    * Go to EC2 console and create a keypair to be able to ssh into Kafka and Prometheus EC2 instances that will be created during the workshop
    * Go to AWS MSK console and create a MSK Cluster Configuration

            * auto.create.topics.enable=true
            * delete.topic.enable=true
            * log.retention.hours=8
            * default.replication.factor=3
            * min.insync.replicas=2
            * num.io.threads=8
            * num.network.threads=5
            * num.partitions=1
            * num.replica.fetchers=2
            * replica.lag.time.max.ms=30000
            * socket.receive.buffer.bytes=102400
            * socket.request.max.bytes=104857600
            * socket.send.buffer.bytes=102400
            * unclean.leader.election.enable=true
            * zookeeper.session.timeout.ms=18000
    * **Copy Cluster Configuration ARN in a notepad to use it in subsequent steps**
    * Go to CloudFormation console
        * Create a VPC stack = [VPC Stack](cloudformation-stacks/MSK-VPC.yml)
            * VPC with 1 public subnet and 3 private subnets + Nat Gateways
        * Create a Kafka clients stack = [Kafka Clients Stack](cloudformation-stacks/MSK-Clients.yaml)
            * 3 EC2 instances for a Kafka Producer, Consumer and Prometheus & Grafana
            * Kafka Producer & Consumer instances in separate Private subnets
            * Prometheus & Grafana instance in a Public subnet
            * Cloud9 environment to be used as a Bastion Host
            * CloudFormation parameters
               *  GlueSchemaRegistryAccountId = __you can use Account A id, if you don't have the second AWS Account__
               *  KeyName = __EC2 keypair from Account A__
               *  VPCStackNAme = __VPC Stack Name that you created prior to this stack__
               *  YouIPAddress = __You can use https://checkip.amazonaws.com/ to find your laptop IP address__
## Setup AWS Certificate Manager (ACM) Private Certificate Authority (PCA)
* Open Terminal in Cloud9 environment and ssh into Producer EC2 instance to run the following CLI command

      aws acm-pca create-certificate-authority \
      --certificate-authority-configuration '{"KeyAlgorithm":"RSA_2048","SigningAlgorithm":"SHA256WITHRSA","Subject":{"Country":"US","Organization":"Amazon","OrganizationalUnit":"AWS","State":"New York","CommonName":"MyMSKPCA","Locality":"New York City"}}' --revocation-configuration '{"CrlConfiguration":{"Enabled":false}}' --certificate-authority-type "ROOT" --idempotency-token 12345

* Copy the ARN (Amazon Resource Name) of the PCA you just created to a notepad application.

* Install a self-signed certificate in the ACM PCA just created. A certificate needs to be installed in the ACM PCA for the PCA to be able to issue and sign end-entity certificates.
    * Go to the AWS ACM Console.
    * Click on Private CAs that you created in the previous step, status should be "__Pending Certificate__" 
    * Select the PCA you just created. Click on __Actions__ dropdown and select __Install CA Certificate__
        * Don't change defaults, click Next and Confirm and Install
## Create MSK Cluster
* Create a MSK Cluster stack = [AWS MSK Cluster Stack](cloudformation-stacks/MSK-Cluster.yaml)
  * CloudFormation parameters
    * ClusterConfigARN = __AWS MSK Cluster Configuration ARN that you created in the previous step__
    * ClusterConfigRevisionNumber = 1 
    * KafkaClientStack = __Kakfa Client CloudFormation Stack Name, stack that you created prior to this__
    * MSKKafkaVersion = 2.7
    * PCAARN = __PCA ARN that you copied in the previous step__
    * TLSMutualAuthentication = __True__
    * VPCStack = __VPC CloudFormation Stack Name, the first stack that you created__
* Once the cluster is up and running proceed with the next steps
    <hr> 

## Cloud9 Bastion Host steps
* Open Cloud 9 Terminal and upload ssh keypair file in cloud9 (upload local files), that you created in Account A
* Change keypair.pem file permissions

        chmod 0400 <keypair.pem>
* Create ssh scripts to be able to ssh into Prometheus, Producer and Consumer instances
    * producer.sh

            ssh -i <ec2 keypair.pem> ec2-user@<private ip of Producer EC2 instance>
    * consumer.sh

            ssh -i <ec2 keypair.pem> ec2-user@<private ip of Consumer EC2 instance>
    * Make shell script executable

            chmod +x producer.sh consumer.sh    
<hr>

## Producer EC2 instance Basic Setup
* From Cloud9 terminal ssh into Prometheus EC2 instance

        ./producer.sh
* Once Sshed into Prometheus instance, configure region to `ap-southeast-2`

        aws configure        

* Set the environment variables, follow the commands/instructions given below bash_profile
    * CLUSTER_ARN = AWS MSK Cluster ARN
    * BS = AWS MSK Cluster Broker nodes endpoint
    * ZK = AWS MSK Cluster Zookeeper nodes endpoint

            ~/ vim .bash_profile
        
            #append ~/kafka/bin to PATH
            #PATH variable should look something like
            
            PATH=$PATH:$HOME/.local/bin:$HOME/bin:~/kafka/bin
            export PATH

            export CLUSTER_ARN=`aws kafka list-clusters|grep ClusterArn|cut -d ':' -f 2-|cut -d ',' -f 1 | sed -e 's/\"//g'`
            
            export BS=`aws kafka get-bootstrap-brokers --cluster-arn $CLUSTER_ARN|grep BootstrapBrokerString|grep 9092| cut -d ':' -f 2- | sed -e 's/\"//g' | sed -e 's/,$//'`
            
            export BS_TLS=`aws kafka get-bootstrap-brokers --cluster-arn $CLUSTER_ARN|grep BootstrapBrokerStringTls|grep 9094| cut -d ':' -f 2- | sed -e 's/\"//g' | sed -e 's/,$//'`
            
            export ZK=`aws kafka describe-cluster --cluster-arn $CLUSTER_ARN|grep ZookeeperConnectString|grep -v Tls|cut -d ':' -f 2-|sed 's/,$//g'|sed -e 's/\"//g'`
            
            export ZK_TLS=`aws kafka describe-cluster --cluster-arn $CLUSTER_ARN|grep ZookeeperConnectStringTls|grep Tls|cut -d ':' -f 2-|sed 's/,$//g'|sed -e 's/\"//g'`
            
            # save changes and exit .bash_profile
        
            # load environment variables in profile
            ~/ source .bash_profile
        
            # verify environment variables values
            
            echo $CLUSTER_ARN
        
            echo $BS
        
            echo $ZK
<hr>

## Consumer EC2 instance Basic Setup

* From Cloud9 terminal ssh into Prometheus EC2 instance

        ./consumer.sh
* Once Sshed into Prometheus instance, configure region to `ap-southeast-2`

        aws configure        

* Set the environment variables, follow the commands/instructions given below bash_profile
    * CLUSTER_ARN = AWS MSK Cluster ARN
    * BS = AWS MSK Cluster Broker nodes endpoint
    * ZK = AWS MSK Cluster Zookeeper nodes endpoint

            ~/ vim .bash_profile
            
            #append ~/kafka/bin to PATH
            #PATH variable should look something like
            
            PATH=$PATH:$HOME/.local/bin:$HOME/bin:~/kafka/bin
            export PATH

            export CLUSTER_ARN=`aws kafka list-clusters|grep ClusterArn|cut -d ':' -f 2-|cut -d ',' -f 1 | sed -e 's/\"//g'`
            
            export BS=`aws kafka get-bootstrap-brokers --cluster-arn $CLUSTER_ARN|grep BootstrapBrokerString|grep 9092| cut -d ':' -f 2- | sed -e 's/\"//g' | sed -e 's/,$//'`
            
            export BS_TLS=`aws kafka get-bootstrap-brokers --cluster-arn $CLUSTER_ARN|grep BootstrapBrokerStringTls|grep 9094| cut -d ':' -f 2- | sed -e 's/\"//g' | sed -e 's/,$//'`
            
            export ZK=`aws kafka describe-cluster --cluster-arn $CLUSTER_ARN|grep ZookeeperConnectString|grep -v Tls|cut -d ':' -f 2-|sed 's/,$//g'|sed -e 's/\"//g'`
            
            export ZK_TLS=`aws kafka describe-cluster --cluster-arn $CLUSTER_ARN|grep ZookeeperConnectStringTls|grep Tls|cut -d ':' -f 2-|sed 's/,$//g'|sed -e 's/\"//g'`

            # save changes and exit .bash_profile

            # load environment variables in profile
            ~/ source .bash_profile

            # verify environment variables values
             
            echo $CLUSTER_ARN

            echo $BS

            echo $ZK
<hr>
!! WORK IN PROGRESS
                    
    aws kafka list-clusters | jq ".ClusterInfoList[0].ZookeeperConnectString"

    
  
