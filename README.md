# Microservice App - PRFT Devops Training

This is the application you are going to use through the whole traninig. This, hopefully, will teach you the fundamentals you need in a real project. You will find a basic TODO application designed with a [microservice architecture](https://microservices.io). Although is a TODO application, it is interesting because the microservices that compose it are written in different programming language or frameworks (Go, Python, Vue, Java, and NodeJS). With this design you will experiment with multiple build tools and environments. 

## Components
In each folder you can find a more in-depth explanation of each component:

1. [Users API](/users-api) is a Spring Boot application. Provides user profiles. At the moment, does not provide full CRUD, just getting a single user and all users.
2. [Auth API](/auth-api) is a Go application, and provides authorization functionality. Generates [JWT](https://jwt.io/) tokens to be used with other APIs.
3. [TODOs API](/todos-api) is a NodeJS application, provides CRUD functionality over user's TODO records. Also, it logs "create" and "delete" operations to [Redis](https://redis.io/) queue.
4. [Log Message Processor](/log-message-processor) is a queue processor written in Python. Its purpose is to read messages from a Redis queue and print them to standard output.
5. [Frontend](/frontend) Vue application, provides UI.

## Architecture

Take a look at the components diagram that describes them and their interactions.
![microservice-app-example](/arch-img/Microservices.png)


# DOCUMENTATION

hi, welcome to my project where we try to automate everything, it was not fun but I hope it works for you.

We are going to work with azure, terraform, ansible and docker to deploy a microservice application, this is a simple todo app that uses multiple microservices to do its job, we are going to use terraform to create the infrastructure in azure, ansible to configure the servers and docker to run the application.

Most stuff runs on github actions so you can see the workflow files in the `.github/workflows` folder.

## first worflow

first, theres the `docker-build.yml` in the workflows folder, this is the main workflow that builds all the docker images and pushes them to your docker hub account, you need to set up a secret called `DOCKER_USERNAME` and `DOCKER_PASSWORD` in your github repo settings, this is the username and access token as 
your password for your docker hub account. This will create two images for each time you push to the master branch, first it will create an image with the tag
latest, and one with the tag with the current time, each time you push it will overwrite the latest image, however you'll end up with all the previous images with the
time tag, so you can always go back to a previous version if you need to.

## second worflow

Then, there's the big one, in the deploy.yml we do a lot of things let's go through it

### azure login

First we need to login to azure, this is done with the `azure/login` action, you need to set up a secret called `AZURE_CREDENTIALS` in your github repo settings, 
what you need here is the output of the following command:

```bash
az ad sp create-for-rbac --name "github-terraform" --role="Contributor" --scopes="/subscriptions/<SUBSCRIPTION_ID>" --sdk-auth
```

You past this output in the secret, this will give you access to your azure account and allow you to create resources.

Remember to replace `<SUBSCRIPTION_ID>` with your subscription id, you can find it in the azure portal. Also, for further secretes, the secretes go with out: " "

We also need some other secretes for the workflow to work, we need to set up the following secretes in your github repo settings:
- `TF_VAR_USERNAME`: the username you want for the vms
- `TF_VAR_PASSWORD`: the password you want for the vms
- `ARM_CLIENT_ID`: the client id of the service principal you created
- `ARM_CLIENT_SECRET`: the client secret of the service principal you created
- `ARM_SUBSCRIPTION_ID`: the subscription id of the service principal you created
- `ARM_TENANT_ID`: the tenant id of the service principal you created

you'll find most of this information in the output of the command we ran before, but you can also find it in the azure portal.
You can always create more secrets for more users, but for now we are going to use the same user for all the vms.

### terraform

Now, terraform is still a bit tricky, let me explain: normally, terraform uses a file called `terraform.tfstate` to keep track of the resources it creates, 
this file is created in the same folder as the terraform files, however, if we push this file to github we'll have a 
big problem, as it contains information like your subscription id, client id, client secret and other sensitive information. 

This is where the tricky part comes in, we need to store it in a new location so that's safe. We can do this using the backend.tf file present.
However, this will cause a problem, as terraform will try to create the backend BEFORE creating the resource where will store the state file, and
we can't use dynamic naming in the naming file. My recommendation is to remove the backend.tf temporarily, run the workflow, and then add it back again.
This will create the resource group and storage account where the state file will be stored, and then you can add the backend.tf file back again.
With this, the state file will be stored in the storage account and you won't have to worry about it anymore.

if this does not work, do the same process but running it locally, if it works the workflow should work too, as it will now know the state of the infrastructure.

Also, apart from the vm for the application, we create a vm for sonar, this is a code quality tool that will help you to see the quality of your code, it will run a scan on the code and show you the results in the sonar dashboard, you can access it using the ip of the vm.

### ansible

Now with all the infrastructure we can create and run the application, we need to install ansible and run the playbook. you need to put the name of the 
vm and rg if you changed it, the workflow uses them to get the ip of the newly created vm so it can install the information.
It automatically gets the ip of the vm and runs the playbook.

With this, everything should be up and running, you can check the logs of the workflow to see if everything worked.

**WARNING**: If you change the name of the resource group OR the name of the vm, you need to change it in the workflow file too, as it uses the name of the vm to get the ip address of the vm. Heres the code fragment
```yaml
RESOURCE_GROUP="<YOUR_RESOURCE_GROUP>"
VM_NAME="<YOUR_VM_NAME>"
VM_NAME_SONAR="<YOUR_VM_NAME_SONAR>"
IP=$(az vm list-ip-addresses --name "$VM_NAME" --resource-group "$RESOURCE_GROUP" --query "[0].virtualMachine.network.publicIpAddresses[0].ipAddress" -o tsv)
IP_SONAR=$(az vm list-ip-addresses --name "$VM_NAME_SONAR" --resource-group "$RESOURCE_GROUP" --query "[0].virtualMachine.network.publicIpAddresses[1].ipAddress" -o tsv)
```

### Sonar
In the ansible playbook we also install sonar, this is a code quality tool that
will help you to see the quality of your code, it will run a scan on the code and
show you the results in the sonar dashboard, you can access it using the ip of the vm.

### Selenium
The last part of the workflow is the selenium test, this is a simple test that 
checks if the application is running and if the login works, and creates a todo
item, you can check the logs to see if it worked. It runs in a docker container in
the githubactions runner, so it should work without any problems.

There is a extra step that is currently commented, if you want to run it locally
you can create a venv with the requirements.txt and run it using behave. You should
also uncomment the headless option in the environment.py file. 
If you run it and get a page that warns the page is not secure, un comment the 
proceed method and it should work fine.

If for any reason the workflow fails to get an ip address for selenium to test it
will default to the localhost.

## Diagram

For a visual of the workflow, at [This diagrama](diagrama_auto.pdf) 


