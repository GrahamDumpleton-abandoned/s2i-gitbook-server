# GitBook Site Generator

This repository implements a [GitBook](https://www.gitbook.com/) site generator and hosting service. It is implemented as a [Source-to-Image](https://github.com/openshift/source-to-image) (S2I) builder and can be used to generate a standalone Docker-formatted image which you can run with any container runtime supporting Docker-formatted images. Alternatively the S2I builder can be used in conjunction with OpenShift to handle both the build and deployment of the generated GitBook formatted site.

## Integration With OpenShift

To use this with OpenShift, it is a simple matter of creating a new application within OpenShift, pointing the S2I builder at the Git repository containing your GitBook document source.

As an example, to build and host the awesome [Django tutorial](https://github.com/DjangoGirls/tutorial) created by [DjangoGirls](https://djangogirls.org), you only need run:

```
oc new-app getwarped/s2i-gitbook-server:2.3~https://github.com/DjangoGirls/tutorial.git --name djangogirls

oc expose svc/djangogirls
```

To have any changes to your document source automatically redeployed when changes are pushed back up to your Git repository, you can use the [web hooks integration](https://docs.openshift.com/container-platform/latest/dev_guide/builds.html#webhook-triggers) of OpenShift to create a link from your Git repository hosting service back to OpenShift.

Do be aware that GitBook appears to a bit of a memory hog and for the Django Girls tutorial can require as much as 1Gi during the build phase. If your OpenShift installation defines default limits on memory of 512Mi or less for time bound compute resources, you would need to edit the build configuration and increase the memory limit to 1Gi.

To make it easier to override the build time memory limit and deploy a GitBook site, a template for OpenShift is also included. This can be loaded into your project using:

```
oc create -f https://raw.githubusercontent.com/getwarped/s2i-gitbook-server/master/template.json
```

Once loaded, select the ``gitbook-server`` template from the web console when wanting to add a new site to the project.

The template details are:

```
Name:		gitbook-server
Description:	GitBook Server
Annotations:	tags=instant-app,gitbook

Parameters:
    Name:		APPLICATION_NAME
    Description:	The name of the application.
    Required:		true
    Value:		gitbook-server
    
    Name:		INSTANCE_ID
    Description:	ID to bind application components.
    Required:		true
    Generated:		expression
    From:		[a-z0-9]{5}
    
    Name:		SOURCE_REPOSITORY
    Description:	Git repository for source.
    Required:		true
    Value:		<none>
    
    Name:		SOURCE_DIRECTORY
    Description:	Sub-directory for source files.
    Required:		false
    Value:		<none>
    
    Name:		BUILD_MEMORY_LIMIT
    Description:	Build time memory limit.
    Required:		true
    Value:		512Mi

Objects:
    ImageStream		${APPLICATION_NAME}
    ImageStream		${APPLICATION_NAME}-s2i
    DeploymentConfig	${APPLICATION_NAME}
    Service		${APPLICATION_NAME}
    Route		${APPLICATION_NAME}
```

The ``APPLICATION_NAME`` and ``SOURCE_REPOSITORY`` should be specified. Override the ``BUILD_MEMORY_LIMIT`` and set it to a higher memory value if the ``gitbook`` application keeps getting killed when processing the site data.

The template applies the label ``appid`` to all resource objects created. The value of the label is constructed using ``${APPLICATION_NAME}-${INSTANCE_ID}``. There is no need to supply ``INSTANCE_ID`` as it will be automatically populated.

To delete all the resource objects created using the template, determine the value of the ``appid`` label and then use ``oc delete all`` to delete them. For example:

```
$ oc delete all --selector appid=gitbook-server-fwo66
buildconfig "gitbook-server" deleted
imagestream "gitbook-server" deleted
imagestream "gitbook-server-s2i" deleted
deploymentconfig "gitbook-server" deleted
route "gitbook-server" deleted
service "gitbook-server" deleted
```

If ``oc new-app`` was used directly against the S2I builder image, you should instead use the ``app`` label and the value assigned to it by ``oc new-app`` when using ``oc delete all`` to delete all the resource objects created.

It is possible to use the ``app`` label if deleting all resource objects after having used the template as well, but you would need to have overridden the ``app`` label to be a unique value when filling out the template from the web console else it isn't unique to the application.

## Standalone Docker Images

To create a standalone Docker-formatted image, you need to [install](https://github.com/openshift/source-to-image/releases) the ``s2i`` program from the Source-to-Image (S2I) project locally. Once you have this installed, you would run within your Git repository:

```
s2i build . getwarped/s2i-gitbook-server:2.3 mygitbooksite
```

In this case this will create a Docker-formatted image called ``mygitbooksite``. You can then run the image using:

```
docker run --rm -p 8080:8080 mygitbooksite
```