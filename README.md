![Linkerd](/assets/images/linkerd.png)

# First of ALL 🎈 # Bem Vindos! 💚 👏👏👏
___Really Thanks to my friends of Buoyant and Linkerd project - Catherine Paganini, Jason and Bill Morgan and Flynn, without you, we can't reach multiclusters in mesh ( because others meshs, are slipslop )___

_Esse repositório foi escrito em 2022/Dezembro, podendo ser atualizado os binários e os paths, [contate-me](mailto:adonai.costa@gmailcom) se precisar de ajuda! Mas tente muito primeiro antes de me contatar ;-)_

A idéia aqui é mostrar o quão fácil é criar e manter multiclusters com service mesh que realmente funciona, é uma necessidade que até então era _hype_ e agora vêem como uma necessidade real, visto que há sim indisponibilidade nas clouds ou em nossos ambientes _onpremisse_.

O linkerd, se mostrou capaz de realizar essa tarefa, com facilidade, rastreabilidade, segurança e observabilidade, outros service meshs apresentam limitações mil que nos impedem de dar implementar e dar manutenção e sequer conseguirmos fazer um simples troubleshoot.

Agradeço tambem ao time [Getup](https://getup.io) que sempre está disposto a desenvolver, validar, testar e disseminar seus conhecimentos com a comunidade


## Instale as ferramentas que te deixarão doido pra criar clusters em mesh. 

**kind:** https://kind.sigs.k8s.io/docs/user/quick-start/#installation  
**kubectl:** https://kubernetes.io/docs/tasks/tools/  
**helm:** https://helm.sh/docs/intro/install/  
**step:** https://smallstep.com/docs/step-cli/installation  
**linkerd:** https://github.com/linkerd/linkerd2/releases/tag/stable-2.13.3  
**linkerd-smi:** https://github.com/linkerd/linkerd-smi/releases/tag/v0.2.1  

Bora lá!

O kind como todo mundo sabe, cria cluster kubernetes na sua máquina pessoal para testes e validações, utilíssimo pra voce não fazer cocô no seu ambiente produtivo.  
O kubectl e helm não precisam de apresentações.  
O step é um utilitário que facilita a criação de certificados que vamos usar. 
O binário linkerd, obviamente foi criado com a graça de Deus e pelos meus amigos da Buoyant para criar, atualizar, interagir com o service mesh mais gostosinho do mundo. 
E por fim o linkerd-smi é o complemento/extensão usado pra criar custom resources como o TrafficSplit, que pode inclusive interagir com o flagger para rollout de deploys canary automáticos.


## Crie 2 kind clustes na sua máquina, YES YOU CAN!!!! Go! Go! Go! 

> Isso vai ser instalado na sua máquina, então o endereço do ApiServer precisa ser alcançavel na sua maquina!!!

A versão de kubernetes esta exposta na imagem, 1.24.7

```bash 
cat << EOF >> kind.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: demo
networking:
  apiServerAddress: 192.168.1.61 # PUT YOUR IP ADDRESSS OF YOUR MACHINE, HERE DUMMIE! ;-)
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
nodes:
- role: control-plane
  image: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
- role: worker
  image: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
- role: worker
  image: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
EOF

kind create cluster --config kind.yaml 
```

Próximo cluster

```bash
cat << EOF >> kind3.yaml  
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: demo3
networking:
  apiServerAddress: 192.168.1.61 # PUT YOUR IP ADDRESSS OF YOUR MACHINE, HERE DUMMIE! ;-)
  podSubnet: "10.245.0.0/16"
  serviceSubnet: "10.97.0.0/12"
nodes:
- role: control-plane
  image: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
- role: worker
  image: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
- role: worker
  image: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
EOF

kind create cluster --config kind3.yaml 
```

Viu a diferença? os dois clusters têm IP pra pods e services diferentes, isso ajuda no tshoot, localização de serviços e não conflita IPs no trafego de pacotes entre os clusters


## Instale o metallb para obter LoadBalancers nos seus clusters

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
for ctx in kind-demo kind-demo3; do
 helm install metallb -n metallb-system --create-namespace metallb/metallb --kube-context=${ctx}
done 
```

> Crie um `ipaddresspool` para cada cluster baseado em sua rede docker kind, mas lembre-se para cada cluster, um `ipaddresspool` diferente, para evitar conflito.

Para isso, primeiro descubra o range de IP do seu kind network

```bash
docker network inspect kind -f '{{.IPAM.Config}}'
{172.17.0.0/16  172.17.0.1 map[]} {fc00:f853:ccd:e793::/64  fc00:f853:ccd:e793::1 map[]}]
```

O meu é `172.17.0.16`, então vou utilizá-lo na configuração de IPs para meus services do tipo `LoadBalancer`.

```bash
kubectl --context=kind-demo apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.17.0.61-172.17.0.70

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  labels:
    todos: verdade
  name: example
  namespace: metallb-system
