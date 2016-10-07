---
layout: post
title:  "Service Fabric Packager"
date:   2016-10-07 14:30:00 +0200
categories: service-fabric
---

Today I'm going to talk a bit about how we package Service Fabric applications before deployment at work.

Let me first describe our application setup

We have a total of 7 ApplicationTypes:
* 4 of these are single instance applications that each has 1 service.
* The 5th is an application for our Api. Single application instance with a few services
* The 6th is the core shared backend services. Single application instance with a bunch of services (around 25). It has both Stateless and statfule services and actors
* The last application type is an application type we run multiple instances of, 1 per customer. It consists of a handful of services (stateless/stateful actors/services).

All our things are in one VS solution with loads of projects. This is cool :)

We consider the above application types to be _one thing_. As in, we don't say we need to deploy an update to the API, we deploy an update to all the things, but it might be that the API is the only thing that has changed.

Also, let me just clarify some terms I'll be using:
* _ApplicationPackage_: All the files needed to upgrade an ApplicationType.
* _ServicePackage_: All the files needed to upgrade a Service.
* _SubPackage_: For lack of a better word, this refers to the _CodePackage_, _ConfigPackage_ and _DataPackges_ inside a _ServicePackage_

#The beginning#
Ok, so now that we've got all that defined, back when we started with SF there weren't any tooling to manage versions inside the manifest. So we used Visual Studio (VS) to package each application we needed, and manually updated the versions inside the manifests. We didn't do any partial upgrades. This was a bit of a chore, as we also had to add stuff like CertificateInfo to the manifest and make sure the endpoints were defined correctly.

Loads of manual things, but it was ok, since we were just starting out.

Back then, there were also a bug in VS that caused the _F5_ experience to not work when you reached a certain amount of services. VS add some debug application parameters when you do F5, and those were based on the Service you have. Back then, if these parameters were bigger than 4096 bytes, it threw a null ref and stopped.

So we decided to try and live without using F5. We developed some custom deploy scripts (basically, you would package in VS and then run a script that did the upload, registering and upgrade). And we would Attach to Process in VS on the processes we wanted to debug. This worked out great for us, and still to this day, we don't use _F5_ in VS to launch debug sessions. We just attach to process.

Anyways, we had these scripts, but they didn't really handle endpoints, certificates and stuff, so for the longest time, I was manually fixing our deploy packages when deploying to our staging/prod clusters. I started doing partial upgrade packages by hand as well. Since we all know that humans do stupid things and have a tendancy to forget things, so did I, and often I had to start over with packaging since I'd forgotten something. This sucked, and deploying code was no fun, only irritation.

So, we decided to fix it once and for all (for us at least).

#Requirements#

We created a list of requirements that the tooling of our choice had to have:
* It should know what have changed between versions
* It should know how to package for partial upgrades
* It should just fix all the version numbers in our manifests
* It should be able to configure our endpoints and certificates in the manifests
* It should work running locally to package for one-node, but also work when running on a build agent packaging for staging/prod clusters
* It should be able to package multiple application types in one go as one _thing_
* The cluster it is packaging for is the master

Also, bonus requirements:
* Making tooling can be fun (and I don't like scripting comples things in powershell)
* I'd like to make a .NET Core CLI app that is actually useful

So, we decided to make a command line packager in .NET Core.

#The packager#
The [Service Fabric Packager](https://github.com/proactima/ServiceFabricPackager) is a .NET Core CLI program creates an application package for each of your application, picking only things that have changed (and by changed, I also mean that if you change only a ServiceManifest for a service, only that will be packaged).

##How it works##
