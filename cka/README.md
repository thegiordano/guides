## k setup
```bash
export do="--dry-run=client -o yaml" # k create deploy nginx --image=nginx $do
export now="--force --grace-period 0"   # k delete pod x $now

alias k='kubectl'
complete -o default -F __start_kubectl k # completion for the alias 'k' 
source <(kubectl completion bash) # completion for kubectl

alias kcurl='k run tmp --restart=Never --rm -i --image=nginx:alpine -- curl -m 5'
```

## vim setup
```bash
~/.vimrc
set tabstop=2
set expandtab
set shiftwidth=2 	
```

## upd image of pod
```bash
k set image po nginx nginx=nginx:17.1
```

## interact w/ pods
```bash
# create a netshoot (busybox-based) pod that sleeps infinitely; the image is useful to troubleshoot network issues; https://github.com/nicolaka/netshoot
k run tmpb --image=nicolaka/netshoot --command -- sleep infinity
```

```bash
k exec tmpb -it -- sh # --std-in or -i keeps the standard input open for the command/container. This allows you to interact with the shell 
# or
k exec tmpb --std-in --tty -- /bin/sh # --tty or -t allocates a pseudo-terminal
```

```bash
# a simple 'nslookup' w/o further interaction
k exec tmpb -- nslookup go.dev
```

## upgrade a node
```bash
# control plane
kubeadm token create --print-join-command
# keep the output available (e.g. in another terminal tab)

# ssh into the target node
kubeadm upgrade plan # / or "node" instead of plan
systemctl stop kubelet

kubeadm version # check versions
kubelet --version

apt update
apt show kubelet # check available upgrade
apt show kubeadm

apt install kubelet=1.29.0=1.1 # install specific version
apt install kubeadm=1.29.0=1.1

systemctl start kubelet # start the upgraded kubelet

# now use the kubeadm join command; it was provided as the output of 'kubeadm token create --print..'
```

## certificates
```bash
# ssh into a control plane node and run:
openssl x509  -noout -text -in /etc/kubernetes/pki/apiserver.crt | grep Validity -a2 # check validity of kubeapi-server

# same w/ kubeadm
kubeadm certs check-expiration | grep apiserver
```

## resources consumption
```bash
k -n moon top po
```

## labels / annotate
```bash
kubectl label po -l app=v2 tier=web  # add label tier=web to pods with label app=v2
kubectl label po nginx{1..3} app-  # remove label app from 3 pods
kubectl annotate po nginx{1..3} description='my description'  # annotate 3 pods
```

## networking
```bash
k -n pluto expose pod project-plt-6cc-api --name project-plt-6cc-svc --port 3333 --target-port 80  # expose an already created pod
```

## network policies
```yaml
# allow access only to pods with label db1 or db2 on specific port
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: np-backend
  namespace: project-snake
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress                    # policy is only about Egress
  egress:
    - to:                           # first condition "to"
      - podSelector:
          matchLabels:
            app: db1
      ports:                        # second condition "port"
      - protocol: TCP
        port: 1111
    - to:                           # first condition "to"
      - podSelector:
          matchLabels:
            app: db2
      ports:                        # second condition "port"
      - protocol: TCP
        port: 2222
```

## resource quota
```bash
kubectl create quota myrq --hard=cpu=1,memory=1G,pods=2  # hard limits
```

## scale / autoscale
```bash
kubectl scale deploy nginx --replicas=5
kubectl autoscale deploy nginx --min=5 --max=10 --cpu-percent=80
# view the horizontalpodautoscalers.autoscaling for nginx
kubectl get hpa nginx
```

## jobs
```bash
k create job pi  --image=perl:5.34 -- perl -Mbignum=bpi -wle 'print bpi(2000)'
# Add job.spec.completions=5 // Add job.spec.parallelism=5 // Add job.spec.activeDeadlineSeconds=30

# cronjobs
# Add cronjob.spec.startingDeadlineSeconds=17 // Add cronjob.spec.jobTemplate.spec.activeDeadlineSeconds=12
```

## configmaps
```bash
kubectl create cm configmap3 --from-env-file=config.env  # from env file // can also be --from-literal & --from-file
kubectl get cm configmap3 -o yaml  ## inspect

# dry run from literal; $do defined in the setup (1st paragraph)
k create cm testcm --from-literal=peka=boo $do
```

## probes
```bash
k get events -A | grep -i "Liveness probe failed" | awk '{print $1,$5}'  # list all pods where liveness probe failed; format: <namespace>/<pod name> per line
```

```yaml
# probe w/ commands
spec:
  containers:
  - image: nginx:1.16.1-alpine
    name: ready-if-service-ready
    livenessProbe:
        exec:
        command:
        - 'true'
    readinessProbe:
        exec:
        command:
        - sh
        - -c
        - 'wget -T2 -O- http://service-am-i-ready:80'
dnsPolicy: ClusterFirst
```

## commands inside containers
```yaml
spec:
  containers:
  - name: command-demo-container
    image: debian
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10; done"]
```

## role / rolebindings
```bash
k -n project-hamster create role processor --verb=create --resource=secret --resource=configmap # role
k -n project-hamster create rolebinding processor --role processor --serviceaccount project-hamster:processor # rolebinding (SA)

# test the role
k auth can-i create secret --as system:serviceaccount:project-hamster:processor -n project-hamster
```

## secrets
```bash
k -n neptune describe secret neptune-secret-1  # decode a token in a secret
k -n neptune get secret neptune-secret-1 -o json

# createa a generic secret
k -n secret create secret generic secret2 --from-literal=user=user1 --from-literal=pass=1234
```

## helm
```bash
helm create chart-test #### this would create a helm
helm install -f myvalues.yaml myredis ./redis
helm upgrade -f myvalues.yaml -f override.yaml redis ./redis

helm pull [chart URL | repo/chartname] [...] [flags] #### this would download a helm, not install 
helm pull --untar [repo/chartname] ## untar the chart after downloading it 

helm -n mercury ls
helm -n mercury uninstall internal-issue-report-apiv1

helm -n mercury upgrade internal-issue-report-apiv2 bitnami/nginx

helm show values bitnami/apache  ## reveal values // add grep to look for replicas number
helm -n mercury install internal-issue-report-apache bitnami/apache --set replicaCount=2  ## set replica N

helm -n mercury ls -a  ## show all
helm -n mercury uninstall internal-issue-report-daniel  ## uninstall
```

## docker / podman
```bash
sudo docker build -t registry.killer.sh:5000/sun-cipher:latest -t registry.killer.sh:5000/sun-cipher:v1-docker .  ## build 2 images w/ tags
sudo docker image ls  ## list images
sudo docker save buzz:1 -o /home/cloud_user/buzz_1.tar ## save a copy of the image to .tar

sudo docker push registry.killer.sh:5000/sun-cipher:latest  ## push to repo
```
```bash
podman build -t registry.killer.sh:5000/sun-cipher:v1-podman .  ## build 1 image w/ tag
podman image ls
podman push registry.killer.sh:5000/sun-cipher:v1-podman

podman run -d --name sun-cipher registry.killer.sh:5000/sun-cipher:v1-podman  ## create a container
podman ps  ## some info
podman logs sun-cipher > /opt/course/11/logs  ## write logs into file
```
