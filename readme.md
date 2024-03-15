
# MLOps Pipeline Environment Setup

Start the containers with docker-compose.
```bash
docker compose up -d
```

## .env File

For convenience, in this tutorial we use the default .env file, creating a copy of .env.example in .env
```
cp .env.example .env
```

## Network setup
Add the following entries to your /etc/hosts:
```
127.0.0.1	gitea-server
127.0.0.1	jenkins
127.0.0.1	postgres-single
```

## Gitea
- Open the [dashboard](gitea-server:3000), leave everything unchanged, only adding the administrator user section at the bottom of the page, entering:
    - name: gitea
    - password: gitea
    - email: git@tea.com
- Add SSH public key in the user profile
- Create a repository ("mlops-pipeline" in this example)
- Command to add remote origin:
    ```bash
    git remote add origin ssh://git@gitea-server222/gitea/mlops-pipeline.git
    ```
- Modify the app.ini file (gitea/gitea/conf/app.ini) so it has:

        ROOT_URL         = http://gitea-server:3000/
        ...
        [webhook]
        ALLOWED_HOST_LIST = jenkins
    

## Jenkins

### Configuration
- Open the [dashboard](jenkins:8080). Retrieve and enter the admin password with the command:
  ```bash
    docker exec -it jenkins bash
    cat /var/jenkins_home/secrets/initialAdminPassword
    ```
    or by opening the container log.

- Install the suggested add-ons.
- Create the admin user:
    - username: jeknins
    - password: jenkins
    - full name: jenkins
    - email: jenkins@jenkins.jenkins
- Provide the URL for Jenkins: http://jenkins:8080
- Once the environment is configured, add the Gitea add-on and configure it:
![alt text](img/gitea-jenkins.png "Gitea Add-on Configuration")

### Pipeline
Create a new pipeline in Jenkins and configure it as follows:
- ![alt text](img/build-triggers.png "Build Triggers")
- ![alt text](img/pipeline.png "Pipeline Configuration")

Note: With this configuration, a Jenkinsfile must be present in the repository.


## Useful Links

- https://wiki.gcube-system.org/gcube/Gitea/Jenkins:_Setting_up_Webhooks

- https://discourse.gitea.io/t/webhooks-between-gitea-and-jenkins/2760

## Serving

The serving tools has been deployed in a K8s environment, MicroK8s in this case. The obersability, ingress, registry, helm3 and dns add-hons has to be enabled. Because TorchServe is not a cloud native tool a bit of configuration is needed to deploy it in a K8s environment.

### TorchServe

For first we need to create a namespace for TorchServe:

```bash
kubectl apply -f torchserve/name-spaces_torchserve.yaml
```

To deploy a TorchServe model a local NFS server (where the ML model are saved) has been used. It is possible to install with:

#### Local NFS server

PLEASE NOTE that all the following files are located in the nfs folder.

We used an NFS server to share the models with all nodes of the cluster. To do this you must install NFS server on your machine and configure it correctly. You can do it with this command:

```bash
sudo apt install nfs-kernel-server
cd nfs
```

After this you must create a directory that will be shared with all nodes of the cluster. Moreover, the TorchServe configuration file has to be copied in this directory. You can do it with these commands:

```bash
sudo mkdir -p /media/nfs/
sudo mkdir -p /media/nfs/shared
cp config.properties /media/nfs/shared
sudo mkdir -p /media/nfs/shared/model_store
```

Where `/media/nfs/shared/model_store` is the shared directory where the models are saved.

After this you must configure the NFS server to share the directory that you have created. You can do it with this command:

```bash
sudo nano /etc/exports
```

And add this line at the end of the file:

```bash
/media/nfs/ *(rw,sync,no_root_squash,no_subtree_check)
```

Where `/media/nfs/` is the path of the directory that you have created. See [this](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-ubuntu-20-04) for more. After this syep you must restart the NFS server with this command:

```bash
sudo systemctl restart nfs-kernel-server
```

Now you can create a persistent volume and a persistent volume claim to bind the NFS server with your cluster.
The YAML necessitates a bit of configuration to match the NFS server IP and the path of the shared directory. So in pv_nfs.yaml "IP_ADDRESS_OF_NFS_SERVER" with the actual ip address. Then execute the following commands:

```bash
kubectl apply -f pv_nfs.yaml
kubectl apply -f pvClaim_nfs.yaml
```

This pvc will be used by TorchServe to load saved models.

#### Deploy TorchServe

In this step you have to deploy torchserve cluster on kubernetes. You can do it with these command to configure your cluster correctly:

```bash
cd ..
kubectl apply -f torchserve/deployment_torchserve.yaml
kubectl apply -f torchserve/service_torchserve.yaml
```

The last step is to configure a K8s Ingress to access the services from the outside. 
Before applying the ingress files you must change the IP address in the file `ingress_management_torchserve.yaml` and `ingress_predictions_torchserve.yaml` with the actual IP address of your node.

```bash
kubectl apply -f torchserve/ingress_management_torchserve.yaml
kubectl apply -f torchserve/ingress_predictions_torchserve.yaml
```

```bash
curl URL_ADDRESS_OF_PREDICTIONS_INGRESS/ping
```

### Yatai

Yatai as benn deploy following the instruction in the [official documentation](https://docs.yatai.io/en/latest/installation/index.html). Yatai provides an easy to use web GUI to manage the models and the experiments. To access the dashboard an ingress has been configurated (please modify the file adding the ip of the node):

```bash
kubectl apply -f yatai/yatai_ingress.yaml
```

The web GUI is accessible at the address `http://INGRESS_URL`. From this dashboard it is possible to manage and deploy the models and the experiments. It is also possible to deploy a local model using BentoML CLI configurated with a token created in the Yatai web GUI. The process involves the following steps:

1. Create a new token in the Yatai web GUI
2. Configure the BentoML CLI with the token
3. Push the model with the BentoML CLI
4. Deploy the model from the GUI