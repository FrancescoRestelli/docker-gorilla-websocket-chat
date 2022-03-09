
# Server setup:
```
az group create --name myResourceGroup --location northeurope
az aks create -n myCluster -g myResourceGroup --network-plugin azure --enable-managed-identity -a ingress-appgw --appgw-name myApplicationGateway --appgw-subnet-cidr "10.2.0.0/16" --generate-ssh-keys
az aks get-credentials -n myCluster -g myResourceGroup

git clone https://github.com/FrancescoRestelli/docker-gorilla-websocket-chat.git  

#sets a very long timeout to be sure we are not getting dropped by that
helm install --debug gorilla-websocket-chat --set ingress.tls=false --set ingress.hosts[0].host="ws.contoso.com" --set ingress.hosts[0].paths[0]="/" --set ingress.enabled=true --set-string ingress.annotations."appgw\.ingress\.kubernetes\.io/request-timeout"="6000"  --set ingress.annotations."kubernetes\.io/ingress\.class"=azure/application-gateway chart/gorilla-websocket-chat
```

# client setup
```
PUBLICIP=$(kubectl get ingress |grep gorilla-websocket-chat|cut -d " " -f10)
#make sure public ip is populated you might have to wait a bit
echo $PUBLICIP


curl -L https://github.com/vi/websocat/releases/download/v1.9.0/websocat_linux64 -o websocat
chmod +x websocat

echo "$PUBLICIP ws.contoso.com" | sudo tee -a /etc/hosts

#open 2x bash  and run websocat in both to have 2 working websocket clients:

./websocat  ws://ws.contoso.com/ws --print-ping-rtts --ping-interval 10
```

you can now chat between the 2 bash terminals and will see a ping each 10s

now to the issue:
with this 2 active websocket connections open as soon as any change happens to the agic rules they are disconnected.
first example unrelated rule change:
kubectl apply -f https://raw.githubusercontent.com/Azure/application-gateway-kubernetes-ingress/master/docs/examples/aspnetapp.yaml 
after that the chat windows are broken and must be restarted, this should not happen

second example scaling the websocket backend by settings replicaCount2:  
helm upgrade --debug gorilla-websocket-chat --set replicaCount=2  --set ingress.tls=false --set ingress.hosts[0].host="ws.contoso.com" --set ingress.hosts[0].paths[0]="/" --set ingress.enabled=true --set-string ingress.annotations."appgw\.ingress\.kubernetes\.io/request-timeout"="6000"  --set ingress.annotations."kubernetes\.io/ingress\.class"=azure/application-gateway chart/gorilla-websocket-chat

in both cases the websocket is closed after ~10s which defeats the purpose of autoscaling websocket servers using agic since all clients will reconnect causing even more load.