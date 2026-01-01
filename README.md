# k8sScwvue.com

Setup sancapweather.com in a kubernetes cluster as deployment named scwvuecom; expose as service scwvuecom; connect to the internet as sancapweather.com through an ingress called scwvuecom-ingress. The Vantage Vue weather station in Sanibel working with the Davis Envoy receiver collects the realtime data.  Cumulus weather software running on a Windows 10 VM collects the data and produces station graphes that the Saratoga weather software (this kubernetes deployment) displays. 

Source image [InstallSCWvue.com](https://github.com/jkozik/InstallSCWvue.com)

# build image, put on docker hub
SanCapWeather.com is running in a docker container on my host, directly -- not in a VM.  To make it run in my kubernetes cluster, I need to push the image to my dockerhub repository.  The kubernetes deployment resource has an "image" field that triggers a pull of this image from dockerhub.
```
[jkozik@dell2 k8sNw.net]$ docker tag jkozik/scw.com jkozik/scw.com:v1
[jkozik@dell2 k8sNw.net]$ docker push jkozik/scw.com:v1
The push refers to a repository [docker.io/jkozik/scw.com]
df6373f27d47: Pushed
26f167811204: Pushed
f3493de6be65: Pushed
5f3f1afd37f8: Pushed
9cb1c8944404: Pushed
17ef7ac3bc3c: Pushed
80e1222a1520: Pushed
84cddc3bfd9f: Pushed
91eac03e1433: Pushed
9fe29cfcdaad: Pushed
c907f065ed59: Mounted from jkozik/nw.com
1920409046ed: Mounted from jkozik/nw.net
1b04f74cccfe: Mounted from jkozik/nw.net
9f80175acb76: Mounted from jkozik/nw.net
4394b53db28c: Mounted from jkozik/nw.net
5b35dc81a1f5: Mounted from jkozik/nw.net
76d251b895a5: Mounted from jkozik/nw.net
7f5982d81e05: Mounted from jkozik/nw.net
ea7c1c005303: Mounted from jkozik/nw.net
df83fc87c7b6: Mounted from jkozik/nw.net
17e81ae0b70b: Mounted from jkozik/nw.net
e80100d0d319: Mounted from jkozik/nw.net
cc0f976c1376: Mounted from jkozik/nw.net
67415c00b86b: Mounted from jkozik/nw.net
ffc9b21953f4: Mounted from jkozik/nw.net
v1: digest: sha256:2ba522ab2d6b7f9ba41dce3d3b47948e8eb3b95c971237c3da726e1d8fc965c1 size: 5554
```
## Deploy
Clone the yaml files in this repository.  One can apply them one at a time, but it works fine to apply them all at once.  On the kubectl host, run the following to deploy all in one command:
```
[jkozik@dell2 ~]$ git clone https://github.com/jkozik/k8sScw.com
Cloning into 'k8sScw.com'...
remote: Enumerating objects: 14, done.
remote: Counting objects: 100% (14/14), done.
remote: Compressing objects: 100% (10/10), done.
remote: Total 14 (delta 1), reused 5 (delta 0), pack-reused 0
Unpacking objects: 100% (14/14), done.

[jkozik@dell2 ~]$ cd k8sScw.com/
[jkozik@dell2 k8sScw.com]$ ls
README.md  scwcom-deploy.yml  scwcom-ingress.yml  scwcom-svc.yml

[jkozik@dell2 k8sScw.com]$ kubectl apply -f .
deployment.apps/scwcom created
ingress.networking.k8s.io/scwcom-ingress created
service/scwcom created
```
## Verify that the application is running
```
[jkozik@dell2 k8sScw.com]kubectl get deployment,service,pod,ingress
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/scwcom                1/1     1            1           112s

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/scwcom            NodePort    10.104.191.227   <none>        80:31436/TCP   111s

NAME                                       READY   STATUS    RESTARTS   AGE
pod/scwcom-74dc4585cf-d5ws8                1/1     Running   0          112s


NAME                                                CLASS    HOSTS                     ADDRESS           PORTS   AGE
ingress.networking.k8s.io/scwcom-ingress            <none>   sancapweather.com         192.168.100.174   80      112s
```
## HomeLAN NATing from external IP address / port 80 to cluster's LAN address and ingress controller's port number.
So, on my home LAN http://192.168.100.174:30410 is where incoming web traffic enters.  The ingress controller parses for sancapweather.com and redirects the traffic to the scwcom service. I have an external IP address for my home LAN that gets NAT'd by my home router to 192.168.100.174.  On that NAT box, I map port 80 to port 30410. 

## Check the web page http://sancapweather.com
```
[jkozik@dell2 k8sScw.com]$  kubectl -n ingress-nginx get service
NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.97.70.29     <none>        80:30140/TCP,443:30023/TCP   23d
ingress-nginx-controller-admission   ClusterIP   10.111.250.10   <none>        443/TCP                      23d
```
## Check the web page http://sancapweather.com
```
[jkozik@dell2 k8sScw.com]$ curl -H "Host: sancapweather.com" 192.168.100.173:30140 | head
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3706    0  3706    0     0    835      0 --:--:--  0:00:04 --:--:--   835<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
   "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
  <head>
    <!-- ##### start AJAX mods ##### -->
    <script type="text/javascript" src="ajaxAWNwx.js"></script>
    <!-- AJAX updates by Ken True - http://saratoga-weather.org/wxtemplates/ -->
    <script type="text/javascript" src="ajaxgizmo.js"></script>
    <script type="text/javascript" src="language-en.js"></script>
        <!-- language for AJAX script included -->
100  8004    0  8004    0     0   1804      0 --:--:--  0:00:04 --:--:--  2332
```
## Ingress --> HttpRoute
I have installed the Gateway API on this cluster.  It is called ` Gateway running in `envoy-gateway-system` namespace.

The HTTPRoute yaml points sancapweather.com and www.sancapweather.com from my Cloudflare tunnel to port 30458.

Ingress has been depricated.
