# Moodle Single-Zone Region Deployment on Kubernetes Cluster in IBM Cloud

Moodle is a Learning Platform or course management system (CMS) - a free Open Source software package designed to help educators create effective online courses based on sound pedagogical principles. Moodle has a large and diverse user community with over 100,000 sites registered worldwide speaking over 140 languages in every country. It is designed to provide educators, administrators and learners with a single robust, secure and integrated system to create personalized learning environments.
This repos provides a step by step guide to show how to build a complete solution for Moodle on a single-zone Kubernetes Cluster on VPC in IBM Cloud. The solution includes data storage, backups, DNS setup, security, logging and monitoring.

## Solution Details
The solution is composed of two main containers: Moodle container that represents Moodle application with a persistence storage attached to store Moodle data and the second container represents MariaDB database with a persistence volume attached to store the database data.

Those two containers are deployed along with all the other required resources in Kubernetes such as persistence volumes, secrets, config maps, etc. by using [Moodle Bitnami Helm Chart](https://github.com/bitnami/charts/tree/master/bitnami/moodle). 

The Kubernetes Cluster is provisioned with its worker nodes in Virtual Private Cloud (VPC) that has one subnet configured in one zone with public gateway. 

Database Backups is configured manually after Moodle deployment by using Kubernetes Cronjob and Object Storage as a persistence volume (Object Storage Storageclasses might need to be configured on the Kubernetes Cluster if it is not already configured).

Cloud Internet Services are used to configure the DNS and then to add some security features such as Distributed Denial of Service (DDoS) protection, and Web Application Firewall (WAF).

The solution also includes logging and monitoring services by using IBM Log Analysis with LogDNA and IBM Cloud Monitoring with Sysdig respectively.

## Components

| Component | Configuration | Notes |
| - | - | - |
| **Virtual Private Cloud (VPC)** | Configured with one subnet with public gateway. | VPC is required to host the Kubernetes Cluster and its worker nodes.|
| **Kubernetes Cluster** | Provisioned with 3 worker nodes each 2 vCPU and 4 GB. |  <ul><li>As per Moodle documentation, the minimum compute power is 1GB memory for development environment and 8 GB for production ones. Reference: https://docs.moodle.org/310/en/Installing_Moodle </li> <li>The minimum number of worker nodes per cluster is 2. </li><li>If is expected heavily usage, you can consider increasing the computing power accordingly or you can even consider provisioning a multi-zone cluster.</li></ul>|
| **Moodle – Bitnami Helm Chart** | Configured to easily deploy both Moodle and Maria DB containers. | Moodle Helm Chart makes the deployment of Moodle much easier as it includes both Moodle and MariaDB containers along with all the required resources such as config maps, secrets and persistence volume. For the persistence volumes, it is also responsible of the provisioning of the physical storage. |
| **Block Storage for VPC** | Two instances are provisioned with 10GB for both Moodle data and database storage using Moodle Helm Chart. | The Helm Chart is responsible of provisioning 2 persistence volume of 10 IOPS block storage with the defined storage. The storage could be configured based on the requirements for example the number of users and data. |
| **Object Storage** | Configured with 80GB to store backups while provisioning the persistence volume for backups. | Object Storage is used for storing database backups. Full database backup is taken by cronjob, you can configure this job to control the frequency of the backups and the retention period for the backups and also you can identify the size of the storage that is required for backups. |
| **Cloud Internet Services** | Standard Plan | It is used to configure DNS. It also contains Web Application Firewall and DDoS attacks protection. |
| **Certificate Manager** | Free Plan | It is used to manage the SSL certificates. |
| **IBM Log Analysis with LogDNA** | 7 days log search – 100GB | It is used for logging. You use other packages with extra storage based on your requirements. |
| **IBM Cloud Monitoring with Sysdig** | Graduated Tier – 3 nodes and 2 containers | It is used for monitoring your cluster. |
---

## Architecture

![Solution Architecture](./Moodle-SZR.png)

## Implementation

This section shows step by step how to implement the full solution starting by creating the Virtual Private Cloud, provisioning the Kubernetes Cluster, then deploying Moodle and MariaDB using Moodle Bitnami Helm Chart, configuring the Database Backups jobs, configuring the DNS and securing your application and finally enabling the logging and monitoring services for the Kubernetes Cluster.

### Prerequisite
-	IBM Cloud Account.
-	Command Line Interface (Preferably Linux Based).
-	Domain name.

### Steps
- [Moodle Single-Zone Region Deployment on Kubernetes Cluster in IBM Cloud](#moodle-single-zone-region-deployment-on-kubernetes-cluster-in-ibm-cloud)
  - [Solution Details](#solution-details)
  - [Components](#components)
  - [| **IBM Cloud Monitoring with Sysdig** | Graduated Tier – 3 nodes and 2 containers | It is used for monitoring your cluster. |](#-ibm-cloud-monitoring-with-sysdig--graduated-tier--3-nodes-and-2-containers--it-is-used-for-monitoring-your-cluster-)
  - [Architecture](#architecture)
  - [Implementation](#implementation)
    - [Prerequisite](#prerequisite)
    - [Steps](#steps)
    - [1. Setup Command Line Tools](#1-setup-command-line-tools)
    - [2. Create Your Virtual Private Cloud (VPC)](#2-create-your-virtual-private-cloud-vpc)
    - [3. Provision Kubernetes Cluster](#3-provision-kubernetes-cluster)
    - [4. Deploy Bitnami Moodle Helm Chart](#4-deploy-bitnami-moodle-helm-chart)
    - [5. Configure Object Storage StorageClasses in the Kubernetes Cluster](#5-configure-object-storage-storageclasses-in-the-kubernetes-cluster)
    - [6. Configure the database backups](#6-configure-the-database-backups)
    - [7. Configure the Cloud Internet Services](#7-configure-the-cloud-internet-services)
    - [8. Enable Logging and Monitoring Services](#8-enable-logging-and-monitoring-services)
  - [References](#references)

### 1. Setup Command Line Tools
1. Install IBM Cloud CLI `ibmcloud` along with IBM Cloud Kubernetes Service plug-in `ibmcloud ks`, IBM Cloud Container Registry plug-in `ibmcloud cr` and IBM Cloud Kubernetes Service observability plug-in `ibmcloud ob` by following the steps [here](https://cloud.ibm.com/docs/containers?topic=containers-cs_cli_install#cs_cli_install_steps).
2. Download the Kubernetes CLI major.minor version that matches the Kubernetes cluster major.minor version that you plan to use. The current IBM Cloud Kubernetes Service default Kubernetes version is 1.18.14.

    **OS X:** https://storage.googleapis.com/kubernetes-release/release/v1.18.14/bin/darwin/amd64/kubectl
    
    **Linux:** https://storage.googleapis.com/kubernetes-release/release/v1.18.14/bin/linux/amd64/kubectl

    **Windows:** Install the Kubernetes CLI in the same directory as the IBM Cloud CLI. This setup saves you some file path changes when you run commands later. https://storage.googleapis.com/kubernetes-release/release/v1.18.14/bin/windows/amd64/kubectl.exe

3. After the installation of Kubernetes CLI `kubectl` on your local machine, make sure to add it to your `PATH` system variable.
4. Verify the installation of kubectl by running the following command:
`kubectl version`
5. Install the latest release of the version 3 Helm CLI on your local machine.
6. Verify the installation of helm by running the following command:
`helm version`

### 2. Create Your Virtual Private Cloud (VPC)
1. Login to IBM Cloud with your credentials from the following link: https://cloud.ibm.com/.
2. From the left side menu, select **VPC Infrastructure** then select **VPCs** under **Network**.
   
   **Note:** You can also reach the VPC page directly from this link: https://cloud.ibm.com/vpc-ext/network/vpcs
3. Select the region where you want to provision your cluster from the region list then click **Create +**.
4. Fill the **New Virtual Private Cloud** form by entering the **Name** for example moodle-demo-vpc and then keep the rest of the values with default value.
5. Scroll down to **New subnet for VPC** form then enter a **Name** for the new subnet for example subnet-01, select the **Location** where you are going to provision the Kubernetes Cluster and enable the Public gateway at the bottom of the form.
6. Click **Create virtual private cloud** from the right side.

### 3. Provision Kubernetes Cluster
1. From the left side menu, select **Kubernetes** and then click on **Clusters**.
2. Click **Create cluster +**.
3. Fill the **Kubernetes cluster** form with the following:
    - Select **Standard** in **Pricing Plan**.
    - Select **Kubernetes** default version, currently it is **1.18.14**.
    - Under **Infrastructure**, select **VPC**.
    - Select the **virtual private cloud** that was created in the previous step.
    - Keep the **Default** **Resource group**.
    - In the **Worker zones**, check only the zone that has subnet and uncheck the other two zones.
    - Under Worker pool, click **Change flavor** and select the node with 2 vCPU and 4GB Memory **cx2.2x4** then click **Save worker pool flavor**.
    - Keep the number of **Worker nodes per zone** as **3**.
    - Keep the **Worker pool name** with the **default** value and select **Both private & public endpoints** under **Master service endpoint**.
    - Enter a name for your cluster and then add tags for your Cluster if you want.
4. After filling the form, Click **Create** button in the right side. 
5. Wait until the cluster is provisioned successfully, that will take some time up to 20 minutes.
6. If your cluster runs Kubernetes version 1.18 or earlier, will need to apply the following steps:
   - From the left side menu, select **VPC Infrastructure** then select **Security groups** under **Network**.
    
        **Note:** You can also reach the VPC page directly from this link: https://cloud.ibm.com/vpc-ext/network/securityGroups
   - Select the region where your cluster is provisioned. 
   - Select the security group that belongs to your VPC, you can identify it from the Virtual Private Cloud column.
   - Select Rules from the top menu.
   - Under **Inbound rules**, click **Create +**.
   - Select **TCP** in **Protocol**, **Port range** in **Port**, *30000* in **Port min**, *32767* in **Port max** and **Any** in **Source type** then click **Save**.
   - Repeat the same step again to add the same rule for the **UDP** protocol.
7. To Confirm that the setup of the Kubernetes Cluster is done successfully, make sure that the cluster status is **Normal** with the green mark and that the ingress status is **Healthy** with the green mark.

### 4. Deploy Bitnami Moodle Helm Chart
1. From the command line, login to IBM Cloud by running the following command:
`ibmcloud login –sso`. Follow the steps to login in the browser then enter the passcode in the command line to login.
2. Make sure to select the region where you provisioned your cluster by running the following command: `ibmcloud target -r <region>`
 
    **Note:** To get the name of your region you can list all the regions available in IBM Cloud by using the following command: `ibmcloud regions`
3. Make sure to select the resource group where you provisioned your cluster by running the following command: `ibmcloud target -g <region>`
 
    **Note:** the default resource group is **Default** unless you change it during the provisioning.
4. List your clusters by running the following command: `ibmcloud ks clusters`
5. Set the cluster that you created as the context for this session by running the following command: `ibmcloud ks cluster config -c <cluster_name_or_ID>`
6. Verify that kubectl commands run properly and that the Kubernetes context is set to your cluster by running the following command: `kubectl config current-context`
7. Create a new namespace for Moodle deployment by running the following command: `kubectl create namespace moodle`
8. Add bitnami-ibm repos to your helm repos by running the following command: `helm repo add bitnami-ibm https://charts.bitnami.com/ibm`
9. Download the [values.yaml](./values.yaml) file in your local machine, in the solution here will keep all the default values except the size of the persistence volumes will make it 10Gi, so will update in line 216 and 341.
    
    **Note:** Feel free to update other values in the file, you can check the full documentation from this link: https://github.com/bitnami/charts/tree/master/bitnami/moodle
### 5. Configure Object Storage StorageClasses in the Kubernetes Cluster
### 6. Configure the database backups
### 7. Configure the Cloud Internet Services
### 8. Enable Logging and Monitoring Services

## References
-	https://moodle.org/
-	https://docs.moodle.org/310/en/Installing_Moodle
-	https://bitnami.com/stack/moodle/helm
-	https://github.com/bitnami/charts/tree/master/bitnami/moodle
-	https://cloud.ibm.com/catalog/content/moodle
-	https://cloud.ibm.com/docs/containers?topic=containers-object_storage
-	https://cloud.ibm.com/docs/cis?topic=solution-tutorials-multi-region-k8s-cis#multi-region-k8s-cis-4
-	https://cloud.ibm.com/docs/certificate-manager?topic=certificate-manager-ordering-certificates
-	https://cloud.ibm.com/docs/containers?topic=containers-health
-	https://cloud.ibm.com/docs/containers?topic=containers-health-monitor
