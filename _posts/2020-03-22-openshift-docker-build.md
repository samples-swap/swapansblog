---
layout: post
published: true
title: openshift shift binary image build
date: '2020-03-22'
subtitle: 
summary: finally oracle drivers are in maven central
---
## Base image creation

====================
**Create a build config for  binary docker build**
Creates build config objects. This sets up a docker build in openshift that pushes to artifactory using the openjdk image as a base.
instructions on sourcing the oc binary [oc-binary-source.md](oc-binary-source.md) 
```bash
oc new-build --name=base-mule-image-build --binary=true --to=openshiftdockerregistry.local.com/base-mule-image:latest --to-docker=true --strategy=docker --image-stream=bac/openjdk18:latest
```
**clone the base build repo**
```bash
git clone ssh://****.git
```
**cd to the cloned project folder and do a docker build and push base image to repo**
```bash
cd $proj_folder
oc start-build base-mule-image-build --from-dir=.
```

## App base image layered on binary base image
====================
This sets up a docker build in openshift, which uses the previously created mule base image as the base layer.  
It pushes the final image to artifactory
**create App image build config**
```
oc new-build --name=app-mule-image-build --binary=true --to=openshiftdockerregistry.local.com/app-mule-image:latest --to-docker=true --strategy=docker --docker-image=openshiftdockerregistry.local.com/base-mule-image:latest
```
**Starts the build for the app image.  Run this in the directory with the application docker file and the application jar**
```
oc sta