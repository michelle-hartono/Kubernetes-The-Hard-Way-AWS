# Environment Setup

## 💻 Local Environment Setup (macOS)

### Amazon Web Services CLI

- Install the AWS CLI using Homebrew:
    ```
    brew install awscli
    ```
- Verify the installation by checking the AWS CLI version:
    ```
    aws --version
    ```

### Set Up IAM User and Permissions
#### If you don't have an AWS IAM user yet:
1. Navigate to the IAM Console.
    - Console Home > All services > IAM > Users > Create User
2. Create a new IAM user:
    - Access type: Select Programmatic access to allow CLI access 
    - Permissions: Create a new group with `AdministratorAccess`
    - Group name: `Kubernetes-Admins `
3. Save Access Keys
    - After creating the user, you'll be shown the Access Key ID and Secret Access Key. 
    - Save these keys securely, as you won’t be able to view them again.

#### If you already have an AWS IAM user:
1. Configure AWS CLI
    - Run the AWS CLI configuration command to set your default credentials and region:
        ```
        aws configure
        ```
    - You will be prompted to enter the following details:
        ```
        AWS Access Key ID: <your-access-key-id>
        AWS Secret Access Key: <your-secret-access-key>
        Default region name: ap-northeast-1 
        Default output format: json
        ```
#### Notes:
- This will save your configuration in `~/.aws/credentials` and` ~/.aws/config`.
- The `json` format for output is typically the best for scripting, and `ap-northeast-1` (Tokyo) is ideal for Taiwan.
- If you need to switch regions later, you can simply run `aws configure` again and update the region.

### Running Commands in Parallel with tmux

#### What is tmux?
tmux can be used to run commands on multiple compute instances at the same time.

#### Installation
1. Install tmux using Homebrew:
    ```
    brew install tmux
    ```
2. Verify the installation by checking the tmux version:
    ```
    tmux -V
    ```
3. Once tmux is installed, you can start it by simply running:
    ```
    tmux
    ```

#### Features
1. Split the Terminal Window
    - Vertical Split: Press Ctrl + b, then Shift + 5 (%)
    - Horizontal Split: Press Ctrl + b, then " (double quote) 
2. Navigating Between Panes
    - Ctrl + b, then use the arrow keys (Up, Down, Left, Right) 
3. Detaching and Reattaching to tmux Session
    - Ctrl + b, then d. This keeps the session running in the background.
4. To reattach to the tmux session, run:
    ```
    tmux attach-session
    ```
5. Synchronize-panes: Run the same command in all panes at the same time 
    - Enable: Ctrl + b, then type :setw synchronize-panes on and hit Enter. 
    - Disable: Ctrl + b, type :setw synchronize-panes off, and hit Enter. 
6. Closing Panes
    - Type `exit` or press Ctrl + d within that pane. 

### Installing The Client Tools 
#### cfssl, cfssljson, kubectl

1. Install cfssl and cfssljson
    - Download and install cfssl and cfssljson using Homebrew:
        ```
        brew install cfssl
        ```
    - Ensure that cfssl and cfssljson version 1.6.4 or higher is installed:
        ```
        cfssl version
        ```
    - Example Output
        ```
        Version: 1.6.5 
        Runtime: go1.23.1 
        ```

3. Install kubectl
    - Download and install kubectl using Homebrew:
        ```
        brew install kubectl
        ```

4. Verify kubectl version
    - Ensure kubectl version 1.28.3 or above is installed:
        ```
        kubectl version --client
        ```
    - Example Output:
        ```
        Client Version: v1.30.5
        Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
        ```

## 🌐 AWS Infrastructure Setup
In this section, we are Provisioning Compute Resources. We will setup the required networking infrastruture and nodes required for the cluster.

### Networking

#### VPC
1. Create the VPC
    ```
    VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --output text --query 'Vpc.VpcId')
    ```
    - This command creates a new VPC with a CIDR block of 10.0.0.0/16, which gives you a private network range of 65536 IP addresses. We also specify the output format and query for the VPC ID.
    - Command breakdown:
        - `--cidr-block 10.0.0.0/16`: Defines the IP address range for the VPC.
        - `--output text`: Specifies the output format, which will return the VPC ID.
        - `--query 'Vpc.VpcId'`: Extracts only the VPC ID from the output. 

