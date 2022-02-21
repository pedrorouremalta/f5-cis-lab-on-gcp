# f5-cis-lab-on-gcp

This is a step by step lab/demo which shows how to use F5 CIS (*Container Ingress Services*) to expose an application running on a Kubernetes cluster - more specifically on a GKE (*Google Kubernetes Engine*) cluster. We will deploy the CIS controller on the cluster and then explore the use of the custom resourses provided by the CIS to expose a demo application (composed by 5 fake microservices). The following CIS custom resources are used on this demo: **VirtualServer**, **TLSProfile** and **Policy**. As these custom resouces are created/updated on cluster, the CIS controller (watching for them) automatically create/update the configuration on the BIG-IP system to expose the application.

## Overview

This is a high level view of what will be performed on this demo (excluding the environment setup):

1. Deploy a demo application on the cluster (**Deployment** and **Service** resources);

2. Create a **VirtualServer** custom resource to expose the application over HTTP (port 80) through BIG-IP and test the application;

3. Scale up one of the application's microservice and see how the BIG-IP configuration is automatically updated by the CIS controller;

4. Create a **TLSProfile** custom resource which will contain the details of how the TLS traffic for the application should be processed;

5. Update the **VirtualServer** resource to reference the **TLSProfile** created in the previous step, test the application over HTTPS  and see the modications done on the BIG-IP by the CIS;

6. Create an *iRule* using *iControl* and a *WAF policy* using the *declarative WAF* interface;

7. Create a **Policy** custom resource which will reference both the *iRule* and the *WAF policy* created on the previous step;

8. Update the **VirtualServer** resource to reference the **Policy** created on the previous step, test the application and see the modifications done on the BIG-IP by the CIS;

## Building an F5 CIS (Container Ingress Services) lab environment (step-by-step)

1. Define some environment variables:

    ```
    export PROJECTNAME="f5-cis-lab-001"
    export REGION="us-central1"
    export ZONE="us-central1-a"
    ```

2. Create a GCP project: 

    ```
    gcloud projects create $PROJECTNAME --name="My F5 CIS Lab" 
    ```

3. Configure the newly created project as the default project:

    ```
    gcloud config set project $PROJECTNAME
    ```

4. Get the billing account ID which will be used by this project: 

    ```
    gcloud alpha billing accounts list
    ```

5. Link the newly created project with your billing account :

    ```
    gcloud alpha billing projects link $PROJECTNAME --billing-account XXXXXX-XXXXXX-XXXXXX
    ```

6. Get your public IP and save it in an environment variable (this public IP will be used to restrict the access to your lab environment): 

    ```
    export MYIP=$(curl api.ipify.org)
    ```

7. Enable some GCP APIs that will be used later on:

    ```
    gcloud services enable compute.googleapis.com
    gcloud services enable deploymentmanager.googleapis.com
    gcloud services enable container.googleapis.com
    ```

8. Delete the default network and firewall rules (just for cleanup, this step is optional):

    ```
    gcloud compute firewall-rules delete default-allow-icmp --quiet
    gcloud compute firewall-rules delete default-allow-internal --quiet
    gcloud compute firewall-rules delete default-allow-rdp --quiet
    gcloud compute firewall-rules delete default-allow-ssh --quiet
    gcloud compute networks delete default --quiet
    ```

9. Create 3 VPC networks (external,internal,management):

    ```
    gcloud compute networks create net-external --subnet-mode=custom
    gcloud compute networks create net-internal --subnet-mode=custom
    gcloud compute networks create net-management --subnet-mode=custom
    ```

10. Create 3 subnets (one for each VPC created in the previous step):

    ```
    gcloud compute networks subnets create subnet-external --network=net-external --range=10.10.0.0/16 --region=$REGION
    gcloud compute networks subnets create subnet-internal --network=net-internal --range=172.16.0.0/16 --region=$REGION
    gcloud compute networks subnets create subnet-management --network=net-management --range=192.168.1.0/24 --region=$REGION
    ```

11. Generare a SSH key pair (which will be used to access your environment):
    ```
    ssh-keygen -f mykey
    echo "admin:$(cat mykey.pub)" > mykey_gcp.pub
    ```


