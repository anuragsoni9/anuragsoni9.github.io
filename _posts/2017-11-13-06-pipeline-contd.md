---
published: true
title: "Taking Models from Data Scientists to Users(Part 6): Joining the Pipeline - II"
image: /assets/Post4n6.png
---
# Building the full Data Pipeline - II

>In the [series](https://github.com/anuragsoni9/ProductionScale/blob/master/README.md), we previously talked about various components and how different stages fit together to form a data pipeline. This post first gives an introduction to the revolutionary *taskmaster* in our production environment - **Kubernetes** and then proceed to demonstrate step-by-step how to possibly implement a running data pipeline of Tyco Object Detection in Google Cloud. 


## Kubernetes - Why?

The processing infrastructure from GCS will be operated and managed by  [Kubernetes](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/)(also known as *K8s; origin from Greek meaning pilot*). In layman’s terms, Kubernetes is that member of your team who can deploy or undeploy a new machine-cluster when traffic fluctuates. But of course, it is much more than that. As a platform, K8s it is the container orchestration tool that provides

- [**visualizer**](https://github.com/kubernetes/dashboard): gives sorted info about running instances, their network utilization 

<div class="image">
<img src="https://github.com/kubernetes/dashboard/raw/master/docs/dashboard-ui.png" alt="sample" width ="900">
  <div  align="left"><i>Kubernetes Dashboard UI </i></div>
</div>
<br>


- [**federator**](https://kubernetes.io/docs/concepts/cluster-administration/federation/): load-balancing;cloud agnostic controls without intervention
- [**operator**](https://coreos.com/blog/introducing-the-etcd-operator.html): bootstrap, resize and backup storage mounts, again without intervention

Additionally, it provides individual IPs to pods(*pods are the smallest deployable container units by K8s*) which provides the ability for applications to discover backing services(*like* Databases etc.) and basically create a network of containers.

So in nutshell Kubernetes becomes the pilot of your container pods which make them **extensible** and **portable** both. It just impressively abstracts away the messy details of cloud providers or in-house data-center to give a seamless experience to the community.


## Architecture of pipeline - connecting the dots


![](https://d2mxuefqeaa7sj.cloudfront.net/s_AB23734E1608EFF3C8081959BA1B9B5FE201B6019603F5869FCF83459FDB4577_1512431011313_1.PNG)


In the above architecture:- 

- Kubernetes spins up clusters (Node1/Node2..) based on the demand of initial data and subsequent loads. There is a master node which is like an API server that talks to other clusters. *K8S* maintains a Persistent volume to store metadata in a **etcd** key-value store system.
- Pachyderm and other container pods are deployed on individual nodes with separate IPs. Multiple Docker images(Pipeline workers) can be put on one node by *K8S* to internally optimize for performance. 
- Pachyderm connects with object storage for data layer thereby maintaining various versions without any overheads
- Pachyderm dashboard allows us to visualize the pipeline in a browser via HTTP connection
## Steps to create pipeline
<div class="image">
<img src="https://d2mxuefqeaa7sj.cloudfront.net/s_AB23734E1608EFF3C8081959BA1B9B5FE201B6019603F5869FCF83459FDB4577_1512431774052_Production+Data+Pipeline+-+Tyco+1.svg" alt="sample">
  <div  align="center"><i>Target Pipeline</i></div>
</div>
<br>

**PREREQUISITES:**


- **Analysis Code**

Object Detection: The example model used in the post is coco-trained  MobileNets TensorFlow model and will attempt to detect objects in the production environment. 
Additionally, it has python scripts for 

- Validation - whether incoming information are standardized
- Threat Detector - matches classified results with rules defined in rule.json
- Plot -  basic Python plotting of number of case observed

We have a ready-made Github repo to get the codes (forked from [Daniel Whitman](http://www.datadan.io/)). except for TensorFlow model which we will get from official repo.


- **Docker Images**
Every processing stage will be deployed as Docker container which is defined with Dockerfile for each stage individually. This contains a layered definition of an image containing dependencies, binaries and codes.
For example:
```sh
    FROM <model>
    RUN <python installation>
    ADD <python script>
```

GH contains Dockerfile as well.
More on Docker container is discussed in [here](https://github.com/anuragsoni9/ProductionScale/blob/master/05-containers.md).


- **JSON Pipeline specification** 
A standard way to represent pipeline in Pachyderm is via a standardized JSON file with four main components tags:

  - *name* - unique name to indetify pipeline
  - *transform* - a reference to the Docker image that jobs run in
  - *parallelism specs* -  a reference to parallel configuration(constant or K8s cluster)
  - *input* - reference to input repo for data inputs

More on the Pipeline Specification can be found [here](http://pachyderm.readthedocs.io/en/latest/reference/pipeline_spec.html)


- **Storage Infrastructure**
This example employed Google Cloud Storage for illustration purpose, though, it can be just as easily be deployed on other Cloud servers like AWS/Digital Ocean etc. or in local machine thanks to cloud-agnostic *K8S* platform


- **Miscellaneous**
  - Bash Shell (We will be using [Linux Subsystem](https://msdn.microsoft.com/en-us/commandline/wsl/install-win10) in Windows)
  - Sendgrid API key (*optional*)




**STEPS** 


> Following steps are the ones I used to implement it on a windows machine with Google Cloud Storage. Steps are formulated to be as detailed as possible for full reproducibility so if it is too elementary for you - I suggest BREATHE!!



1. Activate Linux Subsystem on Win10 and install Ubuntu app from Windows Store. Open Ubuntu shell. Install Python with `sudo apt-get python`


2. Setup GCS account and download SDK in UNIX folder as provided [here](https://cloud.google.com/sdk/docs/quickstart-linux). You may need to put credit card details even for a free-tier account.


3. Go to the SDK folder and run install.sh as `./install.sh`


4. Initiate Google Cloud instance with `gcloud init` and use the default configuration and project(*can be set-up from Google Cloud portal)*
```sh
    ~/google-cloud-sdk$ gcloud init
    Welcome! This command will take you through the configuration of gcloud.
    
    Settings from your current configuration [default] are:
    compute:
      zone: us-west1-a
    container:
      cluster: pach-cluster
      use_client_certificate: 'True'
```
 

5. Spin up a  Kubernetes Cluster with:
```sh
    $ gcloud container clusters create ${CLUSTER_NAME} --scopes storage-rw --machine-type ${MACHINE_TYPE} --zone $GCP_ZONE --num-nodes 2
    Creating cluster pach-cluster...done.
    Created [https://container.googleapis.com/v1/projects/amiable-shuttle-187721/zones/us-west1-a/clusters/pach-cluster].
    kubeconfig entry generated for pach-cluster.
    NAME          ZONE        MASTER_VERSION  MASTER_IP       MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUSpach-cluster  us-west1-a  1.7.8-gke.0     104.198.10.149  n1-standard-4  1.7.8-gke.0   2          RUNNING
 ```   


  ${} are UNIX environmental variables which can be set up by simple assignment
  I configured 2 nodes of  ‘n1-standard-4’ machine which is a 4 virtual CPU machine as described [here](https://cloud.google.com/compute/docs/machine-types) 
  
6. Install different CLI utility to interact with different components
  a. install `kubectl` binary with `$ gcloud components install kubectl`  from  `google-cloud-sdk/bin` directory
  b. install `gsutil` - a tool that enables access to GCS from CLI as described [here](https://cloud.google.com/storage/docs/gsutil_install#deb)
  c. install `pachctl`  - a tool for to make and receive calls from Pachyderm cluster
```sh
    # For Linux (64 bit):
    $ curl -o /tmp/pachctl.deb -L https://github.com/pachyderm/pachyderm/releases/download/v1.6.5/pachctl_1.6.5_amd64.deb && sudo dpkg -i /tmp/pachctl.deb
```

7. Create a GCS Bucket  for Pacyderm with `$ gsutil mb gs://${BUCKET_NAME}`
  and deploy Pachyderm with 
  ```sh 
    $ pachctl deploy google ${BUCKET_NAME} ${STORAGE_SIZE} --dynamic-etcd-nodes=2 --dashboard
    serviceaccount "pachyderm" created
    storageclass "etcd-storage-class" created
    service "etcd-headless" created
    statefulset "etcd" created
    service "etcd" created
    service "pachd" created
    deployment "pachd" created
    secret "pachyderm-storage-secret" created
    
    Pachyderm is launching. Check its status with "kubectl get all"
```
$STORAGE_SIZE specifies size of  a persistent disk. For demo it can be set to 10 i.e 10 GB. Here 2 etc-nodes stores ***key-value*** Pachyderm metadata.


8. Clone Git repository from GH account 
    $ git clone https://github.com/anuragsoni9/mgmt690-pipeline.git


- Update `threat-detect.json` to include the SG API in root  `mgmt690-pipeline` foder
- Update `rule.json` in `mgmt690-pipeline/threat-detect` to include your email address

Putting in SendGrid key is an optional step, we will validate completion of this step by plotting the result.


9. Now this step is all about setting up of Pachyderm pipeline components. These are setup in Pachyderm and not on Unix so repo created will not be viewable by normal ls.
*Create*
```sh
    #Repos
    $ pachctl create-repo images
    $ pachctl create-repo model
    $ pachctl create-repo rules
    
    #Pipelines(Run from the root directory)
    $ pachctl create-pipeline -f validate.json
    $ pachctl create-pipeline -f object-detect.json
    $ pachctl create-pipeline -f threat-detect.json
    $ pachctl create-pipeline -f plot.json
 ```

*Push* 

    #####################################################################
    #Get Model from TensorFlow 
    $ wget http://download.tensorflow.org/models/object_detection/ssd_mobilenet_v1_coco_11_06_2017.tar.gz
    $ tar -xvf ssd_mobilenet_v1_coco_11_06_2017.tar.gz
    $ cd ssd_mobilenet_v1_coco_11_06_2017
    
    #Push object-detection model from TF ssd_mobilenet_v1_coco_11_06_2017 to the model repo 
    $ pachctl put-file model master -c -f frozen_inference_graph.pb
    ######################################################################
    #Push Rule.json
    $ pachctl put-file rules master -c -f rule.json

*Check*
At any point in time, running status of process/pods/cluster can be checked by 

    $ kubectl get pods 
    $ kubectl get all
    
    $ pachctl list-job 
    $ pachctl list-pipeline
    #$ pachctl inspect-pipeline <pipeline-name>
    $ pachctl list-repo
    
    #$ pachctl list-file <repo-name> <commit-id> <path/to/dir [flags]>
    $ pachctl list-file rules master
    
    $ pachctl inspect-pipeline threat-detect



10. Now is some time for action

Push the input file from `test images/` to Pachyderm 

    $ pachctl put-file images master -c -f dev1_1511891098.jpg
    
    $ pachctl list-job
    ID                                   OUTPUT COMMIT                                  STARTED        DURATION   RESTART PROGRESS  DL       UL       STATE
    8bdf8421-81ca-4bad-80c0-e915d7525e54 plot/73f4b873b97a4169919dca8426a5dd76          5 seconds ago  4 seconds  0       1 + 0 / 1 103B     13.67KiB success
    f0ede3ad-bc91-49c2-92b9-5a921521c575 threat-detect/ea61ae8995cb4aea9397c2a17280e82c 11 seconds ago 5 seconds  0       1 + 0 / 1 228B     103B     success
    b75a7963-dfb9-4484-b853-2158228fd6bc object-detect/599fb55fe6e64b99839862dba4a25b42 25 seconds ago 14 seconds 0       1 + 0 / 1 27.89MiB 103B     success
    be827b4f-3d12-4007-a8a0-8a281930c3fc validate/7bad879539fa484ebf7a87264603ca88      28 seconds ago 3 seconds  0       1 + 0 / 1 62.98KiB 62.98KiB success



11. If it ran successfully, you should have gotten an email with body coming in from `threatdetect.py` . Alternatively, if you didn’t put any Sendgrid key for email, you should be able to see the progress on a cool Pachyderm dashboard mentioned in next step.

<div class="image">
<img src="https://d2mxuefqeaa7sj.cloudfront.net/s_AB23734E1608EFF3C8081959BA1B9B5FE201B6019603F5869FCF83459FDB4577_1512447026828_image.png" alt="sample">
  <div  align="center"><i>sample email to customer recieved via Sendgrid API</i></div>
</div>
<br>



12. Deploy Pachyderm Dashboard on the GCS cluster with  `pachctl deploy local --dashboard-only`. Ensure if dashboard pod is running.


  To ensure GCS doesn’t block the viewing in your local do following:
  - Add Firewall Rule under VPC network with `IP ranges 0.0.0.0/0` and `tcp:38080` on *Specified protocols and ports*
  - Select `Ingress` for the *Direction*


  Next run,  `pachctl port-forward &`


  Now you should be able to get the dashboard running on your browser with `localhost:30080`


<div class="image">
<img src="https://d2mxuefqeaa7sj.cloudfront.net/s_AB23734E1608EFF3C8081959BA1B9B5FE201B6019603F5869FCF83459FDB4577_1512443837736_image.png" alt="sample">
  <div  align="center"><i>Pachyderm dashboard UI: generated piepline from JSON </i></div>
</div>
<br>


<div class="image">
<img src="https://d2mxuefqeaa7sj.cloudfront.net/s_AB23734E1608EFF3C8081959BA1B9B5FE201B6019603F5869FCF83459FDB4577_1512444203665_image.png" alt="sample">
  <div  align="center"><i>Pachyderm dashboard UI: repos and jobs </i></div>
</div>
<br>



## 

<div class="image">
<img src="https://d2mxuefqeaa7sj.cloudfront.net/s_AB23734E1608EFF3C8081959BA1B9B5FE201B6019603F5869FCF83459FDB4577_1512444681283_image.png" alt="sample">
  <div  align="center"><i>Result of the Production pipeline - image with identified object and Plot</i></div>
</div>
<br>


## Next Step

In the last post of the series, we will go over some strategies and steps needed to update, maintain and scale the pipeline. We will also discuss about the possible enhancements that can be implemented here and compare it with Hadoop infrastructure.
