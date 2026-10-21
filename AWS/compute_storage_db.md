### EC2

1. Instance type : Networking capabilities, CPU , memory.
2. AMI : OS, and pre installed software.
3. Networking : Choose the VPC, subnet, IP configuration, and security groups.
4. Storage: Choose the root disk and any additional EBS volumes.
5. User data : An optional startup script used to bootstrap the instance, such as installing packages or configuring a service.

EC2 instance families
AWS offers different instance families because workloads have different resource requirements.

Family	        Main purpose	                            Typical use case
t	            Burstable general purpose	                Small applications, test environments, bastion hosts
m	            General purpose	                            Application servers, microservices
c	            Compute optimized	                        CPU-intensive workloads, batch processing
r	            Memory optimized	                        In-memory databases, memory-heavy applications
g	            GPU accelerated	                            Graphics, inference, certain ML workloads
p	            High-performance GPU	                    Large-scale machine learning and HPC workloads


Q: What is the difference between an AMI and user data?
    An AMI provides the initial operating-system and software image. User data provides startup instructions to configure the instance after launch.

  What are Launch Templates and Auto Scaling Groups?
    A Launch Template defines how an EC2 instance should be launched, including settings such as:
    - AMI ID.
    - Instance type.
    - Security groups.
    - Storage configuration.
    - IAM instance profile.
    - User data.
    An Auto Scaling Group (ASG) manages a group of EC2 instances and can launch or terminate instances to maintain desired capacity.
    Launch Template

### Lambda

What is AWS Lambda?
AWS Lambda is a serverless compute service that runs your code without you having to manage servers.
Normally, if you want to run a Python script, you might:
1. Create an EC2 instance.
2. Install Python and the required libraries.
3. Copy your script onto the server.
4. Configure a cron job or another mechanism to run it.
5. Maintain the server, patch the OS, and monitor its resources.
With Lambda, you upload your code, configure when it should run, and AWS manages the underlying compute infrastructure.