2. List all VPC
    - This command will return a list of VPC IDs in your account. 
        ```
        aws ec2 describe-vpcs --query 'Vpcs[*].VpcId' --output text
        ```
    - Get more details about a specific VPC:
        ```
        aws ec2 describe-vpcs --vpc-ids <Your-VPC-ID>
        ```
    - List VPCs with tags and names:
        ```
        aws ec2 describe-vpcs --query 'Vpcs[*].{ID:VpcId,Name:Tags[?Key==`Name`].Value|[0]}' --output table
        ```

3. Tag the VPC with a name (for easier identification)
    ```
    aws ec2 create-tags --resources ${VPC_ID} --tags Key=Name,Value=kubernetes-the-hard-way
    ```
    - Command breakdown:
        - `--resources ${VPC_ID}`: Applies the tag to the VPC you just created.
        - `--tags Key=Name,Value=kubernetes-the-hard-way`: Adds a tag with the key Name and value kubernetes-the-hard-way to the VPC.

4. Enable DNS support in the VPC
    ```
    aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-support '{"Value": true}'
    ```
    - This command enables DNS support within your VPC, which is necessary for Kubernetes services like DNS-based service discovery to function properly.
    - Command breakdown:
        - `--vpc-id ${VPC_ID}`: Specifies the VPC that you want to modify. 
        - `--enable-dns-support '{"Value": true}'`: Enables DNS support for the VPC. 

5. Enable DNS hostnames in the VPC
    ```
    aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-hostnames '{"Value": true}'
    ```
    - Enabling DNS hostnames for VPC is important for resolving hostnames within the VPC (e.g., instances will have public DNS names if they have a public IP address).
    - Command breakdown:
        - `--enable-dns-hostnames '{"Value": true}'`: Enables DNS hostnames, allowing instances in the VPC to have DNS names assigned. 

#### Subnet
Subnet is a range of IP addresses in your VPC that can be used to launch EC2 instances. 
1. Creating a subnet within a VPC on AWS 
    ```
    SUBNET_ID=$(aws ec2 create-subnet \
      --vpc-id ${VPC_ID} \
      --cidr-block 10.0.1.0/24 \
      --output text --query 'Subnet.SubnetId')
    ```
    - Command breakdown: 
        - `aws ec2 create-subnet`
            - Creates a new subnet within an existing VPC in your AWS account. 
        - `--vpc-id ${VPC_ID}`
            - Specifies the VPC in which the subnet should be created. 
            - Placed in the` ${VPC_ID}`, a variable that contains the ID of the VPC.
            - The VPC ID can be set in a previous step or fetched dynamically. 
        - `--cidr-block 10.0.1.0/24`
            - Defines the IP address range for the subnet. 
            - The CIDR block 10.0.1.0/24 means that the subnet will have 256 IP addresses (from 10.0.1.0 to 10.0.1.255)
            - .0/24 represents the network mask, allowing for up to 256 addresses in the subnet.
        - `--output text`
            - Specifies the output format. 
            - Returns the output as plain text. 
        - `--query 'Subnet.SubnetId'`
            - Filters the result to only show the SubnetId. 
            - Querying 'Subnet.SubnetId' shows the ID of the newly created subnet. 
2. Tagging the subnet for easier identification
    ```
    aws ec2 create-tags --resources ${SUBNET_ID} --tags Key=Name,Value=kubernetes
    ```
3. Listing subnets
    - List all subnets in your account
        ```
        aws ec2 describe-subnets --output table
        ```
    - List subnets within a VPC
        ```
        aws ec2 describe-subnets --filters "Name=vpc-id,Values=${VPC_ID}" --output table
        ```
    - Sample
        sample_subnet.png

#### Internet Gateway
An Internet Gateway is required for your VPC to allow resources (e.g., EC2 instances) to communicate with the internet.

1. Create an Internet Gateway
    ```
    INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway --output text --query 'InternetGateway.InternetGatewayId')
    ```
    - Creates a new Internet Gateway in your AWS account and stores it in INTERNET_GATEWAY_ID variable. 
    - Command breakdown:
        - `aws ec2 create-internet-gateway`: AWS CLI command that creates a new Internet Gateway.
        - `--output text`: Specifies the format for the command's output. 
        - `--query 'InternetGateway.InternetGatewayId'`: This uses JMESPath (a query language for JSON) to extract just the InternetGatewayId from the returned JSON response. This will be the unique identifier for the newly created Internet Gateway.