EOF
```

E DEPOIS....

```bash
kubectl --context=kind-demo3 apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.17.0.71-172.17.0.80

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  labels:
    todos: verdade
  name: example
  namespace: metallb-system
EOF
```

Essa é a hora de voce validar se sua máquina consegue se conectar a um serviço nos clusters, do tipo `LoadBalancer`.

```bash
# cria um deploy bobo
kubectl create deploy web --image nginx --context=kind-demo

# expoe o deploy como LoadBalancer
kubectl expose deploy web --port 80 --target-port 80 --type LoadBalancer --context=kind-demo

# verifica o IP designado ao seu service
kubectl get service web --context=kind-demo
... bla bla bla
NAMESPACE              NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
default                web                                  LoadBalancer   10.101.106.156   172.17.0.61   80:31507/TCP                    20s

# faz um teste pra ver se chega
curl 172.17.0.61
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; } .... bla bla bla bla
```

Deu certo? Pode apagar,

```bash
kubectl delete deploy,svc web --context=kind-demo
```

Faça o mesmo no outro cluster mudando o `--context=kind-demo3`

Em caso de problemas verifique:   
- firewall dentro da sua máquina,  
- rede correta do docker network kind  
- se voce distribuiu corretamente os `ipaddresspool` nos metallb sem repeti-los nos clusters, a idéia é cada cluster tem um pool de IPs diferente para LoadBalancers.



## Instalando Linkerd e sua familia que vai nos ajudar com o MultiCluster

Antes de mais nada, é preciso criar a cadeia de certificado pro linkerd , a ser instalado em todos os clusters igualmente

```bash
mkdir certs && cd certs
step certificate create \
  root.linkerd.cluster.local \
  ca.crt ca.key --profile root-ca \
  --no-password --insecure
step certificate create \
  identity.linkerd.cluster.local \
  issuer.crt issuer.key \
  --profile intermediate-ca \
  --not-after 8760h --no-password \
  --insecure --ca ca.crt --ca-key ca.key
```

Guarde os certificados, eles são umas das partes core do seu linkerd, mas se perder voce precisará reinstalar e reiniciar pods que estão em mesh

Roda essa bagaça aqui ainda no diretório certs com os certificados criados

```bash
alias lk='linkerd'
alias ka='kubectl apply -f '

for ctx in kind-demo kind-demo3; do                   
  echo "install crd ${ctx}"
  lk install --context=${ctx} --crds | ka - --context=${ctx};

  echo "install linkerd ${ctx}";
  lk install --context=${ctx} \
    --identity-trust-anchors-file=ca.crt \
    --identity-issuer-certificate-file=issuer.crt \
    --identity-issuer-key-file=issuer.key | ka - --context=${ctx};

  echo "install viz ${ctx}";
  lk --context=${ctx} viz install | ka - --context=${ctx};

  echo "install multicluster ${ctx}";    
  lk --context=${ctx} multicluster install | ka - --context=${ctx};

  echo "install smi ${ctx}";        
  lk smi install --context=${ctx}  | ka - --context=${ctx};
done
```

Cheque se seu linkerd-multicluster recebeu um ip LoadBalancer e faça um telnet pra eles só pra validar

```bash
for ctx in kind-demo kind-demo3; do
  printf "Checking cluster: ${ctx} ........."
  while [ "$(kubectl --context=${ctx} -n linkerd-multicluster get service linkerd-gateway -o 'custom-columns=:.status.loadBalancer.ingress[0].ip' --no-headers)" = "<none>" ]; do
      printf '.'
      sleep 1
  done
  echo "`kubectl --context=${ctx} -n linkerd-multicluster get service linkerd-gateway -o 'custom-columns=:.status.loadBalancer.ingress[0].ip' --no-headers`"
  printf "\n"