12. Upload the public key to the project metadata (the corresponding private key will be used to login in the VMs):

    ```
    gcloud compute project-info add-metadata --metadata-from-file=ssh-keys=./mykey_gcp.pub
    ```

13. Create a GKE cluster: 

    ```
    gcloud container clusters create f5-cis-gke-cluster --project=$PROJECTNAME --zone=$ZONE --network=net-internal --subnetwork=subnet-internal --cluster-ipv4-cidr=10.200.0.0/14
    ```

14. Configure the *kubectl* command to use the newly created GKE cluster:

    ```
    gcloud container clusters get-credentials f5-cis-gke-cluster
    ```

15. Check the CGE cluster's nodes:

    ```
    kubectl get nodes
    ```

16. Clone F5 GDM templates repository:

    ```
    git clone https://github.com/F5Networks/f5-google-gdm-templates.git
    ```

17. Copy the needed templates files to your current directory (we will the deploy a BIG-IP Standalone with 3-NICs using PAYG): 

    ```
    cp f5-google-gdm-templates/supported/standalone/3nic/existing-stack/payg/* .
    ```

18. Configure the GDM template: 

    ```
    sed -i "s/region:/region: $REGION/" f5-existing-stack-payg-3nic-bigip.yaml
    sed -i "s/availabilityZone1:/availabilityZone1: $ZONE/" f5-existing-stack-payg-3nic-bigip.yaml
    sed -i "s/mgmtNetwork:/mgmtNetwork: net-management/" f5-existing-stack-payg-3nic-bigip.yaml
    sed -i "s/mgmtSubnet:/mgmtSubnet: subnet-management/" f5-existing-stack-payg-3nic-bigip.yaml
    sed -i "s/restrictedSrcAddress:/restrictedSrcAddress: $MYIP/" f5-existing-stack-payg-3nic-bigip.yaml
    sed -i "s/restrictedSrcAddressApp:/restrictedSrcAddressApp: $MYIP/" f5-existing-stack-payg-3nic-bigip.yaml
    sed -i "s/network1:/network1: net-external/" f5-existing-stack-payg-3nic-bigip.yaml
    sed -i "s/subnet1:/subnet1: subnet-external/" f5-existing-stack-payg-3nic-bigip.yaml
    sed -i "s/network2:/network2: net-internal/" f5-existing-stack-payg-3nic-bigip.yaml
    sed -i "s/subnet2:/subnet2: subnet-internal/" f5-existing-stack-payg-3nic-bigip.yaml
    sed -i "s/instanceType: n1-standard-4/instanceType: e2-standard-8/" f5-existing-stack-payg-3nic-bigip.yaml
    sed -i "s/applicationPort: 443/applicationPort: 80 443/" f5-existing-stack-payg-3nic-bigip.yaml
    sed -i "s/bigIpModules: ltm:nominal/bigIpModules: ltm:nominal,asm:nominal/" f5-existing-stack-payg-3nic-bigip.yaml
    ```

19. Deploy the BIG-IP:

    ```
    gcloud deployment-manager deployments create f5-cis-lab --config f5-existing-stack-payg-3nic-bigip.yaml
    ```
    **Note:** The BIG-IP can take up to 8 min to became available. 

20. Get the BIG-IP public management IP :

    ```
    export BIGIP=`gcloud compute instances describe bigip1-f5-cis-lab --format='get(networkInterfaces[1].accessConfigs[0].natIP)' --zone $ZONE`
    ```

21. Get the BIG-IP internal self-IP (which will be used when configuring CIS controller):

    ```
    export INTERNAL_SELFIP=`gcloud compute instances describe bigip1-f5-cis-lab --format='get(networkInterfaces[2].networkIP)' --zone $ZONE`
    ```

22. Log in the BIG-IP using the private key created previously: 

    ```
    ssh -i mykey admin@$BIGIP
    ```

23. Change the admin's password :

    ```
    modify /auth user admin password "F5training@123"
    ```

24. Create a partition (which will be managed by CIS):

    ```
    create /auth partition GKE
    ```

25. Add a network route to the Pods network (10.200.0.0/14)

    ```
    create /net route k8s_pods network 10.200.0.0/14 gw 172.16.0.1
    ```

26. Change the port lockdown setting of the Internal Self-IP to "Allow Default":

    ```
    modify /net self self_internal allow-service default
    ```

