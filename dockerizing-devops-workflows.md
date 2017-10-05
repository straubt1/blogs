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

Visit https://docs.docker.com/engine/installation/ and follow the setup for your operating system.

The folks at docker.com have made it very easy to install Docker so you shouldn’t have any problems here. For this I happen to be using Docker For Windows which allows me to build linux and windows images.



We can ensure docker is running by firing up our favorite terminal and running a \`docker version\` command. 

You should see something like this:





`$ docker version`

`Client:`

` Version:      17.06.2-ce`

` API version:  1.30`

` Go version:   go1.8.3`

` Git commit:   cec0b72`

` Built:        Tue Sep  5 20:00:17 2017`

` OS/Arch:      linux/amd64`

``

`Server:`

` Version:      17.06.2-ce`

` API version:  1.30 (minimum version 1.12)`

` Go version:   go1.8.3`

` Git commit:   cec0b72`

` Built:        Tue Sep  5 19:59:19 2017`

` OS/Arch:      linux/amd64`

` Experimental: true`