done
```


## Hora de ligar os clusters

Essa é a hora em que o linkerd cria serviceaccounts nos 2 clusters, cria secret com o kubeconfig de cada um cruzado e voilá!

```bash
lk --context=kind-demo multicluster link --cluster-name kind-demo | ka - --context=kind-demo3
lk --context=kind-demo3 multicluster link --cluster-name kind-demo3 | ka - --context=kind-demo
```

Vamos ver?

```bash
for ctx in kind-demo kind-demo3; do
  echo "Checking link....${ctx}"
  lk --context=${ctx} multicluster check

  echo "Checking gateways ...${ctx}"
  lk --context=${ctx} multicluster gateways

  echo "..............done ${ctx}"
done
```


## Vamos instalar o podinfo

___Um detalhe aqui muito importante,___
- como estamos usando o kind entao o service de DNS do cluster é `kube-dns.kube-system.svc.cluster.local`, se estiver usando K8s Gerenciado por alguma cloud ou kubespray verifique o nome do service e edite o `configMap` do `multicluster/base/frontend.yml`.
- atente-se se mudar o nome dos clusters, é necessrário também mudar o nome das pastas em `multicluster/kind-demo`.
- se for ocaso faça um fork ou clone e mude os nomes e conteúdo, mas estude o repositório, veja o que ele faz, ___be curious___.

```bash
for ctx in kind-demo kind-demo3; do
  echo "Adding test services on cluster: ${ctx} ........."
  kubectl --context=${ctx} create ns test
  kubectl --context=${ctx} apply \
    -n test -k "github.com/adonaicosta/linkerd-multicluster/multicluster/${ctx}/"

  kubectl --context=${ctx} -n test \
    rollout status deploy/podinfo || break
  echo "-------------"
done
```

Verifique se os pods estão no ar

```bash
for ctx in kind-demo kind-demo3; do
  echo "Check pods on cluster: ${ctx} ........."
  kubectl get pod -n test --context=${ctx}
done
```

Vamos acessar a aplicação e validar se o frontend consegue bater em ambos os pods, para cada cluster ainda em separado, *não estamos em multicluster ainda hein!*

```bash
kubectl port-forward -n test svc/frontend 8080:8080 --context=kind-demo
```

Abra seu browse http://localhost:8080

Viu?!

![kind-demo](/assets/images/kind-demo.gif)

Agora faça para o outro cluster

```bash
kubectl port-forward -n test svc/frontend 8080:8080 --context=kind-demo3
```

Abra seu browse http://localhost:8080

Viu?!

![kind-demo3](/assets/images/kind-demo3.gif)



## Chegou a hora de ver o crosscluster funcionar

Para que os pods frontend em cada cluster consigam acessar os podinfo do outro, é necessário espelhar o serviço de podinfo entre os clusters.  
A aplicação frontend já faz a chamada para cada cluster pois foi pré-configurada.  
Então vamos etiquetar o serviço podinfo em cada cluster e depois criar um trafficsplit para cada um.

Etiquetando ...

```bash
for ctx in kind-demo kind-demo3; do
  echo -en "\n\nLabel svc podinfo on cluster: ${ctx} .........\n"
  kubectl label svc -n test podinfo mirror.linkerd.io/exported=true --context=${ctx}
  sleep 4

  echo "Check services transConnected (if word exists )....on cluster ${ctx}"
  kubectl get svc -n test --context=${ctx}