27. Save the config and quit :

    ```
    save sys config
    quit
    ```

28. Create a firewall rule which will allow the CIS controller to communicate with the BIG-IP:

    ```
    gcloud compute firewall-rules create fw-rule-allow-cis --direction=INGRESS --priority=1000 --network=net-internal --action=ALLOW --rules=tcp:443 --source-ranges=10.200.0.0/14 --target-tags=appfw-f5-cis-lab
    ```

29. Adjust the CIS deploy file (configuring the Internal Self-IP):

    ```
    sed "s/X.X.X.X/$INTERNAL_SELFIP/" k8s_manifests/f5cis/cis_deploy.original.yaml > k8s_manifests/f5cis/cis_deploy.yaml
    ```

30. Check the CIS deploy file:

    ```
    less k8s_manifests/f5cis/cis_deploy.yaml
    ```

31. Deploy the CIS controller:

    ```
    kubectl create secret generic bigip-login -n kube-system --from-literal=username="admin" --from-literal=password="F5training@123"
    kubectl create serviceaccount bigip-ctlr -n kube-system
    kubectl apply -f  k8s_manifests/f5cis/customresourcedefinitions.yaml
    kubectl apply -f  k8s_manifests/f5cis/k8s_rbac.yaml
    kubectl apply -f  k8s_manifests/f5cis/cis_deploy.yaml
    ```

32. Deploy the demo SHOP application with all of your microservices (*payment,order,shipping,product,user*) on GKE: 

    ```
    kubectl apply -f k8s_manifests/demo/shop.yaml
    ```

33. Check the PODs of all microservices:

    ```
    kubectl get pods
    NAME                        READY   STATUS    RESTARTS   AGE
    order-788575f6cb-k5zdx      1/1     Running   0          12s
    order-788575f6cb-t8t76      1/1     Running   0          12s
    order-788575f6cb-vwh4j      1/1     Running   0          12s
    payment-77bc8bd9cb-2v9hs    1/1     Running   0          14s
    payment-77bc8bd9cb-4v8rt    1/1     Running   0          14s
    payment-77bc8bd9cb-wnpm9    1/1     Running   0          14s
    product-6b8c9986f4-ccz8b    1/1     Running   0          11s
    product-6b8c9986f4-kzgp6    1/1     Running   0          11s
    product-6b8c9986f4-s777p    1/1     Running   0          11s
    shipping-676bd48c48-862tl   1/1     Running   0          12s
    shipping-676bd48c48-qccjn   1/1     Running   0          11s
    shipping-676bd48c48-zp9lh   1/1     Running   0          11s
    user-76dccd7788-cvjdp       1/1     Running   0          10s
    user-76dccd7788-jwrjr       1/1     Running   0          10s
    user-76dccd7788-nqthw       1/1     Running   0          10s
    ```

34. Create a static public IP (which will be used to access the SHOP demo application):

    ```
    gcloud compute addresses create static-shop-ip
    ```

35. Get the public IP address which were allocated:

     ```
     export SHOP_IP=`gcloud compute addresses list --filter="name=('static-shop-ip')" --format="value(address)"`
     ```

36. Create a target instance (which points to the BIG-IP VM instance):

    ```
    gcloud compute target-instances create bigip1-target-instance --instance=bigip1-f5-cis-lab
    ```

37. Create two forwarding rules (one for HTTP and other for HTTPs):

    ```
    gcloud compute forwarding-rules create forwarding-rule-shop-http --ip-protocol=TCP --load-balancing-scheme=EXTERNAL --network-tier=PREMIUM --ports=80 --target-instance=bigip1-target-instance --address $SHOP_IP
    gcloud compute forwarding-rules create forwarding-rule-shop-https --ip-protocol=TCP --load-balancing-scheme=EXTERNAL --network-tier=PREMIUM --ports=443 --target-instance=bigip1-target-instance --address $SHOP_IP
    ```

38. Adjust the **Virtual Server** CIS resource file:

    ```
    sed "s/X.X.X.X/$SHOP_IP/" k8s_manifests/demo/shop-virtualserver.v1.original.yaml > k8s_manifests/demo/shop-virtualserver.yaml
    ```

39. Check the **Virtual Server** CIS resource file:

      ```
      less k8s_manifests/demo/shop-virtualserver.yaml
      ```

