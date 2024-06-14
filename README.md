# Add let's encrypt support to multicloud ingress trait

These steps are now simplified becuase let's encrypt support is generally available. 


## configure the capabilties and traits

1. update the ingress capabiltiy on the cluster group to use let's encrypt. In the advanced yaml section for the package add the following and apply the change

```yaml
clusterIngressCa:
  acme:
    email: your-email
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

