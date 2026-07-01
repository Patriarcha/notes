# Kubernetes Fundamentals (LFS258)

Kubernetes to open source projekt otwarty przez Google w 2014 roku.

Koncepcja **context** w Kubernetes to kombinacja klastra, użytkownika i opcjonalnie namespace'a.

### Przełączanie contextu

```bash
kubectl config use-context foobar
```

## Wtyczki sieciowe (CNI)

Różne pluginy sieciowe dla K8S:

- **Calico** (proste)
- **flannel** (proste)
- **kube-router** (zaawansowane)
- **Cilium** (zaawansowane)

## Metody instalacji Kubernetes

- **Kubespray, Kops, Kind** — różne metody instalacji Kubernetes
- **Minikube** — lokalny setup
- **kind** — klastry lokalne oparte na Dockerze
- **kops** — zarządzanie klastrami zorientowane na AWS
- **kubeadm** — bootstrapuje klastry produkcyjne

### Pytania, na które trzeba odpowiedzieć przed instalacją

- Dostawca infrastruktury (public cloud / on-prem / private cloud)
- System operacyjny: Ubuntu, CentOS, Debian, Fedora CoreOS, Red Hat CoreOS
- Sieć: czy potrzebna jest overlay network do komunikacji pod-to-pod?
- Umiejscowienie etcd: zewnętrzne / współdzielone / embedded
- HA: failover / redundancja

### Opcje środowiska kontenerowego

`cri-o`, `containerd`, `Docker`

## Konfiguracja obowiązkowa

```bash
swapoff -a
modprobe overlay
modprobe br_netfilter

cat << EOF | tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Wymuszenie zmian w aktualnie działającym jądrze
sysctl --system

# Konfiguracja containerd z domyślnymi wartościami po instalacji
containerd config default | tee /etc/containerd/config.toml
sed -e 's/SystemdCgroup = false/SystemdCgroup = true/g' -i /etc/containerd/config.toml
```

## Instalacja Kubernetes

```bash
apt install -y kubeadm=1.34.2-1.1 kubelet=1.34.2-1.1 kubectl=1.34.2-1.1
apt-mark hold kubeadm kubelet kubectl
```

IP: `10.244.3.192/25`

> Uwaga: należy używać aliasu node'a, a nie adresu IP, aby certyfikaty sieciowe działały poprawnie po wdrożeniu load balancera w przyszłym labie.

### Minimalna konfiguracja

```yaml
# root@cp:~# cat kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: 1.34.2
controlPlaneEndpoint: "k8scp:6443"
networking:
  podSubnet: 192.168.0.0/16
```

### Inicjalizacja klastra

```bash
kubeadm init \
  --config=kubeadm-config.yaml \
  --upload-certs \
  --node-name=cp \
  | tee kubeadm-init.out
```

#### Przykładowy output

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes running the following command on each as root:

  kubeadm join k8scp:6443 --token ij3nge.qunp34duai9arqqv \
        --discovery-token-ca-cert-hash sha256:f829e45ef65f2e2d0a11f10432a15b07908dac314603f510829da8f8d3dc9c8f \
        --control-plane --certificate-key 2d719ed6027524af9f2ed917ae5164c5a455233f5d5d83ad84bdaa4badacb79f

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8scp:6443 --token ij3nge.qunp34duai9arqqv \
        --discovery-token-ca-cert-hash sha256:f829e45ef65f2e2d0a11f10432a15b07908dac314603f510829da8f8d3dc9c8f
```

**CNI** = Container Networking Interface. Sieć musi umożliwiać komunikację: container-to-container, pod-to-pod, pod-to-service, external-to-service.

### Dodatkowe polecenia

```bash
# Włączenie bash completion
echo "source <(kubectl completion bash)" >> $HOME/.bashrc

# Pokazanie konfiguracji init
sudo kubeadm config print init-defaults
# 10.244.4.42

# Wydrukowanie komendy join
sudo kubeadm token create --print-join-command
```

Przykład (tylko poglądowy, nie do użycia wprost):

```bash
kubeadm join \
  k8scp:6443 --token bn67ni.2elimydao1l4j4ke \
  --discovery-token-ca-cert-hash \
  sha256:ae4f395146c7278f06d2bc08092c6e2f3d8f7df207aeeec685f1a50aca8e4fed \
  --node-name=worker