40. Deploy the **VirtualServer** CIS resource:

    ```
    kubectl apply -f k8s_manifests/demo/shop-virtualserver.yaml
    ```

41. Check all the demo microservices: 

    ```
    curl -s http://$SHOP_IP/payment -H "Host: shop.f5lab.com"
    Server address: 10.200.0.4:80
    Server name: payment-77bc8bd9cb-wnpm9
    Date: 18/Feb/2022:23:49:47 +0000
    URI: /payment
    Request ID: b6d9d6b1c5e2403369cf89743ce6fa09

    curl -s http://$SHOP_IP/order -H "Host: shop.f5lab.com"
    Server address: 10.200.0.5:80
    Server name: order-788575f6cb-t8t76
    Date: 18/Feb/2022:23:49:48 +0000
    URI: /order
    Request ID: c0e97a4e5578a96bd6555244c48244c3

    curl -s http://$SHOP_IP/shipping -H "Host: shop.f5lab.com"
    Server address: 10.200.0.6:80
    Server name: shipping-676bd48c48-qccjn
    Date: 18/Feb/2022:23:49:48 +0000
    URI: /shipping
    Request ID: bbc6bd57764d01632e42b26a3b01dcb5

    curl -s http://$SHOP_IP/product -H "Host: shop.f5lab.com"
    Server address: 10.200.0.7:80
    Server name: product-6b8c9986f4-ccz8b
    Date: 18/Feb/2022:23:49:49 +0000
    URI: /product
    Request ID: 19be107dd0832b3aa41da1fcdd0a1a0e

    curl -s http://$SHOP_IP/user -H "Host: shop.f5lab.com"
    Server address: 10.200.0.8:80
    Server name: user-76dccd7788-nqthw
    Date: 18/Feb/2022:23:52:09 +0000
    URI: /user
    Request ID: 16ed9f21c19e4cdacaa2f7b26384a85e
    ```
      
      **Note:** In the output of the above commands, the **Server name** contains the POD's name. You can run the above commands multiple times to see how the PODs are being load balanced.  

42. Try to access a non existent URI PATH (you shoud receive an RST):

    ```
    curl http://$SHOP_IP/nonexistent -H "Host: shop.f5lab.com"
    curl: (56) Recv failure: Conexão fechada pela outra ponta
    ```

