
keepalived'nin sanal olarak atayacagi ip => 192.168.0.70

interface ens18 senin makinendeki ethernet neyse o (ip a s dediginde gordugun ethernet karti)


paketleri yukle
``` bash
apt update && apt install -y keepalived haproxy
```


keepalived healthcheck sh dosyasini ekle
``` bash
cat >> /etc/keepalived/check_apiserver.sh <<EOF
#!/bin/sh

errorExit() {
  echo "*** $@" 1>&2
  exit 1
}

curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
if ip addr | grep -q 172.16.16.100; then
  curl --silent --max-time 2 --insecure https://192.168.0.70:6443/ -o /dev/null || errorExit "Error GET https://192.168.0.70:6443/"
fi
EOF
```


healthcheck execute hakki ver
``` bash
chmod +x /etc/keepalived/check_apiserver.sh
``` 


keepalived conf dosyasini ekle
``` bash
cat >> /etc/keepalived/keepalived.conf <<EOF
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  timeout 10
  fall 5
  rise 2
  weight -2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens18
    virtual_router_id 1
    priority 100
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass mysecret
    }
    virtual_ipaddress {
        192.168.0.70
    }
    track_script {
        check_apiserver
    }
}
EOF
``` 

keepalived yi enable et
``` bash
systemctl enable --now keepalived
``` 

harproxy conf dosyasini set et
``` bash
cat >> /etc/haproxy/haproxy.cfg <<EOF

frontend kubernetes-frontend
  bind *:6443
  mode tcp
  option tcplog
  default_backend kubernetes-backend

backend kubernetes-backend
  option httpchk GET /healthz
  http-check expect status 200
  mode tcp
  option ssl-hello-chk
  balance roundrobin
    server km-1 192.168.0.61:6443 check fall 3 rise 2

EOF
```


servisleri restart et
``` bash
systemctl enable haproxy && systemctl restart haproxy
```