kubeadm join k8scp:6443 --token bn67ni.2elimydao1l4j4ke --discovery-token-ca-cert-hash sha256:ae4f395146c7278f06d2bc08092c6e2f3d8f7df207aeeec685f1a50aca8e4fed
```

### Taints

```bash
# Sprawdzenie taintów
kubectl describe node | grep -i taint

# Usunięcie tainta
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

> `crictl` jest instalowany przez kubeadm jako zależność i działa jako CLI dla containerd.

### Podstawowe operacje na deploymentach

```bash
# Najprostszy deployment
kubectl create deployment nginx --image=nginx

# Dry run, tylko podgląd konfiguracji
kubectl create deployment two --image=nginx --dry-run=client -o yaml

# Skalowanie
kubectl scale deployment nginx --replicas=3

# Wyświetlenie ENV w podzie
kubectl exec nginx-1423793266-13p69 -- printenv | grep NGINX

# Utworzenie service dla deploymentu
kubectl expose deployment nginx --type=LoadBalancer

# Pobranie wielu zasobów naraz (przecinek)
kubectl get deploy,pod
```

Edycja istniejących zasobów: `apply`, `edit`, `patch`, `replace --force` (odpowiednik edit, także dla pól, których nie można zaktualizować po inicjalizacji).

## Architektura Kubernetes

- Kubernetes ma **API-centryczny design**
- Tylko API ma dostęp do etcd (magazyn danych key:value)
- Kubelet uruchamia pody i wykonuje zadania
- **API** = centralny punkt control plane
- Na node'ach **kubelet** utrzymuje i wykonuje zadania, komunikując się z API na control plane, żeby wdrażać pody/obrazy, obsługiwać config, wysyłać status i metryki do API
- Każdy node Kubernetes (control plane lub worker) uruchamia: **Kubelet**, **Kube-proxy** i **container runtime** (containerd lub CRI-O)
- **Operatory** pomagają wykonywać złożone zadania, np. automatyzację wdrażania, skalowania, upgrade'u bazy danych za pomocą prostej instrukcji wysokiego poziomu przez API
- **Services** definiują logiczny zbiór podów i politykę dostępu do nich. Udostępniają aplikacje działające w klastrze innym aplikacjom wewnątrz klastra lub użytkownikom zewnętrznym
- **Pods** — najmniejsza jednostka wdrożeniowa w Kubernetes. Zawiera jeden lub więcej kontenerów
- **Namespace** pozwala organizować zasoby w klastrze, dzieląc pojedynczy klaster na wirtualne klastry. Resource quotas są stosowane w obrębie namespace'a
- **Sieć** w Kubernetes jest obsługiwana przez network plugin, który zarządza ruchem wewnętrznym i zewnętrznym
- **Storage** — Kubernetes zapewnia elastyczne opcje przechowywania danych dla aplikacji stanowych

### Diagram: wywołanie API przez kubectl (curl)

