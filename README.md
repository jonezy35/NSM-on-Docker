I recently set up Elasticsearch, Kibana, and Suricata in my homelab in docker containers utilizing filebeat to ship the logs from Suricata's eve.json to Elasticsearch. This was the process.

# Why

First off, why did I do this?

This was a project I wanted to do as a precursor to bigger things. The benefits of running containers are great for development environments and at scale. My next project is to tear this down and set it up on a larger scale using Kubernetes. My long term goal is to get my [Certified Kubernetes Administrator](https://www.cncf.io/certification/cka/) cert.

This was a way to validate what was in my head and get more comfortable with Docker. I'll let it run for awhile while I work through some pluralsight kubernetes courses and then I'll tear it down and do it again in Kubernetes.

### The Benefits of Containerization

  1. <ins>Portability</ins>: You can run a Docker image almost anywhere as long as Docker is installed on the host.
  2. <ins>Better resource utilization</ins>: When running VM's, each VM needs a host OS which utilizes resources. Again, on a small scale this doesn't matter, but in a large scale environment this is a lot of resources lost. With containers, you don't need to waste resources on multiple OS's. [This site has a good image to explain what I'm talking about.](https://www.google.com/url?sa=i&url=https%3A%2F%2Fwww.sdxcentral.com%2Fcloud%2Fcontainers%2Fdefinitions%2Fcontainers-vs-vms%2F&psig=AOvVaw3ERCyu7z_SDJGz7XCEmSbo&ust=1652019873455000&source=images&cd=vfe&ved=0CAwQjRxqFwoTCKCQ2ODLzfcCFQAAAAAdAAAAABAJ)
  3. <ins>Microservices</ins>: Containers are great for developing [Microservices](https://www.redhat.com/en/topics/microservices/what-are-microservices).
  ###### There are a lot more benefits to Containerization than I don't want to take the time go into here. If you're interested here are a few links to go deeper: [Here](https://www.ibm.com/cloud/blog/the-benefits-of-containerization-and-what-it-means-for-you) and [Here](https://www.veritis.com/blog/7-key-containerization-benefits-for-your-it-business/)

Containers really shine in the cloud. All cloud platforms support [container orchestration](https://www.vmware.com/topics/glossary/content/container-orchestration.html#:~:text=Container%20orchestration%20is%20the%20automation,networking%2C%20load%20balancing%20and%20more.) with [Kubernetes](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/) or a proprietary orchestration method (AWS has [ECS](https://aws.amazon.com/ecs/) and [EKS](https://aws.amazon.com/eks/) Azure and GCP have their own version as well).



## Downloading Docker

To start, you need to make sure you have docker.

[Here](https://docs.docker.com/engine/install/) is the link to download Docker

## Housekeeping
I did this on an ubuntu server that I'm running on ESXI. Keep in mind the OS you're running on when looking at my commands.

## Capture Interface
First you need to set up a capture interface (ex. a SPAN port off of a managed switch). You need two interfaces on your box for this: your capture interface and a management interface. On ESXI when you create the capture interface you need to create a new virtual switch and enable promiscous mode. Then in the VM you need to run an `ip link set <capture interface> promisc on`

Now check to see if you're collecting traffic on the interface with a `tcpdump -i <capture interface>` You should see traffic after a few seconds. `CTRL+C` to exit the tcpdump.

## Create a Directory

In my home directory I have a Docker directory for my Docker projects.. Inside that Docker directory I created an ELK directory and inside there I created a suricata directory. Once you have that done and you've installed docker you should be ready to start.



## VM Memory Count
First, you have to `sysctl -w vm.max_map_count=262144` If you don't do this your containers will crash.

## Elasticsearch & Kibana
I installed elasticsearch and kibana in docker containers as shown [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html).

The steps from the link above (assuming you have docker downloaded already) are:
```bash

docker pull docker.elastic.co/elasticsearch/elasticsearch:8.2.0
docker pull docker.elastic.co/kibana/kibana:8.2.0


docker network create elastic
docker run --name es01 --net elastic -p 9200:9200 -p 9300:9300 -d docker.elastic.co/elasticsearch/elasticsearch:8.2.0
docker cp es01:/usr/share/elasticsearch/config/certs/http_ca.crt .
curl --cacert http_ca.crt -u elastic https://localhost:9200

docker run --name kib-01 --net elastic -p 5601:5601 -d docker.elastic.co/kibana/kibana:8.2.0
```

Instead of running the containers with the `-it` flag, which runs them in the terminal, you should instead run them with the `-d` flag which will run them in the background.

Then, to get the enrollment token for Kibana you can run a `docker logs es01` If that doesn't work you can run the following commands to get the enrollment token and the elastic password.

`docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana`


`docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic`


## Suricata
Once Elasticsearch and Kibana are up and running, you can install suricata. I did that [here](https://github.com/jasonish/docker-suricata).
The steps from the link above are:

```bash

cd <suricata directory>

docker pull jasonish/suricata:latest

mkdir logs

docker run --rm -it --net=host \
    --cap-add=net_admin --cap-add=net_raw --cap-add=sys_nice \
    -v $(pwd)/logs:/var/log/suricata \ #This will save the eve.json logs to your local box in your current directory/logs. This ensures that your logs persist outside of the container to survive container restarts/shutdowns/etc.
	jasonish/suricata:latest -i <capture interface>

```

One note: you want to use the `-v` option as it's shown because that will save the eve.json logs to your local box (in my case my ubuntu server). This allows your logs to persist through container failurs/restarts/shutdowns/etc.

## Filebeat as a service
Now for filebeat. The documentation on installing filebeat is [here](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-installation-configuration.html). You can run filebeat in a container but since the logs are saved to the localhost I decided to run filebeat directly on the ubuntu server for now.

The filebeat.yml should be in `/etc/filebeat`

 When configuring filebeat, make sure you edit the filebeat.yml and pass it a username and password for elastic as well as make sure you specify the path as https and enable ssl. An example filebeat.yml:
 ```bash
 hosts: ["https://localhost:9200"]
  username: "name"
  password: "password"
  ssl:
    enabled: true
    ca_trusted_fingerprint: "b9a10bbe64ee9826abeda6546fc988c8bf798b41957c33d05db736716513dc9c"
    #You generate this fingerprint down below. This sample fingerprint is what elastic provides as an example. You shouldn't post you fingerprint anywhere as a best practice.
 ```

To generate the fingerprint needed for ssl. Refer to [this documentation](https://www.elastic.co/guide/en/elasticsearch/reference/8.0/configuring-stack-security.html#_connect_clients_to_elasticsearch_5) specifically: `openssl x509 -fingerprint -sha256 -in config/certs/http_ca.crt` You should be able to run this command in the directory you ran the `docker cp es01:/usr/share/elasticsearch/config/certs/http_ca.crt .` command up above.


You then need to cd into modules.d and edit the suricata.yml to enable it (set it to true) and give it the path of your eve.json (the path on your local host).

Sample suricata.yml:

```bash
eve:
    enabled: true
    var.paths: ["/home/jonezy/Docker/ELK/suricata/logs/eve.json"]
    # Replace var.paths with the path for your eve.json

```

You now have to run `filebeat setup -e` to set up the index template and some dashboards.

Once thats done you should be able to start the service. On ubuntu the command is `sudo service filebeat start`

If you cat eve.json and there's logs in there you should now be seeing those logs in Kibana.
