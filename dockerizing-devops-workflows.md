# Dockerizeing DevOps Workflows

"It works on my machine" is often associated with application development when developers test something locally but breaks something in production.

We have all been there, trying to run a command line utility and something isn't right, something has changed, and your peers respond with "I can run it from my machine", sound familiar?Each day I am using the Azure CLI, Terraform, and Ansible. The Azure CLI and Ansible both require Python, and it just so happens that they use different versions as well. Of course Python can run side by side with each other and for a long time things just worked. Until they didn’t… So how can Docker help?

What if we build a docker image that can be used to setup an environment that is purpose built for the utility we are trying to run?

Break down the problems and try to solve them.

* Running workflows locally for development doesn’t always work
* I may be working on a new process tool and it may take some time to get setup, how can I ensure my team can use that utility, or that a build server can also run my commands?
* Layering on additional functionality, easily.

Docker to the rescue.

Let us go through an example that is "for the sake of argument/blog post" that also just so happens to be very real and something I use everyday.

I am a member of a Cloud DevOps team that is responsible for creating, configuring and maintaining cloud infrastructure in Azure. A common and very powerful tool at my disposal is the Azure CLI 2.0. My goal is to create a docker image that my team and I can use to easily use this utility.

The high level steps that I will perform are as follows, keep in mind that these steps could be done regardless of what utility you wish to dockerize:

1. Create a github repo that contains a \`Dockerfile\` to create an environment to work out of and any other files you may need.
2. Build and test the image locally to ensure things are working as expected.
3. Create a DockerHub repo to house the docker image for all to see.
4. Setup continuous image building to capture changes to your image.
5. Share with your team.

Let's get started.

## Setup Docker

Visit [https://docs.docker.com/engine/installation/](https://docs.docker.com/engine/installation/) and follow the setup for your operating system.

The folks at docker.com have made it very easy to install Docker so you shouldn’t have any problems here. For this I happen to be using Docker For Windows which allows me to build linux and windows images.

We can ensure docker is running by firing up our favorite terminal and running a \`docker version\` command.

You should see something like this:

```bash
$ docker version
Client:
 Version:      17.06.2-ce
 API version:  1.30
 Go version:   go1.8.3
 Git commit:   cec0b72
 Built:        Tue Sep  5 20:00:17 2017
 OS/Arch:      linux/amd64

Server:
 Version:      17.06.2-ce
 API version:  1.30 (minimum version 1.12)
 Go version:   go1.8.3
 Git commit:   cec0b72
 Built:        Tue Sep  5 19:59:19 2017
 OS/Arch:      linux/amd64
 Experimental: true
```

## Setup git repository

We need a place to put all of our hard work, so let's create a git repository and push our initial changes to [https://github.com/](https://github.com/).  
This is straight forward but if you need some help head over here for more information [https://help.github.com/articles/create-a-repo/](https://help.github.com/articles/create-a-repo/)  
Once this is set up we are ready to start adding files. It is always a good idea to create a README.md file to describe your repository.

## Dockerfile

A Dockerfile represents how our docker image will be built and if you need to know more about the specifics, take a look at the docs [https://docs.docker.com/engine/reference/builder/](https://docs.docker.com/engine/reference/builder/). Everything we will do here is going to be very simple so I wouldn't stress the specifics just yet.

We need to setup an environment that can run the azure cli, lucky for us the azure cli team has done this already.  
Rather than repeat what they have done in their image, we will just use that image as our base. This also has the added benefit that we can get any changes that the team developing the azure cli might make in the future.

```Dockerfile
FROM azure-cli:latest

CMD bash
```

Let's go ahead and build this and test out what we have.

```bash
$ docker build -t azhelper .
Sending build context to Docker daemon  113.2kB
Step 1/2 : FROM azuresdk/azure-cli-python:latest
 ---> b95f51b22e75
Step 2/2 : CMD bash
 ---> Running in f0009bc62755
 ---> 404cf2421bd4