2. Tag the Internet Gateway
    ```
    aws ec2 create-tags --resources ${INTERNET_GATEWAY_ID} --tags Key=Name,Value=kubernetes
    ```
    - This command tags the newly created Internet Gateway with the name kubernetes for easier identification.
    - Command breakdown: 
        - `aws ec2 create-tags`: AWS CLI command to create tags for AWS resources.
        - `--resources ${INTERNET_GATEWAY_ID}`: Specifies the resource (the Internet Gateway) to tag. 
        - `--tags Key=Name,Value=kubernetes`: Adds a tag with the key Name and value kubernetes to the Internet Gateway. 
3. Attach the Internet Gateway to the VPC:
    ```
    aws ec2 attach-internet-gateway --internet-gateway-id ${INTERNET_GATEWAY_ID} --vpc-id ${VPC_ID}
    ```
    - This command attaches the created Internet Gateway to your VPC. This allows resources in the VPC to access the internet.
    - Command breakdown:
        - `aws ec2 attach-internet-gateway`: AWS CLI command to attach an Internet Gateway to a VPC. 
        - `--internet-gateway-id ${INTERNET_GATEWAY_ID}`: Specifies the ID of the Internet Gateway you want to attach. 
        - `--vpc-id ${VPC_ID}`: Specifies the ID of the VPC to which you want to attach the Internet Gateway. 
4. View the Internet Gateway
    ```
    aws ec2 describe-internet-gateways \
        --query 'InternetGateways[*].{ID:InternetGatewayId, VpcAttachments:Attachments[*].VpcId}' \
        --output table
    ```
    - Sample: internet_gateways.png
#### Route Tables
A route table contains rules (routes) that determine how traffic is directed between different subnets or to the internet.

1. Create a route table in your VPC
    ```
    ROUTE_TABLE_ID=$(aws ec2 create-route-table --vpc-id ${VPC_ID} --output text --query 'RouteTable.RouteTableId')
    ```
    - Command breakdown:
        - `aws ec2 create-route-table`: This AWS CLI command creates a new route table for the specified VPC.
        - `--vpc-id ${VPC_ID}`: This option specifies the VPC to which the route table belongs. ${VPC_ID} is the variable that holds the ID of the VPC.
        -` --output text`: Specifies the format of the output, which will be plain text. 
        - `--query 'RouteTable.RouteTableId'`: Uses a JMESPath query to extract only the RouteTableId from the JSON response. This is the unique identifier for the route table. 
2. Tag the route table
    ```
    aws ec2 create-tags --resources ${ROUTE_TABLE_ID} --tags Key=Name,Value=kubernetes
    ```
    - Tags the newly created route table with the name kubernetes for easier identification.
    - Command breakdown: 
        - `aws ec2 create-tags`: This is the AWS CLI command to create tags for AWS resources. 
        - `--resources ${ROUTE_TABLE_ID}`: Specifies the resource (the route table) to tag. 
        - `--tags Key=Name,Value=kubernetes`: Adds a tag with the key Name and value kubernetes to the route table. 
3. Associate the route table with a subnet
    ```
    aws ec2 associate-route-table --route-table-id ${ROUTE_TABLE_ID} --subnet-id ${SUBNET_ID} 
    ```
    - Command breakdown:
        - `aws ec2 associate-route-table`: Associates a route table with a subnet, so that the subnet's traffic is directed according to the routes in that table. 
        - `--route-table-id ${ROUTE_TABLE_ID}`: Specifies the ID of the route table you want to associate with the subnet. ${ROUTE_TABLE_ID} is the variable holding the route table ID from the previous steps. 
        - `--subnet-id ${SUBNET_ID}`: Specifies the ID of the subnet to which the route table will be associated. ${SUBNET_ID} is the variable holding the ID of the subnet that was created earlier. 
