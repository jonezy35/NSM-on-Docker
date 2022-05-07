I recently set up Elasticsearch, Kibana, and Suricata in my homelab in docker containers utilizing filebeat to ship the logs from Suricata's eve.json to Elasticsearch. This is the process.



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

cd <directory you want to run suricata from>

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