Removing intermediate container f0009bc62755
Successfully built 404cf2421bd4
Successfully tagged azhelper:latest
```

This command will build the image locally, which I can then run a container from.

```bash
Docker run -it azhelper
Bash:
```

Now that we are running a container where we can execute the azure cli from.  
Go ahead and try `az accounts list`  
Oops, we need to login, try a `az login`. A web address will show up with a code that can be used to authenticate against Follow those steps and you should see a success:

`az account list`

Question: Are we going to have to login every time?  
Answer: Yes! If we leave it this way.

To fix this problem, lets exit out of our running container and map a volume where our login access tokens can be stored and persisted outside the container.

`docker run -it -v ${HOME}/.azure:/root/.azure straubt1/azhelper:latest`  
What are we doing here is mapping a folder on my machine INTO the container that can be used by the CLI to store needed information. This will allow us to start/stop the container and not be required to login every time.

At this point you may ask, what have we really done here? Why don’t we just use the azure-cli image directly. To that I say, but there is more.  
If you JUST wanted the CLI and that was it, just use the base image and you are good.

However, what if you wanted to add things that USED the CLI to make some of your common tasks easier? That is exactly what we are wanting here and we will dive into in the next section

## Taking another step

As someone who uses the Azure portal I will tell you it can be a source of relief for quick tasks, and a source of immense pain for repeated tasks. Things quickly fall apart at scale and if you are dealing with hundreds of resources and is exacerbated if they are across multiple subscriptions.

As a DevOps engineer working in Azure, some of the common requests I get are:  
    • Turn on/off/restart every VM in several resource groups  
    • Check the current power state of several resources groups  
    • Given an IP address, what is the name of the VM

Back in our git repo lets add a scripts folder and some common `az` CLI calls I find myself performing every day. BASH is the flavor we are using here and there are several more functions being added, I will only cover a few to get us through the overall process.

In the scripts folder I create a file `search.sh` that will contain functions that are related to searching for resources \(namely resource groups and VM's\). The calls here are basic but it should be obvious why having these available to you can save you a lot of time.

```bash
# search for Resource Group by name
function search-group () {
    query=$1
    az group list --query "[?name | contains(@,'$query')].{ResourceGroup:name}" -o table
}

# search for VM by name
function search-vms () {
    query=$1
    az vm list --query "[?name | contains(@,'$query')].{ResourceGroup:resourceGroup,Name:name}" -o table
}
```

Note: The query language used by the Azure CLI 2.0 is a standard called JMESPath \(insert link\) which is a far cry from the where we were with the original CLI that had no built in querying. Instead you were forced to output in JSON and pipe to something like jq \(insert link\). Of course you could still use this approach for 2.0, but I find the syntax much easier to follow for JMESPath than jq, it is also a standardize spec. Alright, end of rant, back to Dockerizing.

Now lets get these script into the container. I could just copy this single script, but knowing I am going to want to build on these scripts in the future, let's assume that we will have an entire folder of scripts.

```Dockerfile
COPY scripts/ scripts/
```

Now we need a way to load these scripts into the environment so that they are available when we run a container.  
More BASH love here, let's insert some dynamic awesomeness into our `.bashrc` file so that this gets loaded every time.

```bash
RUN echo -e "\
for f in /scripts/*; \
do chmod a+x \$f; source \$f; \
done;" > ~/.bashrc
```

This may look a bit wild, but I assure you it is of the simplest intent. Any time that BASH loads, anything in the `scripts` folder will get sourced and the functions made available. This will become even more beneficial as I will show later \(setting hooks to keep you reading!\).

Let's fire up another build and start a new container.

```bash
Docker build
Docker run -it
$bash: az account list
LIST
$bash: search-group testgroup

$bash: search-group test

Search-vms test
```

Things are looking good, push your changes up to github to save all the good work.

## Dockerhub

So we have created this awesome little container to run the Azure CLI from anywhere, and even have room to grow some handy functions for common use. But all this docker building seems a lot like shipping a code and requiring the end user to build it \(I am looking at you linux…\), let's address this. Now I am going to use Dockerhub.io since this is a completely open source and public image \(we also get automatic builds\), but the same concepts could be applied to a private/on-prem setup.

```bash
docker push straubt1/azhelper:latest
```

will push our latest build up to Dockerhub.  
Success! We are done! Or are we?

Remember I told you I was a DevOps Engineer and how good would I be if I left this in a state that required manually building and pushing up any time there was a change?

Login to Dockerhub and view your dashboard where you should see your first image push.  
Now lets add an integration to the github repo to allow for automatic builds.

Note: Dockerhub will only provide this free service if you github repo and docker image are both publicly available. If you were private/on-prem, similar output could be found by using your build server to handle this for you.

Once the integration is done we can set up triggers to determine when to build a new image and what tags to apply. For this example I am going with the most basic, checkins on the master branch will result in a new build that is tagged `latest`.

If I go to the "Build Details" pages I can manually trigger a build, then see the status as it builds.  
\(Show build details\)  
These steps should look familiar to what you were seeing locally, but now it is all done in the cloud.

> Using two cloud hosted services \(github.com and dockerhub.io\) to build a docker image that contains a CLI tool used for deploying/configuring cloud services

## Customizing

_might take this out_

Remember earlier I mentioned that creating the BASH to load everything in the `scripts` directory could also have benefit.  
Consider you have functions that you don't want in a public repository.  
You could map that local folder into the scripts folder and they will be loaded at runtime.

## Conclusion

We took a utility that we use locally, dockerized it, added some additional functionality and now everyone on your team can access it.  
Running in this manner should eliminate the "it doesn’t work anymore" problems since everyone is essentially running the same container.  
Of course we have also have solved another problem. What if I have a need to access the Azure CLI from a build server? Well, now all you need is this image and the ability to map credentials into the container.

This has been a simple yet powerful example of how to dockerize a utility.

