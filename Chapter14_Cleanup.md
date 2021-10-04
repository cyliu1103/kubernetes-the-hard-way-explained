# Cleanup

This is a cleanup section which deletes all resources we created. This is also a wrap up so we can examine what resources are used:

- EC2 instances

We have 6 EC2 instances, 3 are running Control Plane, 3 are Worker Nodes

- Application Load Balancer and Target Group

We have 1 application load balancer. The DNS name of this ALB is used as an endpoint to API Servers. We then registered 3 Control Plane nodes as the instances of the Target Group

- A security group

This security group contains rules that applies to Control Plane and Worker Nodes

- A routing table

It includes 3 kind of routes: route to the subnet, route to each Pods and route to 0.0.0.0/0 

- An internet gateway

The Internet gateway works as a default route to all traffic to the Internet

- Subnets and a VPC

All of these resources are created in this VPC with one subnet
