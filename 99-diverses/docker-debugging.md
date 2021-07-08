# Liste der Container und ihrer Adressen

```bash
# l=$(docker ps --format '{{ .Names  }}' )
# for i in $l; do echo -n "$i "; docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $i; done | column -t
mqttbridge 172.19.0.5
z2m 172.19.0.4
grafana 172.19.0.6
influxdb 172.19.0.3
mosquitto 172.19.0.2
unifi 172.20.0.2
jitsi-jicofo 172.3.0.2
jitsi-jvb 172.3.0.3
jitsi-prosody 172.3.0.5
jitsi-web 172.3.0.4
gitlab 172.18.0.3
gitlab-runner 172.18.0.2
```
Hint: use ```sort -V -k 2 | column -t ``` to sort by IP-Address