```
                              ┌────────────────────────────────────────────────────┐
                              │              CONTROL PLANE (CP) NODE                │
                              │                                                      │
                              │   ┌───────────────────────┐                         │
                              │   │ kube-controller-manager│                         │
                              │   └───────────┬───────────┘                         │
                              │               │↕                                     │
                              │               ▼                  ┌──────────────┐    │
                              │   ┌───────────────────────┐◄─────┤kube-scheduler│    │
              kubectl         │   │                       │      └──────────────┘    │
           ───────────────────┼──►│     kube-apiserver    │                         │         WORKER NODES
           API call (curl)    │   │                       ├────┐                    │      ┌───────────────────────────┐
                              │   └───────────┬───────────┘    │                    │      │ kubelet  ┄┄► containerd   │
                              │               │↕                │  ──────────────────┼─────►│                       ⬡⬡ │
                              │               ▼                 │                    │      │ kube-proxy ─► iptables/  │
                              │   ┌───────────────────────┐     │                    │      │              eBPF        │
                              │   │         etcd          │     │                    │      └───────────────────────────┘
                              │   │      (cylinder DB)    │     │                    │
                              │   └───────────────────────┘     │                    │      ┌───────────────────────────┐
                              │                                 │                    │      │ kubelet  ┄┄► containerd   │
                              │      ┌──────────────┐           ├────────────────────┼─────►│                      ⬡⬡⬡ │
                              │      │   kubelet    │·· (dotted)│                    │      │ kube-proxy ─► iptables/  │
                              │      └──────────────┘           │                    │      │              eBPF        │
                              │      ┌──────────────┐           │                    │      └───────────────────────────┘
                              │      │ kube-proxy   │·· (dotted)│                    │
                              │      └──────────────┘           │                    │      ┌───────────────────────────┐
                              │                                 │                    │      │ kubelet  ┄┄► containerd   │
                              │   ┌───────────────────────┐     │                    │      │                        ⬡ │
                              │   │ cloud-controller-mgr  │·····┘                    │      │ kube-proxy ─► iptables/  │
                              │   └───────────┬───────────┘                         │      │              eBPF        │
                              │               ┊                                      │      └───────────────────────────┘
                              └───────────────┊──────────────────────────────────────┘
                                              ┊
                                          ☁ (cloud)
```

### Upgrade klastra

```bash
kubeadm upgrade plan
```

Weryfikuje kompatybilność przed upgrade'em Kubernetes.

### Zewnętrzne, wyspecjalizowane konektory

`cloud-controller-manager` wspiera integrację z Rancher, DigitalOcean i innymi narzędziami firm trzecich.

### Typowe dodatki (add-ony) klastra

- Usługi DNS (**CoreDNS**)
- Ingress controllers
- Logowanie na poziomie klastra
- Monitorowanie zasobów

## Control Plane Node

Węzeł Control Plane składa się z podów i procesów. Podczas tworzenia klastra przez `kubeadm`, uruchamiane są pody, których definicje znajdują się w `/etc/kubernetes/manifests/` — obejmuje to API server, scheduler, controller manager.

- **kube-apiserver** — centralny hub klastra Kubernetes, jako jedyny łączy się z bazą etcd
- **Konnectivity** poprawia wydajność sieci, oddzielając ruch inicjowany przez użytkownika od ruchu inicjowanego przez serwer
- **kube-scheduler** decyduje, gdzie umieścić pody. Jego politykę mogą wpływać taints, policies, bindings, a nawet wdrożenie własnego (custom) schedulera
- **etcd** — rozproszony magazyn key-value (B+tree), przechowujący wszystkie dane klastra: stan, konfigurację i informacje sieciowe. Dopisuje tylko nowe informacje (append-only), obsługuje operacje HTTP, np. przez curl
- `etcdctl` — polecenie do wykonywania snapshot save/restore w celach backupu
- **kube-controller-manager** — główny daemon pętli kontrolnej, który monitoruje stan klastra przez kube-apiserver. Gdy widzi rozbieżność między stanem pożądanym a rzeczywistym, podejmuje działania naprawcze
- **cloud-controller-manager** odciąża kube-controller-manager z zadań specyficznych dla chmury, umożliwiając integrację z zewnętrznymi dostawcami chmury. Współpracuje z API chmury do zarządzania load balancerami czy storage. Musi być uruchomiony z flagą `--cloud-provider=external`
- **CoreDNS** zastąpił kube-dns jako domyślny serwer DNS

## Worker Nodes

- **Kubelet** — krytyczny agent działający na każdym node, odpowiedzialny za zarządzanie cyklem życia kontenerów. Komunikuje się z control plane przez kube-apiserver, aby otrzymywać instrukcje, jakie pody mają działać na jego node. Następnie współpracuje z silnikiem container runtime na node, aby zapewnić, że działa to, co powinno. Monitoruje aktualny stan i raportuje go z powrotem do control plane
- **kube-proxy** — kolejny kluczowy komponent na każdym node, odpowiedzialny za zarządzanie łącznością sieciową podów. Domyślnie używa iptables do konfiguracji reguł routingu. Alternatywnie wspiera tryb IPVS (IP Virtual Server) dla konkretnych przypadków użycia

## Add-ony

