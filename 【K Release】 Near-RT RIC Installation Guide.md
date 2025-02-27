# 【K Release】 Near-RT RIC Installation Guide

Branch: K Release
Status: 進行中

<aside>
💡

Hardware

- RAM：8G RAM
- CPU：6 core
- Disk：100G Storage

Installation Environment：

- VM ware
- Ubuntu 20.04

Reference:

- [【G Release】 Installation Guide of Near-RT RIC Platform](https://hackmd.io/@2xIzdkQiS9K3Pfrv6tVEtA/G-release_Near-RT-RIC_Install#%E3%80%90G-Release%E3%80%91-Installation-Guide-of-Near-RT-RIC-Platform)
- https://flex-solution.com/page/blog/install-k8s-lower-than-1_24
- [【G Release】 Improve Near-RT RIC Platform](https://hackmd.io/@2xIzdkQiS9K3Pfrv6tVEtA/Bkzlwdzci)
</aside>

## 1. Install the Docker, Kubernetes and Helm 3

### **1.1 Open terminate (Ctrl+Alt+T) and become root user：**

```bash
sudo -i
```

### **1.2 Install the Dependent Tools**

```bash
apt-get update
apt-get install -y git vim curl net-tools openssh-server python3-pip nfs-common
```

### **1.3 Download the source code of RIC Platform**

```go
git clone https://gerrit.o-ran-sc.org/r/ric-plt/ric-dep -b master
```

https://gerrit.o-ran-sc.org/r/gitweb?p=ric-plt/ric-dep.git;a=commit;h=33b5940a15ea852d47f73521295eef53d295b7fd

- It doesn’t add k-release tag, so we use the master branch

### 1.4 Install docker, kubernetes and helm

```bash
cd ric-dep/bin
./install_k8s_and_helm.sh
```

| Kubernetes | CNI | Helm | Docker |
| --- | --- | --- | --- |
| v1.28.15 | 0.7.5 | 3.14.4 | 20.10.21-0ubuntu1~20.04. |
- Check the status of pods

```bash
kubectl get pods -A
```

![image.png](https://github.com/bmw-ece-ntust/Near-RT-RIC/blob/master/image/image.png)

## 2. Install the Near-RT RIC Platform

### 2.1 Add the ric-common templates

```bash
./install_common_templates_to_helm.sh
./setup-ric-common-template
```

![image.png](https://github.com/bmw-ece-ntust/Near-RT-RIC/blob/master/image/image%201.png)

NOTE: How many '`servecm not yet running. sleeping for 2 seconds`' it is depends on your download speed. Because it will wait to download [download chartmuseum](https://s3.amazonaws.com/chartmuseum/release/latest/bin/linux/amd64/chartmuseum).

### 2.2 Edit Deployment Configuration

- **2.2.1 Modify the code of xApp Manager**
    - Add loglevel “DEBUG”：
    
    ```bash
    vim ~/ric-dep/helm/appmgr/resources/appmgr.yaml
    ```
    
    ```bash
    "xapp":
      #Namespace to install xAPPs
      "namespace": __XAPP_NAMESPACE__
      "tarDir": "/tmp"
      "schema": "descriptors/schema.json"
      "config": "config/config-file.json"
      "tmpConfig": "/tmp/config-file.json"
    "loglevel" :  4
    ```
    
- **2.2.2 Modify the code of A1 Mediator**
    - Modify the Log-Level
    
    ```bash
    vim ~/ric-dep/helm/a1mediator/templates/config.yaml
    ```
    
    ```bash
    ...
        rte|20011|{{ include "common.servicename.a1mediator.rmr" . }}.{{ include "common.namespace.platform" . }}:{{ include "common.serviceport.a1mediator.rmr.data" . }}
        rte|20012|{{ include "common.servicename.a1mediator.rmr" . }}.{{ include "common.namespace.platform" . }}:{{ include "common.serviceport.a1mediator.rmr.data" . }}
        newrt|end
      loglevel.txt: |
        log-level: {{ .Values.a1mediator.loglevel }}
    ```
    
    - Change loglevel from INFO to DEBUG & change Component Connection
    
    ```bash
    vim ~/ric-dep/helm/a1mediator/values.yaml
    ```
    
    ```bash
    # these are ENV variables that A1 takes; see docs
      rmr_timeout_config:
        a1_rcv_retry_times: 20
        ins_del_no_resp_ttl: 5
        ins_del_resp_ttl: 10
      loglevel: "DEBUG"
      a1ei:
        ecs_ip_port: "http://<ecs_host>:<ecs_port>"
    ```
    
    - Add ENV for A1EI
    
    ```bash
    vim ~/ric-dep/helm/a1mediator/templates/env.yaml
    ```
    
    ```bash
    data:
      ...
      ...
      ECS_SERVICE_HOST: {{ .Values.a1mediator.a1ei.ecs_ip_port }}
    ```
    
- **2.2.3 Modify the code of E2 Termination**
    - Change loglevel from ERR to 4 (Debug)
    
    ```bash
    vim ~/ric-dep/helm/e2term/values.yaml
    ```
    
    ```bash
    health:
        liveness:
          command: "ip=`hostname -i`;export RMR_SRC_ID=$ip;/opt/e2/rmr_probe -h $ip"
          initialDelaySeconds: 10
          periodSeconds: 10
          enabled: true
    
        readiness:
          command: "ip=`hostname -i`;export RMR_SRC_ID=$ip;/opt/e2/rmr_probe -h $ip"
          initialDelaySeconds: 120
          periodSeconds: 60
          enabled: true
    
    loglevel: 4
    
    common_env_variables:
      ConfigMapName: "/etc/config/log-level"
      ServiceName: "RIC_E2_TERM"
    ```
    
    - Add log-level section
    
    ```bash
    data:
      log-level: |
        {{- if hasKey .Values "loglevel" }}
        log-level: {{ .Values.loglevel }}
        {{- else }}
        log-level: 1
        {{- end }}
      rmr_verbose: |
        0
      ...
      ...
    ```
    
- **2.2.4 Modify the code of Subscription Manager**
    - Modify the Log-Level
    
    ```bash
    vim ~/ric-dep/helm/submgr/templates/configmap.yaml
    ```
    
    ```bash
    data:
      # FQDN and port info of rtmgr
      submgrcfg: |
        "local":
          "host": ":8080"
        "logger":
          "level": 4
        "rmr":
          "protPort" : "tcp:4560"
          "maxSize": 8192
          "numWorkers": 1
    ```
    
    - Add new port for subscription in service file
    
    ```bash
    vim ~/ric-dep/helm/submgr/templates/service-http.yaml
    ```
    
    ```bash
    spec:
      selector:
        app: {{ include "common.namespace.platform" . }}-{{ include "common.name.submgr" . }}
        release: {{ .Release.Name }}
      clusterIP: None
      ports:
      - name: http
        port: {{ include "common.serviceport.submgr.http" . }}
        protocol: TCP
        targetPort: http
      - name: subscription
        port: 8088
        protocol: TCP
        targetPort: 8088
    ```
    
    - Add new port for subscription in deployment file
    
    ```bash
    vim ~/ric-dep/helm/submgr/templates/deployment.yaml
    ```
    
    ```bash
         containers:
              ...
              envFrom:
                - configMapRef:
                    name: {{ include "common.configmapname.submgr" . }}-env
                - configMapRef:
                    name: {{ include "common.configmapname.dbaas" . }}-appconfig
              ports:
                - name: subscription
                  containerPort: 8088
                  protocol: TCP
    	    ...
    ```
    
- **2.2.5 Modify the code of Routing Manager**
    - Add A1EI Msgtype and A1EI Routes
    
    ```bash
    vim ~/ric-dep/helm/rtmgr/templates/config.yaml
    ```
    
    ```bash
     "messagetypes": [
              ...
              "A1_EI_QUERY_ALL=20013",
              "A1_EI_QUERY_ALL_RESP=20014",
              "A1_EI_CREATE_JOB=20015",
              "A1_EI_CREATE_JOB_RESP=20016",
              "A1_EI_DATA_DELIVERY=20017",
              ]
              ...
          "PlatformRoutes": [
             ...
             { 'messagetype': 'A1_EI_QUERY_ALL','senderendpoint': '', 'subscriptionid': -1, 'endpoint': 'A1MEDIATOR', 'meid': ''},
             { 'messagetype': 'A1_EI_CREATE_JOB','senderendpoint': '', 'subscriptionid': -1, 'endpoint': 'A1MEDIATOR', 'meid': ''},
              ]
    ```
    
- **2.2.6 Modify the code of Alarm Manager**
    - Change controls.promAlertManager.address
    
    ```bash
    vim ~/ric-dep/helm/alarmmanager/templates/configmap.yaml
    ```
    
    ```bash
         	controls": {
            "promAlertManager": {
              "address": "r4-infrastructure-prometheus-alertmanager:80",
              "baseUrl": "api/v2",
              "schemes": "http",
              "alertInterval": 30000
            },
    ```
    
    - Add livenessProbe and readinessProbe
    
    ```bash
    vim ~/ric-dep/helm/alarmmanager/templates/deployment.yaml
    ```
    
    ```bash
        spec:
          hostname: {{ include "common.name.alarmmanager" . }}
          imagePullSecrets:
            - name: {{ include "common.dockerregistry.credential" $imagectx }}
          serviceAccountName: {{ include "common.serviceaccountname.alarmmanager" . }}
          containers:
            - name: {{ include "common.containername.alarmmanager" . }}
              image: {{ include "common.dockerregistry.url" $imagectx }}/{{ .Values.alarmmanager.image.name }}:{{ $imagetag }}
              imagePullPolicy: {{ include "common.dockerregistry.pullpolicy" $pullpolicyctx }}
              readinessProbe:
                failureThreshold: 3
                httpGet:
                  path: ric/v1/health/ready
                  port: 8080
                  scheme: HTTP
                initialDelaySeconds: 5
                periodSeconds: 15
              livenessProbe:
                failureThreshold: 3
                httpGet:
                  path: ric/v1/health/alive
                  port: 8080
                  scheme: HTTP
                initialDelaySeconds: 5
                periodSeconds: 15
              ...
    
    ```
    
- **2.2.7 Modify the code of O1 Mediator**
    - Add livenessProbe and readinessProbe
    
    ```bash
    vim ~/ric-dep/helm/o1mediator/templates/deployment.yaml
    ```
    
    ```bash
        spec:
          hostname: {{ include "common.name.o1mediator" . }}
          imagePullSecrets:
            - name: {{ include "common.dockerregistry.credential" $imagectx }}
          serviceAccountName: {{ include "common.serviceaccountname.o1mediator" . }}
          containers:
            - name: {{ include "common.containername.o1mediator" . }}
              image: {{ include "common.dockerregistry.url" $imagectx }}/{{ .Values.o1mediator.image.name }}:{{ .Values.o1mediator.image.tag }}
              imagePullPolicy: {{ include "common.dockerregistry.pullpolicy" $pullpolicyctx }}
              livenessProbe:
                failureThreshold: 3
                httpGet:
                  path: ric/v1/health/alive
                  port: 8080
                  scheme: HTTP
                initialDelaySeconds: 5
                periodSeconds: 15
                successThreshold: 1
                timeoutSeconds: 1
              readinessProbe:
                failureThreshold: 3
                httpGet:
                  path: ric/v1/health/ready
                  port: 8080
                  scheme: HTTP
                initialDelaySeconds: 5
                periodSeconds: 15
                successThreshold: 1
                timeoutSeconds: 1
              ...
    ```
    

### **2.3 Install nfs for InfluxDB**

```jsx
kubectl create ns ricinfra
helm repo add stable https://charts.helm.sh/stable
helm install nfs-release-1 stable/nfs-server-provisioner --namespace ricinfra 
kubectl patch storageclass nfs -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
sudo apt install nfs-common
```

### **2.4 Execute the Installation Script of Near-RT RIC**

- **Find your IP of VM**

```bash
ifconfig
```

![image.png](https://github.com/bmw-ece-ntust/Near-RT-RIC/blob/master/image/image%202.png)

- **Modify the IP of RIC and AUX**

```jsx
vim ~/ric-dep/RECIPE_EXAMPLE/example_recipe_oran_k_release.yaml
```

![image.png](https://github.com/bmw-ece-ntust/Near-RT-RIC/blob/master/image/image%203.png)

- Deploy the RIC Platform

```bash
./install -f ../RECIPE_EXAMPLE/example_recipe_oran_k_release.yaml -c "jaegeradapter influxdb"
```

### **2.5 Check the Status of Near-RT RIC deployment**

- Results similar to the output shown below indicate a complete and successful deployment, all are either **“Completed”** or **“Running”**, and that none are **“Error”**, **“Pending”**, **“Evicted”**,or **“ImagePullBackOff”**.
- The status of pods “**PodInitializing”** & **“Init”** & **“ContainerCreating”** mean that the pods are creating now, you need to wait for deploying.

```bash
kubectl get pods -A
```

![image.png](https://github.com/bmw-ece-ntust/Near-RT-RIC/blob/master/image/image%204.png)

### 3. Install the DMS tool

- Install Chatmuseum server

```bash
docker run --rm -u 0 -it -d -p 8080:8080 -e DEBUG=1 -e STORAGE=local -e STORAGE_LOCAL_ROOTDIR=/chart -v $(pwd)/charts:/charts chartmuseum/chartmuseum:latest
```

![image.png](https://github.com/bmw-ece-ntust/Near-RT-RIC/blob/master/image/image%205.png)

- Install DMS tool

```bash
cd ~
git clone https://gerrit.o-ran-sc.org/r/ric-plt/appmgr -b g-release
cd appmgr/xapp_orchestrater/dev/xapp_onboarder
apt-get install python3-pip
sudo apt install -y python3.8
pip3 uninstall xapp_onboarder
pip3 install ./
chmod 755 /usr/local/bin/dms_cli
ls -la /usr/local/lib/ptyhon3.8
chmod -R 755 /usr/local/lib/python3.8
```

## Appendix

### 1. Install buildkit and nerdctl

<aside>
💡

In the K release, it uses K8S v1.28 that just support containerd as container repository source. Thus, we need to install these tools to build and management images for containerd 

</aside>

```bash
wget https://github.com/moby/buildkit/releases/download/v0.11.3/buildkit-v0.11.3.linux-amd64.tar.gz
tar xf buildkit-v0.11.3.linux-amd64.tar.gz
cp bin/buildkitd bin/buildctl /usr/local/bin/

cat >> /lib/systemd/system/buildkitd.socket << EOF
[Unit]
Description=BuildKit
Documentation=https://github.com/moby/buildkit

[Socket]
ListenStream=%t/buildkit/buildkitd.sock

[Install]
WantedBy=sockets.target
EOF

cat >> /lib/systemd/system/buildkitd.service << EOF
[Unit]
Description=BuildKit
Requires=buildkitd.socket
After=buildkit.socket
Documentation=https://github.com/moby/buildkit

[Service]
ExecStart=/usr/local/bin/buildkitd --oci-worker=false --containerd-worker=true

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now buildkitd
systemctl status buildkitd
```

```bash
wget https://github.com/containerd/nerdctl/releases/download/v1.2.0/nerdctl-1.2.0-linux-amd64.tar.gz
tar xf nerdctl-1.2.0-linux-amd64.tar.gz
mv nerdctl /usr/local/bin/
```

### 2. Install kubenetes and xApp alias

```bash
git clone https://github.com/SoSonoFuriren/alias.git
cd alias/
./install.sh aliassu
```
