## Task 3: Create an Azure Kubernetes Service cluster

In this task, you will create an Azure Kubernetes service and review the deployed resources.

1. In the Azure portal, in the Search resources, services, and docs text box at the top of the Azure portal page, type Kubernetes services and press the Enter key.

2. On the Kubernetes services blade, click + Create and, in the drop-down menu, click + Create a Kubernetes cluster

3. On the Basics tab of the Create Kubernetes cluster blade, click Change preset configurations, select Dev/Test ($) and click Apply. Now specify the following settings (leave others with their default values):

 | Setting                  |           Value                                                  |
 |--------------------------|------------------------------------------------------------------|
 | Subscription             | 	the name of the Azure subscription you are using in this lab   |
 | Resource group           |	  AZ500LAB09                                                     |
 | Kubernetes cluster name  |   MyKubernetesCluster                                            |
 | Region                   |  (US) East US                                                    |
 | Availability zones       |  None                                                            |
 | Node count               |  1                                                               |




4. Click Next: Node Pools > and, on the Node Pools tab of the Create Kubernetes cluster blade, specify the following settings (leave others with their default values):

  |    Settings          |        Value        |
  -----------------------|---------------------|
  | Enable virtual nodes |	cleared checkbox   |
  | VM scale sets        |	cleared checkbox   |

5. Click Next: __Authentication__ >, on the Authentication tab of the __Create Kubernetes cluster__ blade, accept the defaults, and click __Next: Networking__ >.

6. On the __Networking tab__ of the __Create Kubernetes cluster__ blade, specify the following settings (leave others with their default values):


  |    Settings           |        Value              |
  ------------------------|---------------------------|
  | Network configuration |	Azure CNI                 |
  | DNS name prefix       |	Leave the default value   |




>AKS can be configured as a private cluster. This assigns a private IP to the API server to ensure network traffic between your API server and your node pools remains on the private network only. For more information, visit https://docs.microsoft.com/en-us/azure/aks/private-clusters page.

7. Click __Next: Integrations__ > and, on the __Integrations__ tab of the Create Kubernetes cluster blade, set __Container monitoring__ to __Disabled__.

>In production scenarios, you would want to enable monitoring. Monitoring is disabled in this case since it is not covered in the lab.

8. Click Review + Create and then click Create.

>Wait for the deployment to complete. This might take about 10 minutes.

9. Once the deployment completes, in the Azure portal, in the __Search resources, services, and docs__ text box at the top of the Azure portal page, type __Resource groups__ and press the Enter key.

10. On the Resource groups blade, in the listing of resource groups, note a new resource group named MC_AZ500LAB09_MyKubernetesCluster_eastus that holds components of the AKS Nodes. Review resources in this resource group.

11. Navigate back to the Resource groups blade and click the AZ500LAB09 entry.

  >In the list of resources, note the AKS Cluster and the corresponding virtual network.

12. In the Azure portal, open a Bash session in the Cloud Shell.

 > Ensure Bash is selected in the drop-down menu in the upper-left corner of the Cloud Shell pane.
13. In the Bash session within the Cloud Shell pane, run the following to connect to the Kubernetes cluster:

```
    sh
    az aks get-credentials --resource-group AZ500LAB09 --name MyKubernetesCluster
```

14. In the Bash session within the Cloud Shell pane, run the following **to list nodes of the Kubenetes cluster**:


```
  sh
    kubectl get nodes
```
>Verify that the Status of the cluster node is listed as Ready.

####  Task 4: Grant the AKS cluster permissions to access the ACR and manage its virtual network

In this task, you will grant the AKS cluster permission to access the ACR and manage its virtual network.

1. In the Bash session within the Cloud Shell pane, run the following to configure the AKS cluster to use the Azure Container Registry instance you created earlier in this lab.

```sh
    ACRNAME=$(az acr list --resource-group AZ500LAB09 --query '[].{Name:name}' --output tsv)

    az aks update -n MyKubernetesCluster -g AZ500LAB09 --attach-acr $ACRNAME
```
  > This command grants the 'acrpull' role assignment to the ACR.

  >It may take a few minutes for this command to complete.

2. In the Bash session within the Cloud Shell pane, run the following to grant the AKS cluster the Contributor role to its virtual network.

```
  sh
    RG_AKS=AZ500LAB09
    AKS_VNET_NAME=AZ500LAB09-vnet

    AKS_CLUSTER_NAME=MyKubernetesCluster

    AKS_VNET_ID=$(az network vnet show --name $AKS_VNET_NAME --resource-group $RG_AKS --query id -o tsv)

    AKS_MANAGED_ID=$(az aks show --name $AKS_CLUSTER_NAME --resource-group $RG_AKS --query identity.principalId -o tsv)

    az role assignment create --assignee $AKS_MANAGED_ID --role "Contributor" --scope $AKS_VNET_ID
```

#### Task 5: Deploy an external service to AKS

In this task, you will download the Manifest files, edit the YAML file, and apply your changes to the cluster.