done
```

Agora em cada cluster vai existir um serviço podinfo-(CONTEXTO) que aponta pro outro cluster no IP do serviço linkerd-multicluster/linkerd-gateway.

```bash
kubectl get svc --context=kind-demo -n linkerd-multicluster linkerd-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
172.17.0.61
kubectl get endpoints --context=kind-demo3 -n test podinfo-kind-demo -o jsonpath='{.subsets[*].addresses[*].ip}'
172.17.0.61
```

São os mesmos certo? Isso diz que o endpoint do serviço test/podinfo-kind-demo3 aponta pro serviço LoadBalancer do linkerd-multicluster/linkerd-gateway do cluster kind-demo.

Faça o mesmo pro outro cluster

```bash
kubectl get svc --context=kind-demo3 -n linkerd-multicluster linkerd-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
172.17.0.71
kubectl get endpoints --context=kind-demo -n test podinfo-kind-demo3 -o jsonpath='{.subsets[*].addresses[*].ip}'
172.17.0.71
```


## Distribuindo a carga entre os clusters

Criando o trafficsplit ...

```bash
kubectl --context=kind-demo apply -f - <<EOF
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: podinfo
  namespace: test
spec:
  service: podinfo
  backends:
  - service: podinfo
    weight: 50
  - service: podinfo-kind-demo3
    weight: 50
EOF    
```

```bash
kubectl --context=kind-demo3 apply -f - <<EOF
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: podinfo
  namespace: test
spec:
  service: podinfo
  backends:
  - service: podinfo
    weight: 50
  - service: podinfo-kind-demo
    weight: 50
EOF    
```

Dividimos a carga do podinfo em 50% para cada cluster, para cada um, então temos um `cross access` entre eles.  
Faça um port-forward novamente ( se já cancelou anteriormente ) para um frontend e acompanhe.

```bash
kubectl port-forward -n test --context=kind-demo svc/frontend 8080
```

Abra seu browser de novo http://localhost:8080 

Voilá...

![traffic-split running](/assets/images/trafficsplit-running.gif)

Se voce fizer port-forward no mesmo serviço frontend do outro cluster o resultado vai ser o mesmo.


## Instalando ingress controller, criando um ingress e simulando a vida real como ela é

Instale o ingress-nginx em cada um dos clusters.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
for ctx in kind-demo kind-demo3; do
  echo "Instalando o ingress-nginx - ${ctx}"
  helm install ingress-nginx -n ingress-nginx --create-namespace ingress-nginx/ingress-nginx --set controller.podAnnotations.linkerd.io/inject=enabled --kube-context=${ctx}
  
  echo "Aguarde a instalação do ingress-nginx terminar, 30s, acho"
  kubectl rollout status deploy -n ingress-nginx ingress-nginx-controller --context=${ctx}

  echo "Criando um ingress para o podinfo - ${ctx}"
  kubectl --context=${ctx} -n test create ingress frontend --class nginx --rule="frontend-${ctx}.127-0-0-1.sslip.io/*=frontend:8080" --annotation "nginx.ingress.kubernetes.io/service-upstream=true"
done
```

Pegue o fqdn da sua aplicação pod info em cada cluster e adicione no seu /etc/hosts

```bash
for ctx in kind-demo kind-demo3; do
  echo "Adicione o hostname e IP no seu /etc/hosts e aguarde o ingress-nginx atribuir IP ao ingress, uns 30s"
  kubectl --context=${ctx} get ingress -n test 
done
```

> Não esqueca de colocar no seu /etc/hosts o IP host criados acima

![ingress em cada cluster](/assets/images/ingress.gif)

## Conclusão

Até agora vimos que multicluster em mesh, seguro, fácil e unplugged nao tem muito segredo, e sim muitos passos a atentar

- Verifique se o linkerd nos clusters tem a mesma chave TLS, criada com openssl ou step como demonstrado
- Instale o linkerd-multicluster para estabeler conectividade entre eles
- Instale o linkerd-smi para criar distribuição de tráfego
- Crie objetos trafficsplit para distribuir esse acesso
- Lembre-se, cada cluster no exemplo tem uma URL, é a hora de colocar seu DNS pra trabalhar com nao somente roundrobin, mas com health checks e failover
- O linkerd tem ainda uma extensao para multicluster failover, o que vai aumentar ainda mais a disponibilidade de aplicações entre clusters, que é o que se pretende afinal. Procure saber mais.

### _Referências:_

**Getup:** https://getup.io  
**Linkerd:** https://linkerd.io  
**ServiceMesh:** https://smi-spec.io/  
**An Really Service Mesh:** https://linkerd.io 😙  