43. Check the BIG-IP configuration on the Configuration Utiliy (get the management IP in environment variable "BIGIP"):

    ```
    echo "https://$BIGIP/"
    ```

    ![Virtual Server List](https://github.com/pedrorouremalta/f5-cis-lab-on-gcp/blob/master/images/cis00.png)
    ![Virtual Server](https://github.com/pedrorouremalta/f5-cis-lab-on-gcp/blob/master/images/cis01.png)
    ![Virtual Server Resources](https://github.com/pedrorouremalta/f5-cis-lab-on-gcp/blob/master/images/cis02.png)
    ![LTM Policy](https://github.com/pedrorouremalta/f5-cis-lab-on-gcp/blob/master/images/cis03.png)
    ![Pools](https://github.com/pedrorouremalta/f5-cis-lab-on-gcp/blob/master/images/cis04.png)
    ![Pool product - Before](https://github.com/pedrorouremalta/f5-cis-lab-on-gcp/blob/master/images/cis05.png)

44. Scale up the "product" microservice to 10 replicas (you should see the changes happen automatically on BIG-IP):

    ```
    kubectl scale deployment/product --replicas=10
    kubectl get pod -l app=product
    NAME                       READY   STATUS    RESTARTS   AGE
    product-6b8c9986f4-6vrld   1/1     Running   0          3s
    product-6b8c9986f4-bbwlq   1/1     Running   0          3s
    product-6b8c9986f4-ccz8b   1/1     Running   0          14m
    product-6b8c9986f4-ckvjg   1/1     Running   0          3s
    product-6b8c9986f4-dbppv   1/1     Running   0          3s
    product-6b8c9986f4-fmtbb   1/1     Running   0          3s
    product-6b8c9986f4-kzgp6   1/1     Running   0          14m
    product-6b8c9986f4-nqqr5   1/1     Running   0          3s
    product-6b8c9986f4-s777p   1/1     Running   0          14m
    product-6b8c9986f4-xljrn   1/1     Running   0          3s
    ```

    ![Pool product - After](https://github.com/pedrorouremalta/f5-cis-lab-on-gcp/blob/master/images/cis06.png)

45. Create the **TLSProfile** CIS resource (which contains the details about how the TLS traffic will be processed):

    ```
    kubectl apply -f k8s_manifests/demo/shop-tlsprofile.yaml
    ```

46. Update the **Virtual Server** CIS resource file:

    ```
    sed "s/X.X.X.X/$SHOP_IP/" k8s_manifests/demo/shop-virtualserver.v2.original.yaml > k8s_manifests/demo/shop-virtualserver.yaml
    ```

47. Check the **Virtual Server** CIS resource file:

    ```
    less k8s_manifests/demo/shop-virtualserver.yaml
    ```

48. Update the **VirtualServer** CIS resource:

    ```
    kubectl apply -f k8s_manifests/demo/shop-virtualserver.yaml
    ```

49. Check all the demo microservices (using HTTPS): 

    ```
    curl -k -s https://$SHOP_IP/payment -H "Host: shop.f5lab.com"
    Server address: 10.200.1.6:80
    Server name: payment-77bc8bd9cb-2v9hs
    Date: 19/Feb/2022:00:04:08 +0000
    URI: /payment
    Request ID: 3c716484b224fed55ea2f50e07c9c1e1

    curl -k -s https://$SHOP_IP/order -H "Host: shop.f5lab.com"
    Server address: 10.200.0.5:80
    Server name: order-788575f6cb-t8t76
    Date: 19/Feb/2022:00:04:09 +0000
    URI: /order
    Request ID: 36a1c0ccbff0c9075ed9c481aa02ba12
    
    curl -k -s https://$SHOP_IP/shipping -H "Host: shop.f5lab.com"
    Server address: 10.200.0.6:80
    Server name: shipping-676bd48c48-qccjn
    Date: 19/Feb/2022:00:04:09 +0000
    URI: /shipping
    Request ID: 1b41622aaae84552821002194fcbe6d8

    curl -k -s https://$SHOP_IP/product -H "Host: shop.f5lab.com"
    Server address: 10.200.1.9:80
    Server name: product-6b8c9986f4-kzgp6
    Date: 19/Feb/2022:00:04:10 +0000
    URI: /product
    Request ID: 0ddf865f3fde70ca727142c686425d7c

    curl -k -s https://$SHOP_IP/user -H "Host: shop.f5lab.com"
    Server address: 10.200.1.10:80
    Server name: user-76dccd7788-cvjdp
    Date: 19/Feb/2022:00:04:14 +0000
    URI: /user
    Request ID: a83c5d7ce65276b45131a2f38c0285b8
    ```

50. Check the BIG-IP configuration on the Configuration Utiliy (get the management IP in the environment variable "BIGIP"):

    ```
    echo "https://$BIGIP/"
    ```

    ![Virtual Server List](https://github.com/pedrorouremalta/f5-cis-lab-on-gcp/blob/master/images/cis07.png)

    ![Virtual Server HTTP](https://github.com/pedrorouremalta/f5-cis-lab-on-gcp/blob/master/images/cis08.png)

    ![Virtual Server HTTP Resources](https://github.com/pedrorouremalta/f5-cis-lab-on-gcp/blob/master/images/cis09.png)

    ![Virtual Server HTTPS](https://github.com/pedrorouremalta/f5-cis-lab-on-gcp/blob/master/images/cis10.png)

    ![Virtual Server HTTPS Resources](https://github.com/pedrorouremalta/f5-cis-lab-on-gcp/blob/master/images/cis11.png)

    ![LTM Policy](https://github.com/pedrorouremalta/f5-cis-lab-on-gcp/blob/master/images/cis12.png)


51. Create an iRule (which will show a "Content not found" message when the user try to access a non existent URI PATH): 

    ```
    curl -sku "admin:F5training@123" -H "Content-Type: application/json" -X POST https://$BIGIP/mgmt/tm/ltm/rule -d '{"name":"irule_not_found","apiAnonymous":"when HTTP_REQUEST {\n     set pool [LB::server pool]\n     if { $pool == \"\"} {\n       HTTP::respond 404 content \"{\\\"status\\\": 404, \\\"message\\\": \\\"Content not found\\\"}\\r\\n\"\n     } \n }"}' | jq .
      ```

52. Create an ASM policy using the **Declarative WAF** interface (policy in *blocking mode* with *signature staging disabled*):

    ```
    curl -sku "admin:F5training@123" -X POST https://$BIGIP/mgmt/tm/asm/file-transfer/uploads/asmpolicy_shop_f5lab_com.json -H 'Content-Range: 0-554/555' -H 'Content-Type: application/json' -d @asmpolicy_shop_f5lab_com.json | jq .
    curl -sku "admin:F5training@123" -X POST https://$BIGIP/mgmt/tm/asm/tasks/import-policy/ -H 'Content-Type: application/json' -d '{"filename":"asmpolicy_shop_f5lab_com.json", "policy": { "fullPath":"/Common/asmpolicy_shop_f5lab_com" } }' | jq .
    ```

53. Create the **Policy** CIS resource file (which contain details about what irules, profiles and policies will be applied to the virtual server):

    ```
    kubectl apply -f k8s_manifests/demo/shop-policy.yaml
    ```

54. Update the **Virtual Server** CIS resource file:

    ```
    sed "s/X.X.X.X/$SHOP_IP/" k8s_manifests/demo/shop-virtualserver.v3.original.yaml > k8s_manifests/demo/shop-virtualserver.yaml
    ```

55. Check the **Virtual Server** CIS resource file:

    ```
    less k8s_manifests/demo/shop-virtualserver.yaml
    ```

56. Update the **VirtualServer** CIS resource:

    ```
    kubectl apply -f k8s_manifests/demo/shop-virtualserver.yaml
    ```

57. Try to access a non existent URI PATH (you should see a JSON error message ):

    ```
    curl -k https://$SHOP_IP/nonexistent -H "Host: shop.f5lab.com"
    {"status": 404, "message": "Content not found"}
    ```

58. Simulate an attack to the application (you should see the ASM blocking response page):

    ```
    curl -k -s "https://$SHOP_IP/payment?name=<script>alert('xss');</script>" -H "Host: shop.f5lab.com"
    <html><head><title>Request Rejected</title></head><body>The requested URL was rejected. Please consult with your administrator.<br><br>Your support ID is: 1311895922317197328<br><br><a href='javascript:history.back();'>[Go Back]</a></body></html>
    ```
59. Check the BIG-IP configuration on the Configuration Utiliy (get the management IP in environment variable "BIGIP"):

    ```
    echo "https://$BIGIP/"
    ```
 
    ![Virtual Server HTTPS Resources](https://github.com/pedrorouremalta/f5-cis-lab-on-gcp/blob/master/images/cis13.png)

    ![Virtual Server HTTPS Security](https://github.com/pedrorouremalta/f5-cis-lab-on-gcp/blob/master/images/cis14.png)

    ![Security Event Log](https://github.com/pedrorouremalta/f5-cis-lab-on-gcp/blob/master/images/cis15.png)

## Cleaning up the lab environment (step-by-step)

1. Delete the two forwarding rules:

    ```
    gcloud compute forwarding-rules delete forwarding-rule-shop-http --quiet
    gcloud compute forwarding-rules delete forwarding-rule-shop-https --quiet
    ```

2. Delete the target instance:

    ```
    gcloud compute target-instances delete bigip1-target-instance --quiet
    ```

3. Delete the static public IP:

    ```
    gcloud compute addresses delete static-shop-ip --quiet
    ```

4. Delete the BIG-IP deployment:

    ``` 
    gcloud deployment-manager deployments delete f5-cis-lab --delete-policy=DELETE --quiet
    ```

5. Delete the GKE cluster:

    ```
    gcloud container clusters delete f5-cis-gke-cluster --quiet
    ```

6. Delete the remaining firewall rule:

    ```
    gcloud compute firewall-rules delete fw-rule-allow-cis --quiet
    ```

7. Clean up the directory:

    ```
    rm -rf ./f5-google-gdm-templates/
    rm f5-existing-stack-payg-3nic-bigip.* 
    rm k8s_manifests/f5cis/cis_deploy.yaml
    rm k8s_manifests/demo/shop-virtualserver.yaml
    ```
