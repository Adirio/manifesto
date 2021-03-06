# manifesto
Manifesto lets users store and query metadata for Docker images. This metadata can be information that you want to store about an image *post-build* - where labels are not sufficient. 

[![Build Status](https://travis-ci.org/aquasecurity/manifesto.svg?branch=master)](https://travis-ci.org/aquasecurity/manifesto)

## Use cases
* **Managing QA approval status** After an image has been built, it needs to go through various testing and approval processes before your organization is ready to use it in production. Keep track of approval status, and who has given sign-off by storing it alongside the image itself. 
* **Storing security profiles for an image** Manifesto makes it easy to associate a Seccomp or AppArmor profile with an image, so that you can automatically retrieve the correct profile at the point you want to run a container. 
* **Storing vulnerability scan reports** Images should be scanned regularly for vulnerabilities as new ones may be found in existing code. Manifesto enables storing the latest scan report for an image without modifying the image itself. 
* **Support contacts** Store the phone number or Slack channel to contact in the event this container image starts causing problems in your live deployment. Update these details without needing to update the image. 
* **Tracking active images** With CI/CD it's easy to end up with hundreds of thousands of container images in your registry. Use manifesto to store whether an image is actively being used in production - or to indicate the images that can safely be pruned.

The intention is that each piece of metadata could be signed using Notary. This means you can reliably get back the most recent version of that piece of metadata and know that it was put in place by someone with the authority to do so. 

At the moment this is a Proof of Concept - feedback and ideas very welcome. 

## Demo 

[![asciicast](https://asciinema.org/a/128283.png)](https://asciinema.org/a/128283)

## Installation

We automatically build binary executables for Mac, Linux and Windows, or you can rebuild from source. Whichever approach you take, **you'll also need to set up credentials** - see below. 

### Installing binaries
Download the latest binary for your platform from the [releases tab](https://github.com/aquasecurity/manifesto/releases) and unzip it. 

You may find it easiest to move the binary into your path once you have downloaded it. For example 

```
$ wget https://github.com/aquasecurity/manifesto/releases/download/<release tag>/manifesto-darwin-amd64.tar.gz
$ tar xf manifesto-darwin-amd64.tar.gz
$ cp bin/darwin/amd64/manifesto /us/local/bin
```

### Building from source
* Clone this repo (or `go get github.com/aquasecurity/manifesto`)
* Go to the directory and `go build .`

### Set up credentials
In this release you will need to be logged in to the Docker Registry - do this with [`docker login`](https://docs.docker.com/engine/reference/commandline/login/). This means that manifesto can execute docker commands directly. 

In addition, since we are now using the Registry API directly to store and retrieve metadata in blobs, you need to pass in your username and password to manifesto itself. You can do this with the command line, environment variables REGISTRY\_USERNAME and REGISTRY\_PASSWORD, or by responding to prompts. 

## Usage

```
$ ./manifesto --help
Store, retrieve and list pieces of metadata alongside your container images in the registry.
Metadata is associated with specific images (by hash).

Usage:
  manifesto [command]

Available Commands:
  get         Show metadata for the container image
  help        Help about any command
  list        List currently stored metadata for the container image
  put         Put metadata for the container image

Flags:
  -h, --help              help for manifesto
  -p, --password string   Registry password (can also be passed in with the env var REGISTRY_PASSWORD)
  -u, --username string   Registry username (can also be passed in with the env var REGISTRY_USERNAME)
  -v, --verbose           Send debug output to stderr

Use "manifesto [command] --help" for more information about a command.
```

By default (like Docker images) manifesto assumes the 'latest' tag if a tag is not given. 

### Example

```
$ ./manifesto put myorg/imagetest something ~/temp.json
Storing metadata 'something' for 'myorg/imagetest:latest'
Metadata 'something' for 'myorg/imagetest:latest' stored at sha256:7be34480285971f16eed284b13fa7d417649f18c7d1af9b2de6970ce99e3cbbd
Updating manifesto for myorg/imagetest
Replacing 'something' metadata in manifesto for 'myorg/imagetest:latest'

$ ./manifesto list myorg/imagetest
Metadata types stored for image 'myorg/imagetest:latest':
    something

$ ./manifesto get myorg/imagetest something
{
  "key" : "value",
  "createdBy" : "liz",
  "number" : 56
}
```

# Proof of concept status

In this proof of concept:  

* we store arbitrary metadata within the Docker registry 
* we store a "manifesto" for the repository with the fixed tag "_manifesto"
* the manifesto is a json file with references to all the metadata stored for this repository

![Manifesto is stored as an image, which references the data blobs for individual pieces of metadata](https://docs.google.com/drawings/d/1IGm4WnhL3J0hp2hdELrevyn3SMbgs0tlKNHjYIQHqtM/pub?w=960&h=720)

### Note - use of image tags in this prototype
In v 0.0.1 the metadata was stored as separate images, tagged as \_manifesto\_*metadata-type*. This allowed us to build the initial prototype very easily, but it meant there were additional repository tags for each piece of metadata, which seems undesirable.  

In this version we use the Registry API to store metadata directly in blobs, as shown in the diagram above. This will mean there will just be one additional tag in the repository, \_manifesto. 

## Can I store metadata for any image?

As the metadata is stored in the same repository as the image itself, you can only store metadata for images you own. You could of course save a copy of a third-party image in your own repository, and store metadata alongside it. 

## How is this better than labels pointing to some arbitrary location where information is stored? 

Putting metadata into the registry itself allows us to leverage both existing technology and existing infrastructure. An organisation that stores images in its own on-premise registry can simply store metadata in it, and if they are using Notary again they can simply re-use the same deployment. The signing of metadata uses exactly the same process as the signing of images, so any existing audits and controls can be re-used. 

## Manifesto data

The manifesto associated with a repository is built using a Dockerfile like this: 

```
FROM scratch
ADD <filename> /data
```

Where <filename> contains all the metadata references for this repository. An example might look like this:

```
{
	"images": [{
		"image_digest": "70d2f067eb94ec8ab0530068a414d8dbe8c203244ae5d5ad4ba6eb1babd1c1c1",
		"manifesto": [{
				"type": "seccomp",
				"digest": "sha256:a2fe22a6d44aa86432adad99481c3ad526ba35af2223df126620d20e38c70fac"
			},
			{
				"type": "approvals",
				"digest": "sha256:6ced8eb4e6a61639601e7073963ec04a80f70a11442157e1dd825f042879a6da"
			},
			{
				"type": "contact",
				"digest": "sha256:e896a0012a3450d9cef7e040eea8bed3fe06188957439fea501a65b62c65b4f1"
			}
		]
	}, 
    {
		"image_digest": "51d2f067eb94ec8ab0531987a414d8dbe8c203244ae5d5ad4ba6eb1babd1d54a",
		"manifesto": [{
				"type": "seccomp",
				"digest": "sha256:b2f72296d04ea36435adae99481c3ad526ba35af2223df126620d20e38c9763c"
			},
			{
				"type": "documentation",
				"digest": "sha256:9ce18eb4e6a66639601e7073963ec04aa0f70a11442157e1d9825f042879abb1"
			}
		]
    }]
}
```

The *type* of each piece of metadata is simply an arbitrary string to identify that type of data. One possibility is to use standardized names (possibly as defined in the OCI image spec or similar) for the type to indicate that the associated data blob contains JSON in a standardized format (such as the vulnerability scanning report format).

## Wait - in Docker, it's *tags* that get signed in Notary. Why is metadata associated with image digests?
When you pull a Docker image tagged, say, 1.4 you'll get whatever the latest signed version tagged 1.4 is. The tag can move between different images (i.e. with different digests) as updates are made, for example to update from 1.4.3 to 1.4.4. 

For many types of image metadata (e.g. approval status, vulnerability scan report), a piece of metadata must be associated with a particular build i.e. identified by the digest. 

When metadata is added, if signing is enabled the metadata blob gets signed in Notary, as does the manifesto with the references to the metadata blobs. 

# To Do's 

* Add data signing capabilities and verification with Notary. 
* Code currently execs out to the docker client executable - would be better to use the go client and call the API directly. 
* You need to enter your Docker Hub username and password to put metadata (or pass them in as env vars or parameters). Other commands currently exec out to docker, which means an existing `docker login` is sufficient for the credentials.  It would be better to be consistent, and even better if there were a way of using the credentials from that `docker login`

