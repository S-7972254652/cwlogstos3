# cwlogstos3
To store Memory and Disk Utilization logs to s3 bucket
Intro 
Monitoring the memory and disk utilization of EC2 instances is crucial for maintaining their health and ensuring optimal performance. With CloudWatch Agent, you can collect metrics and logs from your EC2 instance and monitor system .
Step 1: Create an IAM Role
•	Create an IAM Role selecting ec2 service 
•	Attach “AmazonSSMFullAccess “& “CloudWatchFullAccess Policy “ to it ,
•	Lastly give name for role 
Step 2: Step 2: Create a Parameter in Systems Manger 
•	Go into system Manager  go to parameter Create New Parameter
•	Give name for parameter “/alarm/AWS-CWAgentLinConfig ”
•	Store the JSON file in value section, which specifies the metrics to be collected, such as memory and disk utilization
{
	"metrics": {
		"append_dimensions": {
			"InstanceId": "${aws:InstanceId}"
		},
		"metrics_collected": {
			"mem": {
				"measurement": [
					"mem_used_percent"
				],
				"metrics_collection_interval": 60
			},
            "disk": {
				"measurement": [
                     "disk_used_percent"
				],
				"metrics_collection_interval": 60
			}
		}
}

•	Choose the appropriate type (String) and data type (Text)

•	Create Parameter
Step 3: Create an EC2 Instance, Attach the role created in Step 1

•	Create EC2 instance and attach IAM Role 

•	Add user data in advance section while creating instance 

•	This script download cloudwatch agent package  Extract package install Cloudwatch agent & Fetch config from system manager parameter and apply it to agent 

#!/bin/bash
wget https://s3.amazonaws.com/amazoncloudwatch- agent/linux/amd64/latest/AmazonCloudWatchAgent.zip
unzip AmazonCloudWatchAgent.zip
sudo ./install.sh
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c ssm:/alarm/AWS-CWAgentLinConfig -s

NOTE:- Highlighted above in user data script should be same as that we provide as name of parameter/path ie in step 2 


•	Do SSH to instance and check CWAgent installed or not using below command

sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status

Step 4: Create Cloudwatch Alarm’

•	Go to cloudwatch section

•	Create Alarm  Select metrics  check CWAgent visible or not click on CWAgent  in CWAgent below pic 

Note:-
•	  Instanceid,device,fstype,path id for Disk Monitoring of ec2
•	  InstanceId – for Memory(RAM) Monitoring 

•	Select instance id for u want to monitor(select volume) Add SNS  create Alarm
User data
#!/bin/bash
wget https://s3.amazonaws.com/amazoncloudwatch-agent/linux/amd64/latest/AmazonCloudWatchAgent.zip
unzip AmazonCloudWatchAgent.zip
sudo ./install.sh
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c ssm:/alarm/AWS-CWAgentLinConfig -s
Commands used for Disk & Memory Monitor
To increase space for testing :  
fio --name=writefile --size=9000M --filesize=9000M --filename=testfile --bs=1M --nrfiles=1 --direct=1 --sync=0 --randrepeat=0 --rw=write --refill_buffers --end_fsync=1 --iodepth=200 --ioengine=libaio
To create LV use pvcreate, vgcreate, lvcreate 
Command to make lv as filesystem : - sudo mkfs.ext4 /dev/volumegroup/lvname
Command to mount at /mnt point: -  sudo mount /dev/ volumegroup / lvname
 /mnt

FOR MEMORY : 
Install stress on machine :    yum install stress
Increase load on RAM :     stress --vm 1 --vm-bytes 600M
To check RAM details :     free –h
To check stress which are running :    ps aux | grep stress
To kill all stress:   kill <pid>  /  To check pid :  pidof stress

Troubleshoot: If we encounter error cwagent stopped
•	If cwagent is stopped unable to start or shown status as stopped 
•	Sol: Then use this command sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c ssm:/alarm/AWS-CWAgentLinConfig –s
•	Change highlighted with ur actual parameter name it will configure it
•	Nxt run start command to start the cwagent:   sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a start
•	Then check status command to check  running and configured 
To Store the Disk and Memory utilization Log to S3 Bucket
SSH to EC2 instance
Install and Configure CloudWatch Agent
•	sudo yum install -y amazon-cloudwatch-agent
Create the configuration file:
•	sudo vi /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
Add the following configuration to log disk and memory utilization:
	{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "metrics": {
    "namespace": "CustomMetrics",
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}"
    },
    "metrics_collected": {
      "disk": {
        "measurement": [
          {"name": "used_percent", "rename": "DiskUsedPercent"}
        ],
        "resources": [
          "/"
        ],
        "ignore_file_system_types": [
          "sysfs", "devtmpfs"
        ]
      },
      "mem": {
        "measurement": [
          {"name": "mem_used_percent", "rename": "MemoryUsedPercent"}
        ]
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/disk_memory_utilization.log",
            "log_group_name": "disk_memory_utilization",
            "log_stream_name": "{instance_id}",
            "timestamp_format": "%Y-%m-%d %H:%M:%S"
          }
        ]
      }
    }
  }
}
Create a Custom Script to Log Metrics
•	Create a script to log disk and memory utilization:
•	sudo nano /usr/local/bin/log_disk_memory_utilization.sh


