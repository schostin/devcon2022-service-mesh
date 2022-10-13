# Spickzettel Präsentation

<http://istio-demo-bookinfo.sebastianneb.de/productpage>

curl -v istio-demo.sebastianneb.de/test

## Istio ServiceEntry Debugging

kubectl create namespace outgoing-traffic
kubectl label namespace outgoing-traffic istio-injection=enabled
kubens outgoing-traffic
kubectl run -i --tty --rm debug --image=curlimages/curl --restart=Never -- sh
curl -v <https://www.google.de>
kc apply -f outgoing-traffic/serviceentry-google.yaml
kc apply -f outgoing-traffic/serviceentry-google-overwrite.yaml
kc delete serviceentry google-https
kc apply -f outgoing-traffic/serviceentry-google.yaml
kc delete serviceentry google-https -n echoserver
