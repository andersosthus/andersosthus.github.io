---
layout: post
title:  "Service Fabric Packager"
date:   2016-10-07 14:30:00 +0200
categories: service-fabric
---

Today I'm going to talk a bit about how we package Service Fabric applications before deployment at work. And after writing this, I realise it's a lot longer than I intially intended :)

Let me first describe our application setup.

We have a total of 7 ApplicationTypes:

* 4 of these are single instance applications that each has 1 service.
* The 5th is an application for our Api. Single application instance with a few services
* The 6th is the core shared backend services. Single application instance with a bunch of services (around 25). It has both Stateless and statfule services and actors
* The last application type is an application type we run multiple instances of, 1 per customer. It consists of a handful of services (stateless/stateful actors/services).

All our things are in one VS solution with loads of projects. This is cool :)

We consider the above application types to be **one thing**. As in, we don't say we need to deploy an update to the API, we deploy an update to all the things, but it might be that the API is the only thing that has changed.

Also, let me just clarify some terms I'll be using:

* **ApplicationPackage**: All the files needed to upgrade an ApplicationType.
* **ServicePackage**: All the files needed to upgrade a Service.
* **SubPackage**: For lack of a better word, this refers to the _CodePackage_, _ConfigPackage_ and _DataPackges_ inside a _ServicePackage_

# The beginning

Ok, so now that we've got all that defined, back when we started with SF there weren't any tooling to manage versions inside the manifest. So we used Visual Studio (VS) to package each application we needed, and manually updated the versions inside the manifests. We didn't do any partial upgrades. This was a bit of a chore, as we also had to add stuff like CertificateInfo to the manifest and make sure the endpoints were defined correctly.

Loads of manual things, but it was ok, since we were just starting out.

Back then, there were also a bug in VS that caused the **F5** experience to not work when you reached a certain amount of services. VS add some debug application parameters when you do F5, and those were based on the Service you have. Back then, if these parameters were bigger than 4096 bytes, it threw a null ref and stopped.

So we decided to try and live without using F5. We developed some custom deploy scripts (basically, you would package in VS and then run a script that did the upload, registering and upgrade). And we would Attach to Process in VS on the processes we wanted to debug. This worked out great for us, and still to this day, we don't use **F5** in VS to launch debug sessions. We just attach to process.

Anyways, we had these scripts, but they didn't really handle endpoints, certificates and stuff, so for the longest time, I was manually fixing our deploy packages when deploying to our staging/prod clusters. I started doing partial upgrade packages by hand as well. Since we all know that humans do stupid things and have a tendancy to forget things, so did I, and often I had to start over with packaging since I'd forgotten something. This sucked, and deploying code was no fun, only irritation.

So, we decided to fix it once and for all (for us at least).

# Requirements

We created a list of requirements that the tooling of our choice had to have:

* It should know what have changed between versions
* It should know how to package for partial upgrades
* It should just fix all the version numbers in our manifests
* It should be able to configure our endpoints and certificates in the manifests
* It should work running locally to package for one-node, but also work when running on a build agent packaging for staging/prod clusters
* It should be able to package multiple application types in one go as one _thing_
* The cluster it is packaging for is the master

Also, bonus wishes:

* Making tooling can be fun (and I don't like scripting comples things in powershell)
* I'd like to make a .NET Core CLI app that is actually useful
* Alternatives in tooling also helps drive the community forward

So, we decided to make a command line packager in .NET Core.

# The packager

The [Service Fabric Packager](https://github.com/proactima/ServiceFabricPackager) is a .NET Core CLI program creates an application package for each of your application, picking only things that have changed (and by changed, I also mean that if you change only a ServiceManifest for a service, only that will be packaged).

## How it works

Let's talk a bit about how the packager works.

First of all, it requiers an external storage to keep both version info files, and also to keep the config file for the packager in. Currently, both a local folder and Azure Storage Blob is supported. We use local folder when running it locally. The config and version files will be described later.

### Assemblies and packages

So when you run it, it'll scan the input folder for ServiceFabric applications, parse those to figure out which services belong to which application. The packager will not build for you, only package.

After all the paths have been determined, the packager connects to the target cluster to get the current global version. The global version is just the highest version deployed to the cluster. If it's unable to parse the versions deployed, it'll fall back to creating a full package, as opposed to a partial package if it's able to determine the current version.

So, how do we detemine what to package?
Well, first of all, you need to have deterministic compilation turned on in all the projects. Currently, it doesn't seem like it's possible to pass that flag down through dotnet, so for now, any ASP.Net Core projects will be packaged if the local assemblies have been rebuilt since last package. Not much to do with that.

For each service it finds, the packager will compute the hash for each of the components in that service. By that I mean it'll compute the hash for each of the subpackages defined in the service manifest and the service manifest itself.
It'll also compute the hash of each of the application manifests. Then it'll compare that with the version file it has loaded for the previous version and determine what has changed.

Then all the changed things are copied to the correct folders.

### Manifests

For the manifests (application/service), it'll create dummy manifests and apply changes from the config file to those and hash them. When it actually package, it'll take the local manifest, remove default services and application parameters (cause we don't use them), apply the same changes that it did to the dummy manifest and package that.

### Config file

I've mentioned a config file several times. The config files contains information about how to connect to a cluster (endpoint, port, pfx file and pfx password). In addition, it contains endpoint configuration (including certificate thumbprints) for services that needs it.

### Version Number

For version number we've decided on a simple variant. The version number is just an incrementing int and then a string of your choosing. We use the git commit hash. That way, we can look at a cluster and immediatley know what commit we need to branch off from if we want to make a hotfix. Example version number: "13-64625c5a130d705995b7bf3b1d21bc297bfdaa0f".

### Deploying the packages

We've just made it easy for ourself, and checked in the complied packager into our solution so that it's always available. We then just have a simple script that launches the packager with the correct settings, and then afterwards copies and register the packages to the local cluster and starts an upgrade.
For our clusters running in Azure, we've created some custom deploy tasks based on the default SF deploy tasks. Mainly we've modified them to be able to deploy multiple packages. For staging, when we do a build, it'll auto start a release that uploads, registers and starts an upgrade.
For prod, it's the same, but it doesn't auto start the upgrade.

# Conclusion

For us this is the perfect packaging solution. We've been using it for about a month both for local onebox, and also for our staging/prod clusters. It has probably created up towards 200 packages already. I don't have to think about what should be deployed or not. It's handled for me. Also, whenever we refresh our SSL certs (about every 10 weeks), after deploying the certs to our VMSS, i only need to change the config file and start a build on the current commit. Then only the manifests will be packaged, and I can start the upgrade to make sure our services are using the new certs. 

## Next steps

So, the code is a bit messy now, and it has some missing features that I plan to fix shortly. Listed in no particular order:

* Make sure all paths are cross-platform compatible
* Create a VSTS task for the packager, with support for using the VSO connection manager for managing the cluster connection info
* Move include/exclude to the config (they are hardcoded now)
* Refactor and clean up some of the internals
* Remove the dummy manifests that are created and use the manifest we plan to deploy as the input for hashing
* Add an example config file since that's missing right now...

Quick note about the include/exclude:

Includes are for taking an external file and copying it in during the packaging process. We use it to copy in a config file with some secrets that we need to bootstrap our config service.

The excludes are used for excluding items from the hashing process. We use it to exclude resource files, since the compilation of those isn't deterministic, and thus without excluding them, every time we build the solution it would look like something has changed because of those files. The downside to this is that if we _only_ change the resource files, the packager won't pick it up and thus won't do any packaging.

Next time, I plan on talking about creating custom VSTS tasks :)