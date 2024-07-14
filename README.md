### Updated User Data Script

```
#!/bin/bash
# Update package lists
apt-get update

# Install NGINX
apt-get install -y nginx

# Install AWS CLI
apt-get install -y awscli

# Configure AWS CLI with IAM role (assuming the instance has an IAM role with the necessary permissions)
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)

# Get the IMDSv2 token
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`

# Get the instance ID
INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/instance-id)

# Get the availability zone
AVAILABILITY_ZONE=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/placement/availability-zone)

# Get the instance name (requires IAM role with ec2:DescribeTags permission)
INSTANCE_NAME=$(aws ec2 describe-tags --region $REGION --filters "Name=resource-id,Values=$INSTANCE_ID" "Name=key,Values=Name" --query "Tags[0].Value" --output text)

# Get the CPU load
CPU_LOAD=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1"%"}')

# Create an HTML page that shows the instance metadata and CPU load
cat <<EOF > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head>
    <title>Server Info</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; padding: 50px; }
        table { margin: auto; border-collapse: collapse; }
        th, td { border: 1px solid #ddd; padding: 8px; }
        th { background-color: #f2f2f2; }
    </style>
</head>
<body>
    <h1>Server Info</h1>
    <table>
        <tr>
            <th>Meta-Data</th>
            <th>Value</th>
        </tr>
        <tr>
            <td>InstanceId</td>
            <td>$INSTANCE_ID</td>
        </tr>
        <tr>
            <td>Availability Zone</td>
            <td>$AVAILABILITY_ZONE</td>
        </tr>
    
    </table>
    <h2>Current CPU Load: $CPU_LOAD</h2>
</body>
</html>
EOF

# Restart NGINX to apply changes
systemctl restart nginx

```

### Instructions to Use the Script in User Data

1. **Launch 2 EC2 instances**:

   - Go to the EC2 Dashboard.
   - Click "Launch Instance."
   - Choose an AMI and instance type.
   - In the "User data" field, paste the above script.
   - Specify the number of instances as 2 to check the load balancer between two servers.
   - Complete the remaining steps and launch the instances.
   - Rename the instances to `server1` and `server2`.

2. **Add the script to user data**:

   - In the "Configure Instance" section, scroll down to "Advanced Details."
   - In the "User data" field, paste the above script.

3. **Configure security groups**:

   - Ensure that the security group associated with your instances allows inbound HTTP traffic on port 80.

4. **Launch the instances**:
   - Complete the remaining steps and launch the instances.

### Setting Up an Application Load Balancer (ALB)

To distribute traffic across multiple EC2 instances, follow these steps:

1. **Launch Multiple EC2 Instances**:

   - Ensure each instance uses the user data script provided earlier.

2. **Configure Security Groups**:

   - Allow inbound traffic on port 80 (HTTP) for the instances.

3. **Create a Target Group**:

   - Go to the EC2 Dashboard.
   - Navigate to Load Balancers > Target Groups.
   - Click "Create target group."
   - Choose "Instances" as the target type.
   - Specify details such as name, protocol (HTTP), port (80), and VPC.
   - Choose or create a security group `server_SG` that allows inbound HTTP traffic on port 80.
   - Register your EC2 instances with the target group.
   - Click "Create target group."

4. **Create an Application Load Balancer**:
   - Go to Load Balancers > Create Load Balancer.
   - Choose "Application Load Balancer."
   - Configure the load balancer with a name, scheme (internet-facing), and availability zones.
   - Choose the security group `server_SG` that allows inbound HTTP traffic on port 80.
   - Specify the target group created earlier.
   - Click "Create load balancer."

### Testing the Load Balancer

- Obtain the DNS name of your load balancer from the Load Balancers section in the EC2 Dashboard.
- Access the DNS name in a web browser. The load balancer will distribute traffic between your EC2 instances, showing the server info page from each instance.

Now, go to your load balancer's DNS name in your browser, and when it loads, refresh the page to see the instance name, instance ID, availability zone, public IP, server load, and CPU load. You have successfully set up an application load balancer.