Kubernetes nie zawiera wbudowanego logowania na poziomie całego klastra — jest ono obsługiwane przez zewnętrzne rozwiązania, np. Fluentd.

Podobnie ze zbieraniem metryk na poziomie klastra: robi to metrics-server, ale w bardzo ograniczonym zakresie. Do bardziej szczegółowych metryk używa się Prometheus jako rozwiązania monitorującego i alertującego.

## Kubelet

Kubelet otrzymuje specyfikację poda (**PodSpec**) z KubeAPI i robi wszystko, żeby przywołać pod do życia, w tym tworzenie kontenerów, zarządzanie wolumenami przechowywania, obsługę danych konfiguracyjnych, takich jak Secrets i ConfigMaps. PodSpec to plik YAML lub JSON.

- Przetwarza PodSpecs
- Montuje wolumeny
- Obsługuje Secrets i ConfigMaps
- Komunikuje się z Container Runtime
- Raportuje status Node i Pod

Kubelet współpracuje też z zaawansowanymi komponentami, takimi jak **Topology Manager**. Topology Manager optymalizuje alokację zasobów, wykorzystując wskazówki od innych komponentów, aby przydzielać zasoby, takie jak CPU i akceleratory sprzętowe, w oparciu o topologię NUMA (Non-Uniform Memory Access).

## Operatory

Operatory to w zasadzie pętle-strażnicy (watchdogs) zbudowane z dwóch komponentów: **Informer** i **pętla rekoncyliacji**. Operatory są silnikiem automatyzacji Kubernetes.

Service operator — IP „przykleja się" do service, a IP poszczególnych podów nie ma znaczenia i nie trzeba go śledzić.

## Pody

Pody to najmniejsza jednostka w Kubernetes — mogą zawierać jeden lub wiele kontenerów.

- **InitContainer** jest potrzebny, aby uruchamiać kontenery w określonej kolejności, ponieważ wszystkie kontenery w podzie startują równolegle
- Każdy pod ma przypisany pojedynczy adres IP, współdzielony przez wszystkie kontenery w tym podzie
- Kontenery komunikują się między sobą przez inter-process communication (IPC), interfejs loopback lub współdzielony system plików
- **Sidecar** jest używany głównie do monitorowania, logowania lub proxy jako uzupełnienie głównego kontenera

Konfiguracja kontenerów może być ustawiona w sekcji `resources` specyfikacji poda (PodSpec).

InitContainer może pomóc w:
- prekonfiguracji środowiska (tworzenie katalogów, ustawianie uprawnień)
- czekaniu na zewnętrzne zależności (baza danych lub usługa), aż staną się dostępne
- uruchamianiu skryptów setupowych lub narzędzi nieuwzględnionych w głównej aplikacji

Przykład: initContainer czekający na dostępność katalogu bazy danych:

```yaml
spec:
  containers:
  - name: main-app
    image: databaseD
  initContainers:
  - name: wait-database
    image: busybox
    command: ['sh', '-c', 'until ls /db/dir ; do sleep 5; done; ']
```

## Wywołanie API

```bash
kubectl create deployment test1 --image=httpd --v=10
```

Pokazuje, że wykonano zwykłe wywołanie CURL z parametrami do utworzenia nowego deploymentu.

```bash
kubectl get deploy,rs,pod
```

### ⚠️ Flow — bardzo ważne

`kubectl create` (wywołanie API curl) trafia do `kube-apiserver`, który zapisuje to wywołanie POST w swojej pamięci, a następnie przekazuje je do `etcd`.

Następnie `kube-controller-manager` cyklicznie pyta apiserver: „czy moja specyfikacja się zmieniła?". Apiserver odpowiada „tak, została wysłana zmiana" i przekazuje nową specyfikację. Wtedy `kube-controller-manager` pyta, czy ta specyfikacja istnieje. Apiserver odpowiada, że nie, więc controller manager każe API ją utworzyć. Wtedy API wywołuje etcd i mówi, że jest utworzona. Deployment zostaje zapisany, ale replicaset operator robi to samo — pyta, czy istnieje replicaset operator, roundtrip odbywa się ponownie w obie strony, po czym jest tworzony — to samo dzieje się z podem. Pod operator pyta, czy istnieje pod — nie istnieje, więc znowu roundtrip.