4. Create a route to the Internet Gateway 
    ```
    aws ec2 create-route --route-table-id ${ROUTE_TABLE_ID} --destination-cidr-block 0.0.0.0/0 --gateway-id ${INTERNET_GATEWAY_ID}
    ```
    - Connects our private network (the VPC) to the public internet. 
    - When we create a route table for a subnet in a VPC, it doesn't automatically know how to reach the internet. 
    - Command breakdown:
        - `aws ec2 create-route`: Creates a new route in the specified route table. 
        - `--route-table-id ${ROUTE_TABLE_ID}`: Specifies the ID of the route table in which the route will be added. ${ROUTE_TABLE_ID} is the variable holding the route table ID. 
        - `--destination-cidr-block 0.0.0.0/0`: This specifies the destination CIDR block for the route. 0.0.0.0/0 means the route applies to all IP addresses, i.e., all outbound traffic. 
        - `--gateway-id ${INTERNET_GATEWAY_ID}`: This option specifies the ID of the Internet Gateway through which traffic will be routed. ${INTERNET_GATEWAY_ID} is the variable holding the ID of the Internet Gateway that was created and tagged earlier.

#### Security Groups (Firewall Rules) 
1. Create a security group
    ```
    SECURITY_GROUP_ID=$(aws ec2 create-security-group \
        --group-name kubernetes \
        --description "Kubernetes security group" \
        --vpc-id ${VPC_ID} \
        --output text --query 'GroupId')
    ```
    - Command breakdown:
        - `aws ec2 create-security-group`: Creates a new security group. 
        - `--group-name kubernetes`: Specifies the name of the security group as "kubernetes". 
        - `--description "Kubernetes security group"`: Adds a description to the security group for easier identification. 
        - `--vpc-id ${VPC_ID}`: Specifies the ID of the VPC in which the security group will be created. ${VPC_ID} is the variable holding the VPC ID. 
        - `--output text`: Specifies the output format as plain text. 
        - `--query 'GroupId'`: Extracts only the GroupId (the security group ID) from the command output. 
        - `SECURITY_GROUP_ID=$(...)`: This stores the security group ID in a variable called SECURITY_GROUP_ID.
2. Tag the security group
    ```
    aws ec2 create-tags --resources ${SECURITY_GROUP_ID} --tags Key=Name,Value=kubernetes 
    ```
    - Command breakdown: 
        - `aws ec2 create-tags`: AWS CLI command to create tags for AWS resources.
        - `--resources ${SECURITY_GROUP_ID}`: Specifies the resource (the security group) to tag. ${SECURITY_GROUP_ID} holds the security group ID. 
        - `--tags Key=Name,Value=kubernetes`: Creates a tag with the key Name and the value kubernetes for the security group.
3. Authorize Ingress
    i. Internal Subnet (CIDR: 10.0.0.0/16)
    ```
    aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol all --cidr 10.0.0.0/16
    ```
    ii. Internal Subnet (CIDR: 10.200.0.0/16)
    ```
    aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol all --cidr 10.200.0.0/16
    ```
    iii. SSH
    ```
    aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 22 --cidr 0.0.0.0/0
    ```
    iv. Kubernetes API Server Access
    ```
    aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 6443 --cidr 0.0.0.0/0
    ```
    v. HTTPS
    ```
    aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 443 --cidr 0.0.0.0/0
    ```
    vi. ICMP
    ```
    aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol icmp --port -1 --cidr 0.0.0.0/0
    ```
    Note: `CIDRIP: 0.0.0.0/0` means: Allow access from anywhere on the internet, regardless of IP address. 

#### Kubernetes Public Access - Create a Network Load Balancer
This setup allows external users or tools (like kubectl) to reach the Kubernetes control plane securely and reliably via the NLB, while maintaining raw TCP communication.

1. Create a Network Load Balancer
    ```
    LOAD_BALANCER_ARN=$(aws elbv2 create-load-balancer \
      --name kubernetes \
      --subnets ${SUBNET_ID} \
      --scheme internet-facing \
      --type network \
      --output text --query 'LoadBalancers[].LoadBalancerArn')
    ```
    - Publicly exposed entry point for Kubernetes API
    - Creates a Network Load Balancer (NLB) named kubernetes, in the specified public subnet, and stores its ARN in LOAD_BALANCER_ARN.
    - Command breakdown:
        - `--name kubernetes`: Name of the load balancer. 
        - `--subnets ${SUBNET_ID}`: Subnet(s) where the load balancer will be placed. Usually a public subnet to be accessible from the internet. 
        - `--scheme internet-facing`: Makes the load balancer publicly accessible. 
        - `--type network`: Specifies the load balancer type as Network Load Balancer (NLB) — low latency and supports static IPs. 
        - `--output text --query 'LoadBalancers[].LoadBalancerArn'`: Extracts and returns just the ARN of the newly created load balancer.
