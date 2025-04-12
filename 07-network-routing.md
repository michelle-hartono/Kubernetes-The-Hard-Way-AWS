# Provisioning Pod Network Routes
- Pods scheduled to a node receive an IP address from the node's Pod CIDR range. 
- At this point pods can not communicate with other pods running on different nodes due to missing network routes. 
- To Do: create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address. 

## The Routing Table and routes
In this section you will gather the information required to create routes in the kubernetes-the-hard-way VPC network and use that to create route table entries.

1. Print the internal IP address and Pod CIDR range for each worker instance and create route table entries:
    ```
    for instance in worker-0 worker-1 worker-2; do
      instance_id_ip="$(aws ec2 describe-instances \
        --filters "Name=tag:Name,Values=${instance}" \
        --output text --query 'Reservations[].Instances[].[InstanceId,PrivateIpAddress]')"
      instance_id="$(echo "${instance_id_ip}" | cut -f1)"
      instance_ip="$(echo "${instance_id_ip}" | cut -f2)"
      pod_cidr="$(aws ec2 describe-instance-attribute \
        --instance-id "${instance_id}" \
        --attribute userData \
        --output text --query 'UserData.Value' \
        | base64 --decode | tr "|" "\n" | grep "^pod-cidr" | cut -d'=' -f2)"
      echo "${instance_ip} ${pod_cidr}"

      aws ec2 create-route \
        --route-table-id "${ROUTE_TABLE_ID}" \
        --destination-cidr-block "${pod_cidr}" \
        --instance-id "${instance_id}"
    done
    ```
    Output
    ```
    10.0.1.20 10.200.0.0/24
    {
        "Return": true
    }
    10.0.1.21 10.200.1.0/24
    {
        "Return": true
    }
    10.0.1.22 10.200.2.0/24
    {
        "Return": true
    }
    ```
2. Validate network routes for each worker instance:
  ```
  aws ec2 describe-route-tables \
    --route-table-ids "${ROUTE_TABLE_ID}" \
    --query 'RouteTables[].Routes'
  ```
  Output
  ```
  [
      [
          {
              "DestinationCidrBlock": "10.200.0.0/24",
              "InstanceId": "i-038331bdf3a57b04c",
              "InstanceOwnerId": "092450162753",
              "NetworkInterfaceId": "eni-02525e5b0b1003cca",
              "Origin": "CreateRoute",
              "State": "active"
          },
          {
              "DestinationCidrBlock": "10.200.1.0/24",
              "InstanceId": "i-0d800097fe9562ea6",
              "InstanceOwnerId": "092450162753",
              "NetworkInterfaceId": "eni-0b59c3cff8b63f2a6",
              "Origin": "CreateRoute",
              "State": "active"
          },
          {
              "DestinationCidrBlock": "10.200.2.0/24",
              "InstanceId": "i-0fe7053951f613911",
              "InstanceOwnerId": "092450162753",
              "NetworkInterfaceId": "eni-06266a77217d10a66",
              "Origin": "CreateRoute",
              "State": "active"
          },
          {
              "DestinationCidrBlock": "10.0.0.0/16",
              "GatewayId": "local",
              "Origin": "CreateRouteTable",
              "State": "active"
          },
          {
              "DestinationCidrBlock": "0.0.0.0/0",
              "GatewayId": "igw-0c578fc0f9f1ed4ef",
              "Origin": "CreateRoute",
              "State": "active"
          }
      ]
  ]
  ```

  