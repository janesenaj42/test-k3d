## Pre-requisites
- Docker Desktop 4.17.1 (runs on WSL2)
- [k3d](https://github.com/k3d-io/k3d/releases/download/v5.4.4/k3d-windows-amd64.exe)
- [helm](https://get.helm.sh/helm-v3.11.2-windows-amd64.zip)
- [kubectl](https://dl.k8s.io/release/v1.26.0/bin/windows/amd64/kubectl.exe)
- (Optional) [k9s](https://webinstall.dev/k9s)

## Introduction
Purpose is to familiarize with k3d functionalities by setting up:
- nginx
- mongodb


1. Create cluster.
    ```
    # k3d local-path-provisioner writes data to `/var/lib/rancher/k3s/storage`

    mkdir data & k3d cluster create asdf -p 80:80@loadbalancer -p 443:443@loadbalancer -v %cd%/data:/var/lib/rancher/k3s/storage@all 
    ```
2. Create new storage class with `ReclaimPolicy: Retain` (local-path-provisioner by default sets to `Delete`).
    ```
    # sc-retain.yaml

    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
        name: retain-local-path
    provisioner: rancher.io/local-path
    volumeBindingMode: WaitForFirstConsumer
    reclaimPolicy: Retain
    ```
    ```
    kubectl apply -f ./k3d/sc-retain.yaml
    ```

Now the cluster is ready! Run `kubectl cluster-info` and  `kubectl get pods -A` to check. All non-job pods should be running.

Bonus: expose Traefik dashboard to host by running `kubectl port-forward traefik-xxx 9000:9000 -n kube-system` where `traefik-xxx` is the name of the pod.

## NGINX
1. Create persistent volume to attach to nginx (nginx from bitnami does not come with persistent volume (no pv/pvc.yaml in tempaltes)).
    ```
    # pvc-nginx.yaml

    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
        name: pv-nginx
    spec:
        storageClassName: retain-local-path
        accessModes:
          - ReadWriteOnce
        resources:
            requests:
                storage: 5Gi
    ```
    ```
    kubectl apply -f ./k3d/pvc-nginx.yaml
    ```

2. Install nginx server with helm:
    ```
    # nginx-values.yaml

    extraVolumes:
        - name: pv-nginx
          persistentVolumeClaim:
            claimName: pv-nginx

    extraVolumeMounts:
        - name: pv-nginx
          mountPath: /app/asdf

    service:
        type: ClusterIP

    ingress:
        enabled: true
        hostname: nginx.<wsl ipv4 ip>.nip.io
    ```
    ```
    helm repo add bitnami https://charts.bitnami.com/bitnami

    helm install -f ./helm/nginx-values.yaml nginx bitnami/nginx
    ```

3. Create a file in the `pvc-xxx-nginx` folder created in Introduction step 1 for testing.
    ```
    echo 'this is my sample data' > test.json
    ```
4. SSH into the nginx pod and confirm that this test.json is in `/app/asdf`.

5. Access `http://nginx.<wsl ipv4 ip>.nip.io/asdf/test.json` in the browser. You should see 'this is my sample data'. Yaye Traefik routed the nginx data!

## MongoDB
Assuming MongoDB runs at `27017`,
1. Add port to the k3d cluster (k3d runs k3s on docker!!) 
    ```
    k3d cluster edit asdf --port-add 27017:27017@loadbalancer
    ```
    Verify that `27017` is mapped to host by running `docker ps | find "k3d-asdf-serverlb-"`.

2. Configure Traefik to route extra ports for MongoDB.
    ```
    # ingressroutetcp-mongo.yaml

    kind: IngressRouteTCP
    metadata:
        name: mongodb-ingress-tcp
        namespace: default
    spec:
        entryPoints:
            - mongodb
    routes:
        - match: HostSNI(`*`)
          services:
            - name: mongodb
              port: 27017
    ---
    apiVersion: helm.cattle.io/v1
    kind: HelmChartConfig
    metadata:
        name: traefik
        namespace: kube-system
    spec:
        valuesContent: |-
            additionalArguments:
            - "--entryPoints.mongodb.address=:27017/tcp"
            ports:
                mongodb:
                port: 27017
                expose: true
                exposedPort: 27017
                protocol: TCP
    ```

    ```
    kubectl apply -f ./k3d/ingressroutetcp-mongo.yaml
    ```

3. Install MongoDB with helm. Tested as standalone architecture.
    ```
    # mongo-values.yaml

    persistence:
        storageClass: "retain-local-path"
    ```

    ```
    helm install -f ./helm/mongo-values.yaml mongodb bitnami/mongodb
    ```

4. Connect to MongoDB using tools like Studio 3T/Robo3T at `mongo.<wsl ipv4 ip>.nip.io:27017` using the default credentials. (NOTE: the exact domain name doesn't matter since we configured `HostSNI(`*`)` instead of `HostSNI(`mongo.\<wsl ipv4 ip>.nip.io`)`).