2. Create a Target Group
    ```
    TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
      --name kubernetes \
      --protocol TCP \
      --port 6443 \
      --vpc-id ${VPC_ID} \
      --target-type ip \
      --output text --query 'TargetGroups[].TargetGroupArn')
    ```
    - Points to IPs of control plane nodes on port 6443
    - Creates a target group for the load balancer, listening on TCP port 6443 (used by Kubernetes API servers), and stores its ARN in TARGET_GROUP_ARN.
    - Command breakdown:
        - `--protocol TCP`: Specifies the protocol for routing traffic — here it's raw TCP. 
        - `--port 6443`: The port Kubernetes API servers listen on.
        - `--vpc-id ${VPC_ID}`: VPC in which the targets reside. 
        - `--target-type ip`: Targets are IP addresses (instead of EC2 instances). Needed when targeting Kubernetes pods or nodes directly by IP.
        - `--output text --query 'TargetGroups[].TargetGroupArn'`: Outputs just the Target Group ARN (Amazon Resource Name). 
3. Register Targets
    ```
    aws elbv2 register-targets --target-group-arn ${TARGET_GROUP_ARN} --targets Id=10.0.1.1{0,1,2}
    ```
    - Internal IPs of Kubernetes control nodes
    - Registers 3 IP addresses (10.0.1.10, 10.0.1.11, 10.0.1.12) as targets in the target group. 
    - Command breakdown: 
        - `--target-group-arn ${TARGET_GROUP_ARN}`: Target group created in the previous step. 
        - `--targets Id=10.0.1.1{0,1,2}`: Shell expansion shorthand. It expands into:
            ```
            Id=10.0.1.10 Id=10.0.1.11 Id=10.0.1.12
            ```
4. Create a Listener
    ```
    aws elbv2 create-listener \
      --load-balancer-arn ${LOAD_BALANCER_ARN} \
      --protocol TCP \
      --port 443 \
      --default-actions Type=forward,TargetGroupArn=${TARGET_GROUP_ARN} \
      --output text --query 'Listeners[].ListenerArn'
    ```
    - Listens on port 443, forwards to port 6443 on targets
    - Creates a listener on the load balancer that listens for incoming TCP connections on port 443, and forwards them to the target group (on port 6443).
    - Command breakdown: 
        - `--load-balancer-arn`: Associates the listener with the load balancer created earlier. 
        - `--protocol TCP`: Uses raw TCP, not HTTP/HTTPS. 
        - `--port 443`: Accepts connections on port 443 (standard for HTTPS). Even though it's not HTTPS here, this port is chosen to avoid client firewall issues. 
        - `--default-actions`: Forwards traffic to the target group. 
        - --query 'Listeners[].ListenerArn': Returns the ARN of the created listener. 
    - Output (ARN): 
        ```
        arn:aws:elasticloadbalancing:ap-northeast-1:092450162753:listener/net/kubernetes/f28a31d1f2c2881b/55e5d1c4790a0622
        ```
5. Retrieves the public DNS name of the Load Balancer
    ```
    KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
      --load-balancer-arns ${LOAD_BALANCER_ARN} \
      --output text --query 'LoadBalancers[].DNSName')
    ```
    - Command breakdown: 
        - `aws elbv2 describe-load-balancers`: This command fetches details about your specified ELBv2 (Elastic Load Balancer v2), which in this case is a Network Load Balancer (NLB). 
        - `--load-balancer-arns ${LOAD_BALANCER_ARN}`: You're narrowing the result to the NLB you created earlier (by its ARN). 
        - `--query 'LoadBalancers[].DNSName'`: This pulls out the DNS name (like kubernetes-123456.elb.amazonaws.com) from the result. This DNS name is how you'll connect to the cluster externally. 
        - `--output text`: Outputs just the raw value — no formatting or JSON, making it easy to store in a shell variable. 
        - `KUBERNETES_PUBLIC_ADDRESS=...`: Saves that DNS name to a variable for use later, like in TLS cert generation or kubeconfig setup. 