Aby umieścić poda gdzieś, apiserver pyta `kube-scheduler`, gdzie powinien trafić. Kube-scheduler odpowiada, na którym node pod powinien działać. Wtedy API wywołuje bezpośrednio kubelet na wybranym node, informując go, że będzie obsługiwał tego poda.

API informuje również `kube-proxy` na każdym node o nowej konfiguracji sieciowej.

Następnie kubelet odpowiada za pobranie wszystkich configmaps, secrets, montowanie systemów plików, a potem wysyła informację do silnika Docker/CRI-O, aby utworzyć poda. Gdy to się stanie, silnik wysyła informację z powrotem do kubeleta, a ten przekazuje ją do apiserver, który zapisuje to w etcd. Wtedy może odpowiedzieć kube-controller-managerowi, że pod istnieje, gdy ten zapyta.

## Node

Node — pojedyncza maszyna. Workery (Linux lub Windows) oraz control-plane nodes (tylko Linux).

- **NodeLease** to status zdrowia wysyłany do apiserver. Jeśli apiserver nie otrzyma statusu zdrowia node przez pięć minut, oznacza status node jako `NotReady`
- Obiekty **NodeLease** są przechowywane w specjalnym namespace `kube-node-lease`, dedykowanym do zarządzania informacjami heartbeat node'ów. Można je sprawdzić poleceniem `kubectl describe node <node-name>`
- `kubectl delete node <node-name>` wyrejestrowuje node z API server i usuwa jego pody. `kubeadm reset` na node czyści konfigurację specyficzną dla klastra. Jeśli node ma być ponownie użyty, konieczne jest ręczne wyczyszczenie artefaktów sieciowych, takich jak reguły iptables

```bash
# Inspekcja szczegółów sieci i wolumenów poda
kubectl describe pod <pod-name>
```

### Pod

- Pody są izolowane z założenia
- **Kontener pause** odpowiada za zarządzanie pojedynczym IP dla wielu kontenerów. Kontenery pause nie są widoczne w Kubernetes — widoczne są tylko za pomocą narzędzi niskopoziomowych, jak docker czy crictl
- Gdy tworzony jest service, tworzony jest obiekt **Endpoint**, który zapisuje adres IP poda wraz z portem docelowym. Następnie service typu **NodePort** mapuje wysokonumerowany port na każdym node (30000–32767) do tego Endpointa. Umożliwia to klientom zewnętrznym dostęp do adresu IP node'a i konkretnego portu, który jest przekazywany do poda

## Services

- **ClusterIP** to domyślny typ Service w Kubernetes, używany do komunikacji WEWNĄTRZ klastra. Nie jest to adres IP samego klastra — to wirtualny adres tworzony i utrzymywany przez Kubernetes
- Service może udostępniać dostęp jako: **NodePort** service, **Ingress Controller** lub proxy. Service może też kierować ruch do grupy podów „backendowych", umożliwiając rozkład obciążenia między wieloma replikami aplikacji

## Komunikacja sieciowa

- **Container-to-container** jest rozwiązywana wewnątrz poda, ponieważ wszystkie kontenery współdzielą tę samą przestrzeń nazw sieciowych i komunikują się przez interfejs loopback (localhost)
- **Pod-to-pod** wymaga pluginu CNI w klastrze, aby przypisać unikalne adresy IP każdemu podowi i umożliwić routing między podami na wszystkich node'ach
- **External-to-pod** wymaga użycia Kubernetes Services, takich jak ClusterIP, NodePort lub LoadBalancer, aby umożliwić dostęp zewnętrzny do podów
- Domyślna komunikacja pod-to-pod nie obsługuje w naturalny sposób komunikacji między podami na różnych node'ach (cross-node)
- Zasady zdefiniowane przez Kubernetes dla komunikacji pod-to-pod: brak NAT, wszystkie pody muszą się komunikować ze sobą na wszystkich node'ach, wszystkie node'y muszą komunikować się ze wszystkimi podami w klastrze

> Przydatny link ilustrujący sieciowanie: [Illustrated Guide to Kubernetes Networking](https://speakerdeck.com/thockin/illustrated-guide-to-kubernetes-networking)
