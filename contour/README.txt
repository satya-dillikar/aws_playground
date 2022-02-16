kubectl get svc -n nginx-sample-traffic -w
NAME             TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)          AGE
apple-service    LoadBalancer   10.100.200.238   a44f31932a5324352b5ba0907d0a86eb-56430427.us-west-2.elb.amazonaws.com     5678:30894/TCP   17s
banana-service   LoadBalancer   10.100.195.213   a182f2b350b4e40f59493c620f3f4053-1389915846.us-west-2.elb.amazonaws.com   5678:32160/TCP   17s

curl http://a44f31932a5324352b5ba0907d0a86eb-56430427.us-west-2.elb.amazonaws.com:5678
apple

curl http://a182f2b350b4e40f59493c620f3f4053-1389915846.us-west-2.elb.amazonaws.com:5678
banana



-------
AWS EKS CLUSTER



1) Install Step:
https://projectcontour.io/guides/deploy-aws-nlb/

https://github.com/projectcontour/contour/tree/v1.20.0/examples/contour
download 02-service-envoy.yaml

Edit the Envoy service (02-service-envoy.yaml) in the examples/contour directory:
Remove the existing annotation: service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
Add the following annotation: service.beta.kubernetes.io/aws-load-balancer-type: nlb

$ kubectl get service envoy --namespace=projectcontour -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
ad7d4e55d96c74828b0f13ff7156570b-1502591712.us-west-2.elb.amazonaws.com%

kubectl get service envoy --namespace=projectcontour -o yaml |grep nlb
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{"service.beta.kubernetes.io/aws-load-balancer-backend-protocol":"nlb"},"name":"envoy","namespace":"projectcontour"},"spec":{"externalTrafficPolicy":"Local","ports":[{"name":"http","port":80,"protocol":"TCP","targetPort":8080},{"name":"https","port":443,"protocol":"TCP","targetPort":8443}],"selector":{"app":"envoy"},"type":"LoadBalancer"}}
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: nlb

2) DNS Setup
kubectl get service envoy --namespace=projectcontour
NAME    TYPE           CLUSTER-IP     EXTERNAL-IP                                                               PORT(S)                      AGE
envoy   LoadBalancer   10.100.2.103   ad7d4e55d96c74828b0f13ff7156570b-1502591712.us-west-2.elb.amazonaws.com   80:31035/TCP,443:32755/TCP   24h

Add A-record with reference to AWS resource (Alias to Application and Classic LoadBalancer)
contour.vmwaremarketplace.com to ingress address (ad7d4e55d96c74828b0f13ff7156570b-1502591712.us-west-2.elb.amazonaws.com)

3) kaurd 
kubectl apply -f kuard-specs.yaml -f kuard-ingress.yaml
namespace/kuard-ns created
deployment.apps/kuard created
service/kuard created
ingress.networking.k8s.io/kuard created

kubectl get pods,svc,ingress -n kuard-ns
NAME                         READY   STATUS    RESTARTS   AGE
pod/kuard-798585497b-jphbq   1/1     Running   0          13s
pod/kuard-798585497b-kdsxt   1/1     Running   0          13s
pod/kuard-798585497b-wkh8p   1/1     Running   0          13s

NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/kuard   ClusterIP   10.100.226.162   <none>        80/TCP    13s

NAME                              CLASS     HOSTS                           ADDRESS                                                                   PORTS   AGE
ingress.networking.k8s.io/kuard   contour   contour.vmwaremarketplace.com   ad7d4e55d96c74828b0f13ff7156570b-1502591712.us-west-2.elb.amazonaws.com   80      13s

Test
✗ curl http://contour.vmwaremarketplace.com/

4) http-echo

kubectl apply -f example-http-echo-pods.yaml -f example-http-echo-service-contour.yaml
namespace/nginx-sample-traffic created
pod/banana-app created
pod/apple-app created
service/banana-service-contour created
service/apple-service-contour created
➜  http-echo git:(origin/main) ✗ kubectl get pods,svc,ingress -n nginx-sample-traffic
NAME             READY   STATUS    RESTARTS   AGE
pod/apple-app    1/1     Running   0          22s
pod/banana-app   1/1     Running   0          22s

NAME                             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/apple-service-contour    ClusterIP   10.100.44.127    <none>        5678/TCP   21s
service/banana-service-contour   ClusterIP   10.100.201.219   <none>        5678/TCP   21s
➜  http-echo git:(origin/main) ✗ kubectl apply -f example-ingress-contour.yaml
ingress.networking.k8s.io/example-ingress-contour created
➜  http-echo git:(origin/main) ✗ kubectl get ingress -n nginx-sample-traffic
NAME                      CLASS     HOSTS                           ADDRESS                                                                   PORTS   AGE
example-ingress-contour   contour   contour.vmwaremarketplace.com   ad7d4e55d96c74828b0f13ff7156570b-1502591712.us-west-2.elb.amazonaws.com   80      8s
curl http://contour.vmwaremarketplace.com/apple
apple
curl http://contour.vmwaremarketplace.com/banana
banana
curl http://contour.vmwaremarketplace.com/junk
404 page not found