### Compute Instances
- A compute instance is a virtual server in the cloud. It lets you run apps, processes, or workloads just like you would on a physical computer — but managed and scaled by a cloud provider (like AWS, GCP, Azure, etc.). 
- In AWS, a compute instance = EC2 instance

#### Instance Image
1. Create an Instance Image
    ```
    IMAGE_ID=$(aws ec2 describe-images --owners 099720109477 \
      --output json \
      --filters \
      'Name=root-device-type,Values=ebs' \
      'Name=architecture,Values=x86_64' \
      'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server*' \
      | jq -r '.Images|sort_by(.Name)[-1]|.ImageId')
    ``` 
    - Without an image, EC2 doesn’t know what OS or software to use. 
    - Command breakdown: 
        - Search for an Ubuntu 20.04 AMI
        - Owned by Canonical (099720109477)
        - 64-bit architecture
        - With an EBS-backed root volume
        - Pick the latest version (sort_by(.Name)[-1])
        - Save the result as IMAGE_ID to use in EC2 launch

#### SSH Key Pair
1. Create an SSH key pair for secure remote access
    ```
    aws ec2 create-key-pair --key-name kubernetes --output text --query 'KeyMaterial' > kubernetes.id_rsa
    ```
    - Command breakdown:
        - Creates a new SSH key pair named kubernetes
        - Saves the private key to a file called kubernetes.id_rsa
2. Restricts file permissions
    ```
    chmod 600 kubernetes.id_rsa
    ```

#### Kubernetes Controllers
1. Set up the controller nodes as part of the Kubernetes Control Plane, using t3.micro instance
    ```
    for i in 0 1 2; do
      instance_id=$(aws ec2 run-instances \
        --associate-public-ip-address \
        --image-id ${IMAGE_ID} \
        --count 1 \
        --key-name kubernetes \
        --security-group-ids ${SECURITY_GROUP_ID} \
        --instance-type t3.micro \
        --private-ip-address 10.0.1.1${i} \
        --user-data "name=controller-${i}" \
        --subnet-id ${SUBNET_ID} \
        --block-device-mappings='{"DeviceName": "/dev/sda1", "Ebs": { "VolumeSize": 50 }, "NoDevice": "" }' \
        --output text --query 'Instances[].InstanceId')
      aws ec2 modify-instance-attribute --instance-id ${instance_id} --no-source-dest-check
      aws ec2 create-tags --resources ${instance_id} --tags "Key=Name,Value=controller-${i}"
      echo "controller-${i} created "
    done
    ```
    - These nodes are necessary for the overall management of the cluster, ensuring the state of your Kubernetes system is as intended and performing necessary tasks like scaling pods, managing deployments, etc.
    - Command breakdown:
        - Create three Controller Plane Nodes (controller-0, controller-1, controller-2), which will also be running controller components (like the controller manager). 

#### Kubernetes Workers
1. Configure worker nodes (EC2 Instances)
    ```
    for i in 0 1 2; do
      instance_id=$(aws ec2 run-instances \
        --associate-public-ip-address \
        --image-id ${IMAGE_ID} \
        --count 1 \
        --key-name kubernetes \
        --security-group-ids ${SECURITY_GROUP_ID} \
        --instance-type t3.micro \
        --private-ip-address 10.0.1.2${i} \
        --user-data "name=worker-${i}|pod-cidr=10.200.${i}.0/24" \
        --subnet-id ${SUBNET_ID} \
        --block-device-mappings='{"DeviceName": "/dev/sda1", "Ebs": { "VolumeSize": 50 }, "NoDevice": "" }' \
        --output text --query 'Instances[].InstanceId')
      aws ec2 modify-instance-attribute --instance-id ${instance_id} --no-source-dest-check
      aws ec2 create-tags --resources ${instance_id} --tags "Key=Name,Value=worker-${i}"
      echo "worker-${i} created"
    done
    ```
    - The worker nodes play a crucial role in actually running the containers (i.e., the pods) that make up your applications