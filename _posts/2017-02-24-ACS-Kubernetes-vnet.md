---
layout: post
title: Deploying an Azure ACS Kubernetes cluster within a vnet
---

## Deploying an Azure ACS Kubernetes cluster within a vnet

Deploying a kubernetes cluster within azure inside a custom/existing vnet can, as of now, not be done via the portal yet. Under the hood azure uses the [acs-engine](https://github.com/Azure/acs-engine) to generate the respective ARM templates out of a definition. Those templates are used to deploy the actual cluster within azure. Using the acs-engine, you can tweak the generation of the template for instance by including the nodes to a vnet. 

### Get ready with acs-engine

The most convenient way is to use acs-engine within docker as described [here](https://github.com/Azure/acs-engine/blob/master/docs/acsengine.md#downloading-and-building-acs-engine). Basically it is as easy as executing a powershellscript that initializes/generates the respective docker image which can then be used starting the container interactively, bringing you to the ubuntu bash where acs-engine is ready.

The image available looks like this
```bash
C:\Work\Git\acs-engine>docker ps -a
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS                    PORTS               NAMES
ca004cb947cd        acs-engine                     "/bin/bash"              41 hours ago        Exited (0) 16 hours ago                       friendly_euler
```

Starting docker and acessing the bash
```bash
C:\Work\Git\acs-engine>docker start -i ca
root@ca004cb947cd:/gopath/src/github.com/Azure/acs-engine#
```


### How to generate a custom ARM template 

The acs-engine repository contains a folder *examples* that can be used as a starting point. In this example the *kubernetes.json* will be modified.
This json contains the abstract information about the cluster, which will be used as input for acs-engine to generate the actual ARM template. A modified version of this, with the subnet specified already is shown below. The sections containing UPPER_CASE letters must be modified to fit your environment. In this case the *vnetSubnetId* property is referencing an exisiting vnet that was specified before. Hence this is done via a naming reference. 

```json
{
  "apiVersion": "vlabs",
  "properties": {
    "orchestratorProfile": {
      "orchestratorType": "Kubernetes"      
    },
    "masterProfile": {
      "count": 1,
      "dnsPrefix": "SOMEDNSPREFIX",
      "vmSize": "Standard_D2_v2",
      "vnetSubnetId": "/subscriptions/SUBSCRIPTIONID/resourceGroups/RESOURCE_GROUP/providers/Microsoft.Network/virtualNetworks/VNET_NAME/subnets/SUBNET_NAME1",
      "firstConsecutiveStaticIP": "10.2.1.5"
    },
    "agentPoolProfiles": [
      {
        "name": "agentpool1",
        "count": 1,
        "vmSize": "Standard_D2_v2",
        "availabilityProfile": "AvailabilitySet",
        "vnetSubnetId": "/subscriptions/SUBSCRIPTIONID/resourceGroups/RESOURCE_GROUP/providers/Microsoft.Network/virtualNetworks/VNET_NAME/subnets/SUBNET_NAME2"        
      }
    ],
    "linuxProfile": {
      "adminUsername": "azureuser",
      "ssh": {
        "publicKeys": [
          {
            "keyData": "SOME PUBLIC SSH KEY"
          }
        ]
      }
    },
    "servicePrincipalProfile": {
      "servicePrincipalClientID": "SERVICE_PRINCIPAL_ID",
      "servicePrincipalClientSecret": "SERVICE_PRINCIPAL_SECRET"
    }
  }
}
```

This file can now be used to generate the ARM template using acs-engine.

```bash
root@ca004cb947cd:/gopath/src/github.com/Azure/acs-engine# acs-engine examples/kubernetes-ralph.json
cert creation took 3.7308727s
wrote _output/Kubernetes-68763472/apimodel.json
wrote _output/Kubernetes-68763472/azuredeploy.json
wrote _output/Kubernetes-68763472/azuredeploy.parameters.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.australiaeast.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.australiasoutheast.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.brazilsouth.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.canadacentral.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.canadaeast.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.centralindia.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.centralus.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.eastasia.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.eastus.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.eastus2.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.japaneast.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.japanwest.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.koreacentral.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.koreasouth.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.northcentralus.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.northeurope.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.southcentralus.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.southeastasia.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.southindia.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.uksouth.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.ukwest.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.westcentralus.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.westeurope.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.westindia.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.westus.json
wrote _output/Kubernetes-68763472/kubeconfig/kubeconfig.westus2.json
wrote _output/Kubernetes-68763472/ca.key
wrote _output/Kubernetes-68763472/ca.crt
wrote _output/Kubernetes-68763472/apiserver.key
wrote _output/Kubernetes-68763472/apiserver.crt
wrote _output/Kubernetes-68763472/client.key
wrote _output/Kubernetes-68763472/client.crt
wrote _output/Kubernetes-68763472/kubectlClient.key
wrote _output/Kubernetes-68763472/kubectlClient.crt
acsengine took 5.3896201s
```
The acs-engine generated quite a bunch of files, though the most interesting ones are **apimodel.json**, **azuredeploy.json** and **azuredeploy.parameters.json**. That is enough to deploy the cluster.

### How to deploy the generate ARM template

To finally bring the cluster in azure to life, the [Azure CLI](https://docs.microsoft.com/en-us/azure/xplat-cli-install) is used. Hence, since an existing vnet should be leveraged in this example, a resource group called **kubrg1** is already existing within Azure before deploying the cluster. Also the vnet is already configured. The following command triggers the deployment of the cluster. This can take some minutes to finish.

```bash
azure group deployment create -f "_output/Kubernetes-68763472/azuredeploy.json" -e "_output/Kubernetes-68763472/azuredeploy.parameters.json" -g kubrg1 -n clusterdeployment
```

### Important: Do not forget the routetable

It is important to make the generated routetable aware of the existing vnet. If this is not happening, then the cluster is not functioning as expected.

![Routetable]({{ site.url }}/assets/images/kubernetes-vnet-routetable.png)


### How to update the cluster leveraging the previously generated files

The **apimodel.json** can be used to generate a modified version of the ARM templates, so in cases where for instance the number of agents should be increased from x to y the the **apimodel.json** is adjusted and acs-engine is executed again. The result will be updated **apimodel.json**, **azuredeploy.json** and **azuredeploy.parameters.json** files.

```bash
root@ca004cb947cd:/gopath/src/github.com/Azure/acs-engine# acs-engine _output/Kubernetes-68763472/apimodel.json
wrote _output/Kubernetes-68763472/apimodel.json
wrote _output/Kubernetes-68763472/azuredeploy.json
wrote _output/Kubernetes-68763472/azuredeploy.parameters.json
```

And the update to the cluster will again happen using the Azure CLI.

Thats it. 
Enjoy your kubernetes cluster within Azure.

