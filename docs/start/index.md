---
layout: default
title: Quick Start
nav_order: 3
has_children: false
---

# Quick Start
{: .no_toc .header }

----
Starting using Mica is simple. Below steps will help you to get a community version running up in just one minute.

*Note: The community version is free for personal use only, if you want to use in your business please reach us by emailing to [xx@xxx.us](mailto:xx@xxx.us)*

## Local Deployment

To deploy locally, ensure your environment has Docker 20+ installed. For stable performance, it's recommended to allocate 2 CPU cores and 8GB of free memory. Then, start with the following command:

```bash
docker run --hostname Mica -d -v /root/data:/data -p 8080:80 --name Mica Mica/pack:latest
```
- `--hostname Mica`: Avoid the hostname of the container changed each time it restarts.
- `-v /root/data:/data`: Specifies the persistent data storage directory. You can modify it as needed. Failure to configure this will result in data loss after container restart.
- `-p 8080:80`: Sets the external port. Default external port is 80, if you want to set the port to other like 8080, you can add this param.
- `--name Mica`: Sets the container name for easy use in docker command.
- Two image options are available: `Mica/pack:latest` and `Mica/aio:latest`. The former uses collectors directly from GitHub, while the latter integrates the latest collectors into the image, albeit with a larger size.

After the container is started, you can check the logs to see if the system is running:
```bash
docker logs Mica
```

When you see the following log, it means the system is running:
```bash
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
 System is ready! You can access the service as follows:
     URL: http://172.17.0.36
     User: xxx@xxxx
     Password: xxx

 Any question or suggestion, please reach out to xxx@xxx.us! Enjoy!!!
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

If you want to upgrade the system, you can use the following command:
```
#!/bin/bash

echo "Upgrade ..."
docker pull Mica/pack:latest
docker stop Mica
docker rm Mica
docker run --hostname Mica -d -v /root/data:/data -p 8080:80 --name Mica Mica/pack:latest
echo "Upgrade Done!"
exit
```

## Cloud Deployment

For cloud deployment using AWS ECS as an example, follow these steps:

1. Create a Task Definition in ECS and configure the Container:
    - Set the Image URI to `Mica/pack:latest`.
    - Map at least port 80 to facilitate UI access and collector data reporting. For example, set Host port to 8080.

![img.png](img.png)

2. Configure Volumes to map `/data` to an EBS volume to ensure data persistence. If you don't add the volume, the data will be lost after restarting the container.

![img_1.png](img_1.png)

3. Launch a Service in a Cluster using the Task Definition.

## Getting Started

Access the system via `http://<your_ip>:8080` in your browser. The default credentials are `xxx@xxx.local/admin`. Upon initial login, remember to change the password.

![img_3.png](img_3.png)

Upon login, you'll enter the Wizard page by default. Follow the prompts to configure:

1. **XXX**: Configure the system xxx
   1. **xxx**: xxx.
   2. **xXX**: xxx.

![img_2.png](img_2.png)

2. **XX**: XXx.
   1. **XXX**: xxx.
   2. **XXX**: xxx.

![img_5.png](img_5.png)

3. **XX**: XXx.


Upon completion, click "xx" to exit the Wizard. You can access Mica details and manage services from the left sidebar's Services page.