1. In the Bash session within the Cloud Shell pane, click the __Upload/Download__ files icon, in the drop-down menu, click Upload, in the Open dialog box, naviate to the location where you downloaded the lab files, select __\Allfiles\Labs\09\nginxexternal.yaml__ click __Open__. Next, select __\Allfiles\Labs\09\nginxinternal.yaml__, and click __Open__.

2. In the Bash session within the Cloud Shell pane, run the following to identify the name of the Azure Container Registry instance:

  ```
    sh
   echo $ACRNAME
  ```
  >Record the Azure Container Registry instance name. You will need it later in this task.

3. In the Bash session within the Cloud Shell pane, run the following to open the nginxexternal.yaml file, so you can edit its content.

  ```
  sh
    code ./nginxexternal.yaml
  ```

 >This is the external yaml file.

 >In the editor pane, scroll down to line 24 and replace the <ACRUniquename> placeholder with the ACR name.

4. In the editor pane, in the upper right corner, click the ellipses icon, click Save and then click Close editor.

5. In the Bash session within the Cloud Shell pane, run the following to apply the change to the cluster:

    ```
    sh
    kubectl apply -f nginxexternal.yaml
    ```
6. In the Bash session within the Cloud Shell pane, review the output of the command you run in the previous task to verify that the deployment and the corresponding service have been created.

```deployment.apps/nginxexternal created
    service/nginxexternal created
```

#### Task 6: Verify the you can access an external AKS-hosted service

In this task, verify the container can be accessed externally using the public IP address.

1. In the Bash session within the Cloud Shell pane, run the following to retrieve information about the nginxexternal service including name, type, IP addresses, and ports.

  ```sh
    kubectl get service nginxexternal
  ```

2. In the Bash session within the Cloud Shell pane, review the output and record the value in the External-IP column. You will need it in the next step.

3. Open a new browser tab and browse to the IP address you identified in the previous step.

 >Ensure the Welcome to nginx! page displays.

#### Task 7: Deploy an internal service to AKS

In this task, you will deploy the internal facing service on the AKS.

1. In the Bash session within the Cloud Shell pane, run the following to open the nginxintenal.yaml file, so you can edit its content.

```
    sh
    code ./nginxinternal.yaml
```
  > This is the internal yaml file.

2. In the editor pane, scroll down to the line containing the reference to the container image and replace the <ACRUniquename> placeholder with the ACR name.

3. In the editor pane, in the upper right corner, click the ellipses icon, click Save and then click Close editor.

4. In the Bash session within the Cloud Shell pane, run the following to apply the change to the cluster:

  ```sh
    kubectl apply -f nginxinternal.yaml
  ```
5. In the Bash session within the Cloud Shell pane, review the output to verify your deployment and the service have been created:

    ```
    deployment.apps/nginxinternal created
    service/nginxinternal created
  ```
 6. In the Bash session within the Cloud Shell pane, run the following to retrieve information about the nginxinternal service including name, type, IP addresses, and ports.

    ```
    sh
    kubectl get service nginxinternal
  ```
7. In the Bash session within the Cloud Shell pane, review the output. The External-IP is, in this case, a private IP address. If it is in a Pending state then run the previous command again.

    >Record this IP address. You will need it in the next task.

    >To access the internal service endpoint, you will connect interactively to one of the pods running in the cluster.

    >Alternatively you could use the CLUSTER-IP address.

#### Task 8: Verify the you can access an internal AKS-hosted service

In this task, you will use one of the pods running on the AKS cluster to access the internal service.

1. In the Bash session within the Cloud Shell pane, run the following to list the pods in the default namespace on the AKS cluster:

```
sh
    kubectl get pods
```
 In the listing of the pods, copy the first entry in the NAME column.

3. This is the pod you will use in the subsequent steps.

4. In the Bash session within the Cloud Shell pane, run the following to connect interactively to the first pod (replace the (pod_name) placeholder with the name you copied in the previous step):

```
  sh
    kubectl exec -it <pod_name> -- /bin/bash
```
5. In the Bash session within the Cloud Shell pane, run the following to verify that the nginx web site is available via the private IP address of the service (replace the (internal_IP) placeholder with the IP address you recorded in the previous task):

    ```
    sh
    curl http://<internal_IP>
    ```
6. Close the Cloud Shell pane.
  >Result: You have configured and secured ACR and AKS.

#### Clean up resources

Remember to remove any newly created Azure resources that you no longer use. Removing unused resources ensures you will not incur unexpected costs.

In the Azure portal, open the Cloud Shell by clicking the first icon in the top right of the Azure Portal.

In the upper-left drop-down menu of the Cloud Shell pane, select PowerShell and, when prompted, click Confirm.

In the PowerShell session within the Cloud Shell pane, run the following to remove the resource groups you created in this lab:

powershell
    ```Remove-AzResourceGroup -Name "AZ500LAB09" -Force -AsJob```

    Close the Cloud Shell pane.

Congratulations!

Click Next to proceed to the Review Questions