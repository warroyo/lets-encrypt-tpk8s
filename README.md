# Add let's encrypt support to multicloud ingress trait

Currently there is an issue in the production version of the multi-cloud ingress trait that prevents acme from working. This workaround applies a new version of the package that has a fix to the clusters and the project. 

## apply the package yaml to the clusters

This package needs to be applied to any clusters that will be using LE.

1. get the cluster kubeconfig
```bash
tanzu ops cluster kubeconfig get <cluster-name> -m eks -p eks -t eks > cluster.conf
export KUBECONFIG=./cluster.conf
```

2. apply the package yaml to the cluster

```bash
kubectl apply -f mutlicloud-ingress.yml
```


## apply the package yaml to the project


1. setup the kubeconfig for talking direcly to the project context

```bash
tanzu project use <your project>
export KUBECONFIG=/Users/<your-user>/.config/tanzu/kube/config
```

2. apply the package yaml to the project

```bash
kubectl apply -f mutlicloud-ingress.yml
```


## configure the capabilties and traits

1. update the ingress capabiltiy to use let's encrypt. In the advanced yaml section for the package add the following and apply the change

```yaml
clusterIngressCa:
  acme:
    email: yopur-email
    server: "https://acme-v02.api.letsencrypt.org/directory"
    solver:
      route53:
        accessKeyID: "aws-accesskey"
        accessKeySecret: "aws-secretkey"
        hostedZoneID: "zone-id"
        region: "us-west-2"
```

2. update the multicloud-ingress trait in the networking profile and set `UseClusterIssuer` to true


## Deploy your app

Now when you deploy an app and httproute you will see that the issuer used is backed by LE and it should creates some dns challenges and eventually generate the cert.  

