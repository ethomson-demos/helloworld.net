## Initial Configuration

1. Set up an Azure Container Service and registry.  Make a note of the
   registry name and substitute it in the following examples.  Future
   sample text uses "registryname".

2. Push up a default image to use for the app service, before we've built
   and pushed our image.  I recommend the ASP.net demo application
   (`microsoft/dotnet-samples:aspnetapp`).  You'll need to upload this
   to your own registry; during a deploy you can't switch a web app from
   being backed by a public container registry to a private one.  Use a
   tag of `0` to avoid colliding with the tags produced in the build, which
   will start at version number `1`.

       docker pull microsoft/dotnet-samples:aspnetapp
       docker tag microsoft/dotnet-samples:aspnetapp registryname.azurecr.io/helloworld:0
       docker push registryname.azurecr.io/helloworld:0

3. Create an azure web app to run the workload.  Create a "Web App for
   Containers".  I use Linux.  Give it a name; in future examples we'll
   use "dotnetdemo".  In the Configure Container, select the source as
   Azure Container Registry.  Choose "registryname" as the Registry.
   Choose "helloworld" as the Image.  Choose "0" as the Tag.  Then
   click Apply, and finally Create.

4. Once created, get the deployment access keys for your container
   registry.  Navigate to the container, then to Access Keys.  Set
   Admin User to Enabled.  Make note of your Username and Password.

5. Navigate to http://dotnetdemo.azurewebsites.net/; you should see the
   ASP.NET demo landing page.

6. Configure your Azure Container Service secrets for your pipeline (so
   that you don't have to add them directly to the pipeline configuration
   or otherwise show them during the demo).

7. Open your Azure DevOps project; navigate to Pipelines then Library.
   Create a Variable group; name it "azuresecrets".  Create the following
   variables, using the username and password as specified in the
   container registry configuration in step 4:

   1. `containerRegistry` = `registryname.azurecr.io`
   2. `containerRegistryUsername` = `registryname`
   3. `containerRegistryPassword` = `password`

8. Create a new repository, either a fork of or containing the contents
   of https://github.com/ethomson-demos/helloworld.net.

## Demo Script

1. Create a build Pipeline, pointing to your git repository.  In the
   YAML phase, you'll want to provide new YAML:

       # Docker image
       # Build a Docker image to deploy, run, or push to a container registry.
       # Add steps that use Docker Compose, tag images, push to a registry, run an image, and more:
       # https://docs.microsoft.com/azure/devops/pipelines/languages/docker

       trigger:
       - master

       pool:
         vmImage: 'Ubuntu-16.04'

       variables:
       - name: imageName
         value: 'helloworld:$(build.buildId)'
       - group: azuresecrets

       steps:
       - script: docker build -f Dockerfile -t $(imageName) .
         displayName: 'docker build'
       - script: |
           docker login -u $(containerRegistryUsername) -p $(containerRegistryPassword) $(containerRegistry)
           docker tag $(imageName) $(containerRegistry)/$(imageName)
           docker push $(containerRegistry)/$(imageName)

   This new YAML will run the build (by executing `docker build, which
  invokes the .NET compiler and produces a docker container of the
  application), tag it with your container registry name and the image
  name, and then uploads it to your container registry.

   (Private container registry images need to be tagged with the container
  registry name.  eg, `registryname.azurecr.io/helloworld:0`).

2. Create a deployment pipeline to deploy the container from the registry
   to your web app.

   1. Select an Azure App Service deployment.
   2. Add an artifact.  This deployment will be dependent on the prior
      build pipeline, so select the build pipeline you created in step 1.
   3. In your stage settings, select and authorize your Azure subscription.
      Set your App type to Web App for Containers (Linux), then select the
      app service to deploy to.  In the registry, select your container
      registry, `registryname.azurecr.io`.  For Image, select `helloworld`,
      and for Tag, enter `$(Build.BuildId)`.  This will allow you to deploy
      the results of a build, since the image published is tagged with the
      build ID.

## Operation

Now, queue a build.  Once the build has finished, you can deploy it.
Go to the releases tab, and Create a Release.  In the artifacts, you
can select the artifact that you want to deploy in the Version tab.