Add the following script content:
#!/bin/bash
while true; do
  TIMESTAMP=$(date +"%Y-%m-%d %H:%M:%S")
  DISK_USAGE=$(df -h / | grep / | awk '{ print $5 }')
  MEMORY_USAGE=$(free | grep Mem | awk '{print $3/$2 * 100.0}')
  echo "$TIMESTAMP Disk Used: $DISK_USAGE, Memory Used: $MEMORY_USAGE%" >> /var/log/disk_memory_utilization.log
  sleep 60
done
Make the script executable:
•	sudo chmod +x /usr/local/bin/log_disk_memory_utilization.sh
Run the script in the background:
•	nohup /usr/local/bin/log_disk_memory_utilization.sh &
Start the CloudWatch Agent
•	Apply the configuration:
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
  -s
Verify the agent is running:
•	sudo systemctl status amazon-cloudwatch-agent
View Logs in CloudWatch
The Logs are stored inside  disk_memory_utilization





Create the S3 Bucket

Modify the permission of S3 Bucket
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "logs.ap-south-1.amazonaws.com"
            },
            "Action": "s3:GetBucketAcl",
            "Resource": "arn:aws:s3:::s3-logs-bucket-pd",
            "Condition": {
                "StringEquals": {
                    "aws:SourceAccount": "678351729656"
                },
                "ArnLike": {
                    "aws:SourceArn": "arn:aws:logs:ap-south-1:678351729656:log-group:disk_memory_utilization:*"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "logs.ap-south-1.amazonaws.com"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::s3-logs-bucket-pd/*",
            "Condition": {
                "StringEquals": {
                    "s3:x-amz-acl": "bucket-owner-full-control",
                    "aws:SourceAccount": "678351729656"
                },
                "ArnLike": {
                    "aws:SourceArn": "arn:aws:logs:ap-south-1:678351729656:log-group:disk_memory_utilization:*"
                }
            }
        }
    ]
}
Replace bucket name, region and  AWS account id
Open log group from AWS Cloudwatch
 
Select Log-group & click on action button 
 
Click on export data to Amazon s3 
Select the date from which you want to export log 
Choose bucket in which you want to save logs
Open the file andwe got to see the  logs are stored in file
Reference :- 
•	https://github.com/yeshwanthlm/EC2-Memory-Disk-Monitoring?tab=readme-ov-file#ec2-memory-disk-monitoring
•	https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent-New-Instances-CloudFormation.html
•	https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html
	


