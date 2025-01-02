# Docs
Mica Documentations. We are using just-the-docs, please visit [Just the Docs (just-the-docs.com)](https://just-the-docs.com/) for more.


## How to view locally without building

You can run with a exist image

```
docker-compose -f docker-compose-from-image.yml up
```

## How to view locally

You can build a docker image and start the document server locally

1. Build image, under root directory, execute *docker-compose build*, this may take a while
2. Start up service by *docker-compose up*
3. Once started, you can access the documents locally following the direction shown in the console
   1. If you made any changes under docs folder, it will be dynamically reflect in the testing server

Note: by default, the started server will service by docker IP and all links will be given using that IP, in case you want to use something like a host name or DNS, you can try start the server by appending HOST=\<hostname or DNS>, e.g. *HOST=myserver docker-compose up*


## How to publish

To release, just **merge you changes into master** branch and the changes will be deployed by GitHub to [https://Mica.github.io](https://Mica.github.io)