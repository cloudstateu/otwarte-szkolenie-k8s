<img src="./img/logo.png" alt="Chmurowisko logo" width="200" align="right">
<br><br>
<br><br>
<br><br>

# Praca domowa część 1

## Wstęp

Witaj w pracy domowej (część 1), w której uruchomisz aplikację składającą się z dwóch skonteneryzowanych usług (React oraz Node.js) na klastrze Kubernetes.

Na początek kilka informacji organizacyjnych:

1. Pierwszym krokiem pracy domowej jest zainstalowanie Docker. Zaprezentowaliśmy dwa scenariusze instalacji: na maszynie wirtualnej w Azure (tak jak na szkoleniu) lub na własnym sprzęcie. Ćwiczenia napisane są z punktu widzenia maszyny wirtualnej w Azure. Możesz wybrać opcję, która jest dla Ciebie wygodniejsza. 
1. Zadania z gwiazdką (`*`) są opcjonalne do wykonania. Zachęcamy do ich wykonania, ale nie są one niezbędne do ukończenia pracy domowej.

W razie jakichkolwiek pytań lub wątpliwości wyślij wiadomość na [piotr.kalinowski@chmurowisko.pl](mailto:piotr.borowy@chumrowisko.pl) lub [daniel.pisarek@chmurowisko.pl](mailto:daniel.pisarek@chumrowisko.pl).

---

## Utwórz maszynę laboratoryjną z Ubuntu w Azure (*)

⚠️ **Uwaga**: nie musisz wykonywać tego kroku jeśli instalujesz Docker na własnym sprzęcie.

### Po co? 

Domyślnie w [Azure Cloud Shell](https://docs.microsoft.com/en-us/azure/cloud-shell/overview) nie możemy korzystać ze wszystkich funkcji Docker. Rozwiązaniem tego problemu jest stworzenie dedykowanej maszyny wirtualnej z Ubuntu na której zainstalujemy Docker Engine i wykorzystamy pełen potencjał kontenerów.

### Kroki
 
1. Uruchom [Azure Cloud Shell](https://shell.azure.com)
1. Wywołaj poniższy snippet w Azure Cloud Shell. Zmień wartość zmiennej `RESOURCE_GROUP_NAME` na nazwę Resource Group do której masz dostęp i możesz tworzyć zasoby. W snippet użyjesz komendy [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) `az vm create`, która stworzy maszynę wirtualną (2 CPU, 7 GiB RAM, Ubuntu). Do maszyny możesz zalogować się korzystając z loginu i hasła podanych w zmiennych `ADMIN_USERNAME` oraz `ADMIN_PASSWORD`.

    ```shell
    RESOURCE_GROUP_NAME="<resource_group_name>"
    VM_NAME="labvm"
    ADMIN_USERNAME="ubuntu"
    ADMIN_PASSWORD='Chmurowisko123!@#'

    az vm create -n $VM_NAME -g $RESOURCE_GROUP_NAME \
        --image "UbuntuLTS" \
        --authentication-type password \
        --admin-username $ADMIN_USERNAME \
        --admin-password $ADMIN_PASSWORD \
        --public-ip-sku Standard \
        --size Standard_DS2_v2
    ```

1. Po utworzeniu maszyny wirtualnej zmień ustawienia Network Security Group (NSG) i zezwól na ruch przychodzący na portach 80 oraz 8000-9000. Dzięki dodaniu tej reguły będziesz mógł wyświetlić aplikacje uruchomione za pomocą Docker w Twojej przeglądarce.

    ```shell
    NSG_NAME="${VM_NAME}NSG"

    az network nsg rule create \
        --name Custom_8000_9000 \
        --nsg-name $NSG_NAME \
        --resource-group $RESOURCE_GROUP_NAME \
        --destination-port-ranges 80 8000-9000 \
        --protocol Tcp \
        --priority 1100
    ```

1. Znajdź publiczny adres IP maszyny i połącz się z maszyną wirtualną za pomocą SSH.

    Publiczny adres IP maszyny znajdziesz w Azure Portal lub możesz skorzystać ze skryptu listującego maszyny wirtualne i ich adresy IP:
    
    ```shell
    az vm list-ip-addresses --query "[].{name:virtualMachine.name, publicIp: virtualMachine.network.publicIpAddresses[0].ipAddress}"
    ```

1. Połącz się z maszyną wirtualną za pomocą SSH. Pamiętaj, aby uzupełnić ją publicznym adresem IP Twojej VM.

    ```shell
    ssh ubuntu@<VM-PUBLIC-IP>
    ```

## Zainstaluj Docker na swojej maszynie lub maszynie laboratoryjnej (*)

### Po co?

Docker będzie niezbędny do wykonania kolejnych ćwiczeń w tej pracy domowej oraz może być przydatny do pracy z kontenerami podczas Twojej pracy w przyszłości. Warto zainstalować go już teraz.

### Kroki

1. Wyświetl stronę [Install Docker Engine](https://docs.docker.com/engine/install/) i wybierz instrukcję instalacji odpowiednią dla swojego systemu operacyjnego. Wykonaj wszystkie kroki wymienione w instrukcji i sprawdź czy możesz uruchomić kontener z obrazem [hello-world](https://hub.docker.com/_/hello-world). Rezultat jaki otrzymasz po uruchomieniu kontenera powinien być podobny do poniższego:
    
    ![img](./img/1-terminal.png)

### Dodatkowe informacje

1. Brak uprawnień do wywołania komend Docker CLI

    W przypadku uruchamiania komend Docker CLI możesz spotkać się z problemem braku uprawnień:

    ```shell
    docker: Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/containers/create": dial unix /var/run/docker.sock: connect: permission denied.
    ```

    W tym przypadku powinieneś dodać swojego użytkownika (`ubuntu`) do grupy użytkowników `docker`. Skopiuj poniższą komendę i nie zmieniając jej treści wykonaj na maszynie wirtualnej z Linux. Następnie przeloguj swojego użytkownika (zakończ i rozpocznij na nowo sesję SSH).
    
    ```shell
    sudo usermod -aG docker $USER
    ```

1. Licencje Docker Desktop

    Jeśli instalujesz Docker na systemie operacyjnym z rodziny Linux wystarczy jeśli zainstalujesz Docker Engine. Docker Engine udostępniany jest na licencji Apache License 2.0 ([link](https://docs.docker.com/engine/#licensing)). Instalując Docker na systemie operacyjnym z rodziny Linux nie musisz instalować Docker Desktop.

    Jeśli instalujesz Docker na sprzęcie z Windows lub MacOS musisz zainstalować Docker Desktop. Docker Desktop podlega płatnej licencji w szczególnych przypadkach ([link](https://docs.docker.com/subscription/#docker-desktop-license-agreement)). Docker Desktop pozostaje darmowy gdy używamy go: na własny użytek, w celach edukacyjnych, w organizacjach non-profit lub komercyjnie w organizacjach zatrudniających mniej niż 250 pracowników i mających poniżej 10 000 000$ przychodu w skali roku.

## Zapoznaj się ze stronami pomocy Docker w terminalu

### Po co?

Wiedza w jaki sposób znaleźć strony pomocy dla interesujących nas komendy oraz jak takie strony wyglądają może ułatwić i przyspieszyć pracę w przyszłości. Naszym zdaniem strony pomocy w Docker CLI są czytelniejsze niż ich odpowiedniki w sieci Internet :)

### Kroki

1. Wyświetl stronę pomocy dla komendy `docker build`
1. Wyświetl stronę pomocy dla komendy `docker run`
1. Sprawdź w jaki sposób możesz podać zmienne środowiskowe uruchamiając kontener

### Podpowiedzi

1. Aby wyświetlić stronę pomocy użyj flagi `--help`
1. Zmienne środowiskowe dla kontenera możesz podać podczas uruchamiania go za pomocą komendy `docker run`. Mimo to, nie jest to jedyny sposób podania zmiennych środowiskowych do kontenera.

## Zbuduj i uruchom kontenery

### Po co?

Budowanie obrazów kontenera i uruchamianie ich jest chlebem powszednim pracy z kontenerami. Bez tego ani rusz 😁

### Kroki

1. Pobierz kod z tego repozytorium na dysk za pomocą `git clone`
1. Zbuduj każdą za aplikacji jako osobny obraz kontenera. Kod aplikacji React znajduje się w katalogu `/app/packages/frontend/`. Kod aplikacji API znajduje się w katalogu `/app/packages/api`. Nazwij obrazy kontenerów odpowiednio: 

    - dla aplikacji frontend - `app-frontend`
    - dla aplikacji API - `app-api`

1. Uruchom oba kontenery i sprawdź w przeglądarce czy otrzymujesz odpowiedzi jak poniżej:

    - Aplikacja: frontend, protokół: http, port: 80, path: `/`

        ![img](./img/2-frontend.png)

        Powyższy rezultat jest spodziewany, ponieważ aplikacja Frontend nie jest jest skonfigurowana z poprawnym adresem API z którego może pobrać dane. W tym momencie aplikacja odwołuje się pod domyślny adres `http://localhost:8001` zdefiniowany w kodzie aplikacji. Adres ten nie powinien w tym momencie w ogóle istnieć. Port w domyślnym adresie jest celowo niezgodny z portem na którym działa aplikacja API - `8888`. W kolejnych krokach tej pracy domowej ustawisz poprawny adres API za pomocą zmiennych środowiskowych.

    - Aplikacja: api, protokół: http, port: 8888, path: `/info`
        
        ![img](./img/3-backend.png)

### Podpowiedzi

1. Użyj komendy `docker build` do zbudownia kontenera oraz komendy `docker run` do uruchomienia kontenera.
1. Chcąc aby kontenery nie zajmowały Twojej sesji terminala możesz uruchomić je w tle za pomocą `docker run -d`.
1. Pamiętaj o odpowiedniej konfiguracji przekierowania portów za pomocą `docker run -p`. W przeciwnym razie nie otrzymasz odpowiedzi z aplikacji uruchomionej w kontenerze. Frontend działa na porcie 80, a API na porcie 8888.

## Zbuduj i uruchom kontenery w trybie "developerskim" (*)

### Po co?

Podczas codziennej pracy z kontenerami przydaje się możliwość obserwowania efektów pracy bez konieczności przebudowywania kontenera po każdej edycji kodu. Jest to szczególnie przydatne gdy nasz kontener buduje się długo. Tworząc odpowiedni Dockerfile, montując kod aplikacji z dysku hosta jako [volumen](https://docs.docker.com/storage/volumes/) oraz uruchamiając aplikację w kontenerze za pomocą dodatkowych programów (np. [nodemon](https://www.npmjs.com/package/nodemon) lub [CRA](https://create-react-app.dev/) development server) możemy osiągnąć efekt prawie natychmiastowego odświeżania kodu aplikacji po każdej zmianie.

### Kroki

1. Uruchom kontener w trybie developerskim
1. Upewnij się, że aplikacje zwracają spodziewane rezultaty
1. Zweryfikuj, że możesz zmienić kod bez konieczności przebudowywania kontenera

### Podpowiedzi

1. Aby uruchomić kontener w trybie deweloperskim powinieneś zbudować kontener z innym plikem Dockerfile. U nas jest to `Dockerfile_dev`. Dodatkowo, powinieneś zamontować folder z kodem z Twojego lokalnego dysku jako volumen na kontenerze.
1. ⚠️ **Uwaga**: jeśli na Twojej maszynie nie uruchomiłeś `npm install` to podczas podmontowywania lokalnego folderu z kodem musisz skonfigurować dodatkowy __anonymous volume__. Taki volumen spowoduje, że Docker podczas podmontowywania folderu z kodem pozostawił folder `/home/app/node_modules`, który stworzył podczas tworzenia kontenera. Aby stworzyć __anonymous volume__ zdefiniuj dodatkowy volumen uruchamiając `docker run`: `docker run -v /home/app/node_modules`. Jeśli nie zdefiniujesz __anonymous volume__ Docker nadpisze zawartość katalogu `/home/app` zawartością z lokalnego katalogu, który nie posiada `node_modules`. W efekcie program zwróci błąd z informacją o braku wymaganych zależności.

## Skomunikuj kontenery ze sobą

### Po co?

Aplikacja przygotowana w tej pracy domowej składa się z dwóch części - aplikacji Frontend oraz aplikacji API. Aplikacje są niezależnymi usługami sieciowymi. Architektura systemu została zaprojektowana tak, żeby aplikacja Frontend odpytywała o dane aplikację API. Odpytywanie aplikacji realizowane jest za pomocą protokołu HTTP.

W tym ćwiczeniu spróbujesz skomunikować kontenery ze sobą aby aplikacja Frontend mogła zaprezentować użytkownikowi dane o kursach walut.

### Kroki

1. Uruchom kontener z obrazem `app-frontend:latest` podając zmienną środowiskową `API_URL` z poprawnym adresem URL do aplikacji API
1. Uruchom kontener z obrazem `app-api:latest`
1. Sprawdź czy aplikacja Frontend prezentuje dane o kursach:

    ![img](./img/4-working-application.png)

### Podpowiedzi

1. W jednym z zadań tej pracy domowej dowiedziałeś się jak uruchomić kontener podając do niego zmienną środowiskową. Wykorzystaj poznany mechanizm w tym zadaniu.

### Dodatkowe informacje

1. W aplikacji React wykorzystujemy mechanizm odczytywania zmiennych środowiskowych opisany w artykule: https://www.freecodecamp.org/news/how-to-implement-runtime-environment-variables-with-create-react-app-docker-and-nginx-7f9d42a91d70/

## Uruchom kontenery za pomocą Docker Compose (*)

### Kroki

1. Zainstaluj Docker Compose na swoim systemie operacyjnym: https://docs.docker.com/compose/install/
1. Sprawdź wartość zmiennej środowiskowej `API_URL` w `docker-compose.yaml`. Upewnij się, że wartość zmiennej wskazuje na adres URL aplikacji 
1. Uruchom kontenery za pomocą Docker Compose

### Podpowiedzi

1. Uruchom kontenery za pomocą komendy `docker-compose up`
1. W repozytorium znajduje się plik `docker-compose.dev.yaml`. Pozwala on uruchomić kontenery w trybie developerskim. Możesz uruchomić kontenery w trybie developerskim używając komendy: `docker-compose -f docker-compose.dev.yaml up --build`. `--build` buduje obrazy kontenera przed uruchomieniem. Opcja przydaje się, gdy zmieniamy plik `docker-compose.yaml` na `docker-compose.dev.yaml` (i vice versa). Dzięki użyciu `--build`  możemy być pewni, że uruchamiamy kontenery zbudowane z poprawnych Dockerfile.

## Udostępnij kontenery w Docker Hub

### Po co?

Chcąc uruchomić skonteneryzowaną aplikację na klastrze musimy udostępnić obraz kontenera z aplikacją w repozytorium obrazów. W tym ćwiczeniu skorzystamy z Docker Hub, ponieważ jest to na razie jedyne repozytorium obrazów kontenerów jakie poznaliśmy na przestrzeni szkolenia.

### Kroki

1. Utwórz konto użytkownika w Docker Hub: https://hub.docker.com/signup. W razie problemów zobacz video z [instrukcją zakładania konta](https://www.youtube.com/watch?v=ty91qhd7L24)
1. Zaloguj się do Docker Hub za pomocą komendy `docker login` wywołanej w terminalu
1. Udostępnij obrazy w dedykowanych repozytoriach utworzonych w ramach Twojego konta na Docker Hub

    Aby udostępnić obrazy w Docker Hub musisz otagować je w taki sposób, aby zawierały Twoją nazwę użytkownika. W ten sposób Docker wie do że obrazy powinien udostępnić w repozytorium stworzonym na Twoim koncie.
    
    ```shell
    docker tag app-frontend <dockerID>/chmurobank-frontend
    docker tag app-api <dockerID>/chmurobank-api
    ```

1. Udostępnij obrazy w Docker Hub za pomocą komendy `docker push`

## Utwórz klaster AKS

### Po co?

Klaster Kubernetes jest niezbędny do wykonania kolejnych ćwiczeń 😁 

### Kroki

1. Uruchom Azure Cloud Shell
1. Utwórz klaster AKS korzystając z poniższej komendy (może zająć 5-10 minut). Uzupełnij `<RESOURCE-GROUP-NAME>` nazwą swojej Resource Group.

    ```shell
    az aks create -n cluster -g <RESOURCE-GROUP-NAME> --node-count 1
    ```

1. Skonfiguruj połączenie pomiędy terminalem, a klastrem

    ```shell
    az aks get-credentials -n cluster -g <RESOURCE-GROUP-NAME>
    ```

1. Sprawdź czy Node (maszyny wirtualnej) są w statusie `Ready`

### Dodatkowe informacje

Jeśli chcesz używać `az` z poziomu swojego terminala musisz go doinstalować. Instrukcje instalacji dla swojego systemu operacyjnego znajdziesz w dokumentacji: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli

## Uruchom aplikacje frontend i API na klastrze

### Po co?

W tym ćwiczeniu uruchomisz aplikację Frontend oraz API na klastrze i wykorzystasz potencjał zarządzania kontenerami jaki daje Ci Kubernetes.

### Kroki

1. Skorzystaj z poniższego template do stworzenia własnych Deployments

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    <deployment-label-key>: <deployment-label-value>
  name: <deployment-name>
spec:
  replicas: <number-of-replicas>
  selector:
    matchLabels:
      <pod-label-key>: <pod-label-value>
  template:
    metadata:
      labels:
        <pod-label-key>: <pod-label-value>
    spec:
      containers:
      - image: <repository-name-from-dockerhub>
        name: <container-name>
        ports:
        - containerPort: <application-port>
```

### Podpowiedzi

1. Skorzystaj z obiektów Deployment
1. Pamiętaj, że referując do obrazu kontenera w Docker Hub powinieneś użyć formatu `<username>/<image-name>[:<tag>]`)
1. W razie problemów z uruchomieniem aplikacji sprawdź informacje prezentowane przez `docker describe pod <pod-name>` oraz `docker logs <pod-name>`

## Utwórz kontener debug, którym sprawdzisz komunikację z Pod

### Po co?

W tym ćwiczeniu stworzymy tymczasowy Pod o nazwie debug. Będzie on używany tylko w celach developersko-diagnostycznych, żeby odpytać aplikację znajdujące się na klastrze bez publikowania ich do sieci Internet. Jest to częsty sposób na weryfikację działania aplikacji w Pod. 

### Kroki

1. Sprawdź adresy IP Pod z aplikacją frontend i API
1. Uruchom tymczasowy Pod z nazwą `debug` wykorzystujący kontener z obrazem `node`. Pod będzie używany w celach sprawdzenia odpowiedzi z Pod z aplikacjami.
1. Mając uruchomioną sesję bash w kontenerze z `node` sprawdź odpowiedzi z aplikacji za pomocą `curl`

    ```shell
    curl http://10.0.0.1
    curl http://10.0.0.1:8888/info
    ```

    ```shell
    # Expected output from frontend
    # curl http://10.0.0.1
    # <!doctype html><html lang="en"><head><meta charset="utf-8"/><link rel="icon" href="/favicon.png"/><meta name="viewport" content="width=device-width,initial-scale=1"/><meta name="theme-color" content="#000000"/><title>Chmurobank - szkolenie Chmurowisko 2021</title><link rel="preconnect" href="https://fonts.googleapis.com"><link rel="preconnect" href="https://fonts.gstatic.com" crossorigin><link href="https://fonts.googleapis.com/css2?family=Lato:ital,wght@0,400;0,700;1,300&display=swap" rel="stylesheet"><script src="/env-config.js"></script><link href="/static/css/2.a95be28f.chunk.css" rel="stylesheet"><link href="/static/css/main.ec668560.chunk.css" rel="stylesheet"></head><body><noscript>You need to enable JavaScript to run this app.</noscript><div id="root"></div><script>!function(e){function r(r){for(var n,f,l=r[0],i=r[1],a=r[2],c=0,s=[];c<l.length;c++)f=l[c],Object.prototype.hasOwnProperty.call(o,f)&&o[f]&&s.push(o[f][0]),o[f]=0;for(n in i)Object.prototype.hasOwnProperty.call(i,n)&&(e[n]=i[n]);for(p&&p(r);s.length;)s.shift()();return u.push.apply(u,a||[]),t()}function t(){for(var e,r=0;r<u.length;r++){for(var t=u[r],n=!0,l=1;l<t.length;l++){var i=t[l];0!==o[i]&&(n=!1)}n&&(u.splice(r--,1),e=f(f.s=t[0]))}return e}var n={},o={1:0},u=[];function f(r){if(n[r])return n[r].exports;var t=n[r]={i:r,l:!1,exports:{}};return e[r].call(t.exports,t,t.exports,f),t.l=!0,t.exports}f.m=e,f.c=n,f.d=function(e,r,t){f.o(e,r)||Object.defineProperty(e,r,{enumerable:!0,get:t})},f.r=function(e){"undefined"!=typeof Symbol&&Symbol.toStringTag&&Object.defineProperty(e,Symbol.toStringTag,{value:"Module"}),Object.defineProperty(e,"__esModule",{value:!0})},f.t=function(e,r){if(1&r&&(e=f(e)),8&r)return e;if(4&r&&"object"==typeof e&&e&&e.__esModule)return e;var t=Object.create(null);if(f.r(t),Object.defineProperty(t,"default",{enumerable:!0,value:e}),2&r&&"string"!=typeof e)for(var n in e)f.d(t,n,function(r){return e[r]}.bind(null,n));return t},f.n=function(e){var r=e&&e.__esModule?function(){return e.default}:function(){return e};return f.d(r,"a",r),r},f.o=function(e,r){return Object.prototype.hasOwnProperty.call(e,r)},f.p="/";var l=this.webpackJsonpfrontend=this.webpackJsonpfrontend||[],i=l.push.bind(l);l.push=r,l=l.slice();for(var a=0;a<l.length;a++)r(l[a]);var p=i;t()}([])</script><script src="/static/js/2.69378b64.chunk.js"></script><script src="/static/js/main.affd0915.chunk.js"></script></body></html>

    # Expected output from API
    # curl http://10.0.0.1:8888/info
    # {"status":"ok","timestamp":"2022-01-08T09:54:29.508Z","hostname":"api-5dd59b49b8-d8tx2","data":null}
    ```

### Podpowiedzi

1. Sesję bash możesz zakończyć komendą `exit`.

## Utwórz Service aby komunikować się z Pod posługując się znaną nazwą DNS

### Po co?

Każdy Pod na klastrze posiada swój unikalny adres IP. Komunikacja z Pod za pomocą adresów IP jest możliwa, ale nie jest wygodna (adresami IP trzeba zarządzać). W tym ćwiczeniu stworzysz obiekt typu Service, który przejmie odpowiedzialność za zarządzanie adresami IP i udostępni dla Ciebie jedną znaną nazwę DNS pod którą dostępne będą Twoje Pod. Korzystając z Service będziesz w stanie wykonywać requesty pod adres np. http://api nie znając konkretnych adresów IP dla każdego Pod oraz będziesz w stanie wykorzystać mechanizm Load Balancingu, który realizują Service.

### Kroki

1. Skorzystaj z poniższego template do stworzenia własnych Service

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
    labels:
      <service-label-key>: <service-label-value>
    name: <service-name>
    spec:
    type: ClusterIP
    ports:
    - port: <service-port>
      protocol: TCP
      targetPort: <application-port>
    selector:
      <pod-label-key>: <pod-label-value>
    ```

1. Sprawdź przyznane do Service adresy IP wewnątrz klastra
1. Utwórz kontener debug, którym sprawdzisz komunikację z Pod. Wyślij request za pomocą `curl` pod adres IP Service oraz pod jego nazwę DNS.
1. Upewnij się, że requesty wysyłane do Service dla aplikacji API obsługiwane są przez różne Pod należące do tego samego Deployment. Najłatwiejszym sposobem jest wysłanie kilku requestów na endpoint `/info` i porównanie wartości `hostname` w otrzymanej odpowiedzi.

    ```shell
    $ while true; do curl http://10.0.248.152:8888/info && echo && sleep 1; done

    {"status":"ok","timestamp":"2022-01-08T10:13:38.862Z","hostname":"api-66fd55f5c-gd6ns","data":null}
    {"status":"ok","timestamp":"2022-01-08T10:13:39.876Z","hostname":"api-66fd55f5c-gd6ns","data":null}
    {"status":"ok","timestamp":"2022-01-08T10:13:40.889Z","hostname":"api-66fd55f5c-z5ck5","data":null}
    {"status":"ok","timestamp":"2022-01-08T10:13:41.903Z","hostname":"api-66fd55f5c-z5ck5","data":null}
    {"status":"ok","timestamp":"2022-01-08T10:13:42.919Z","hostname":"api-66fd55f5c-bkrrm","data":null}
    {"status":"ok","timestamp":"2022-01-08T10:13:43.946Z","hostname":"api-66fd55f5c-gd6ns","data":null}
    {"status":"ok","timestamp":"2022-01-08T10:13:44.958Z","hostname":"api-66fd55f5c-gd6ns","data":null}
    {"status":"ok","timestamp":"2022-01-08T10:13:45.972Z","hostname":"api-66fd55f5c-gd6ns","data":null}
    {"status":"ok","timestamp":"2022-01-08T10:13:46.986Z","hostname":"api-66fd55f5c-z5ck5","data":null}
    ```

## Udostępnij aplikacje poza klaster

### Po co?

Chcąc, żeby aplikacja była dostępna poza klastrem dla użytkowników końcowych musisz odpowiednio skonfigurować Service

### Kroki

1. Zmień typ Service z `ClusterIP` na `LoadBalancer`
1. Sprawdź, że aplikacje dostępne są poza klastrem. Pamiętaj o użyciu odpowiednich portów podanych podczas konfiguracji Service.
1. Zaktualizuj zmienną środowiskową dla Podów z aplikacją Frontend aby wykonywała requesty pod publiczny adres IP Service dla aplikacji API 
1. Sprawdź czy Frontend wyświetla odpowiedzi z API

    ![img](./img/4-working-application.png)

### Podpowiedzi

1. Za równo Frontend jak i API powinny być udostępnione publicznie. Zadaniem aplikacji Frontend jest zwrócenie użytkownikowi plików `*.html`, `*.css` oraz `*.js`. Pliki `*.js` zawierają aplikacje React uruchamianą w przeglądarce użytkownika. Dane prezentowane w aplikacji React pobierane są z API. Oznacza to, że API musi być dostępne publicznie w sieci Internet (tak samo jak Frontend).
1. Aby udostępnić aplikacje publicznie skorzystaj z Service typu `LoadBalancer`. Service będą miały różne publiczne adresy IP.
1. Zmienne środowiskowe dla Pod możesz ustawić korzystając z opisanego tu podejścia: https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/#define-an-environment-variable-for-a-container
1. Service udostępnione są do sieci Internet przez HTTP. Upewni się, że wykonując requesty nie wykonujesz ich korzystając z HTTPS.
1. Gdybyś po ustawieniu zmiennych środowiskowych nie widział poprawnych odpowiedzi z API w aplikacji frontend. Wywołaj stronę w prywatnym oknie przeglądarki lub w spróbuj usunąć wszystkie Pod za pomocą `kubectl delete pod --all` (w efekcie Kubernetes powinien stworzyć nowe Pody).

---

# Praca domowa część 2

Witaj w drugiej części pracy domowej!

W tej części pracy domowej rozbudujesz aplikację z części 1. Wykorzystasz do tego poznane podczas szkolenia funkcjonalności Kubernetes.

Do aplikacji z pierwszej części pracy domowej został dodany Worker. Worker jest programem, który jest uruchamiany cyklicznie (domyślnie: codziennie o 5:00) w celu pobrania informacji o kursach walut z [API NBP](http://api.nbp.pl/api/exchangerates/tables/c?format=json) i zapisania ich w bazie danych.

## Wstęp

1. **Kod aplikacji**

    Katalogi z kodem aplikacji Frontend, API oraz Worker znajdują się w katalogu [./app/packages](./app/packages). W trakcie pracy domowej możesz zbudować swoje własne obrazy kontenerów i udostępnić je we własnych repozytoriach obrazów na Docker Hub.
  
    Mimo to, nie jest to krok wymagany :point_down:

1. **Obrazy kontenerów z aplikacjami**

    Obrazy kontenerów dla aplikacji Frontend, API oraz Worker znajdujące się w repozytoriach [macborowy/chmurobank-front:latest](), [macborowy/chmurobank-api:latest](), [macborowy/chmurobank-exchangerates-downloader:latest]() zostały zaktualizowane do najnowszej wersji.

    Jeśli nie chcesz budować kontenerów samodzielnie - skorzystaj z gotowych. Są one zbudowane z tej samej wersji kodu co dostępna w katalogu [./app](./app)


---

## Zadanie 1

### 1. Podziel rozwiązanie logicznie na Namespace

#### Po co?

Wraz z biegiem czasu na klastrze może pojawić się coraz więcej obiektów. Dobrym pomysłem jest ich logiczne grupowanie np. według komponentów aplikacji lub domen biznesowych. Obiektem w Kubernetes, który pozwala na logiczne grupowanie jest [Namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/).

#### Kroki

1. Stwórz dwa namespace:

  - `frontend`, w którym udostępnisz obiekty związane z aplikacją Frontend
  - `backend`, w którym udostępnisz obiekty związane z aplikacjami API oraz Worker

#### Podpowiedzi

Sposób tworzenia Namespace i umieszczania w nich komponentów K8s został przedstawiony w ćwiczeniu poświęconym [Namespace (link)](https://github.com/cloudstateu/otwarte-szkolenie-k8s/tree/main/labs/Kubernetes/06_namespace).

Jeśli na klastrze posiadasz zasoby, które są już utworzone w Namespace `default` usuń je za pomocą `kubectl delete` i utwórz na nowo we właściwym Namespace.


### 2. Wykorzystanie Ingress do udostępnienia aplikacji poprzez jeden publiczny adres IP

#### Po co?

W pracy domowej #1 aplikacja udostępniona była na dwóch różnych adresach IP. Takie rozwiązanie nie jest pożądane, ponieważ udostępnienie aplikacji w sieci Internet będzie wymagało więcej prac operacyjnych (np. dla każdej kolejnej aplikacji, która powinna być dostępna poza klastrem musimy dodawać kolejne wpisy w DNS).

W sytuacji gdy chcemy udostępnić poza klaster kilka komponentów wchodzących w skład pojedynczej aplikacji powinniśmy zrobić to na jednym adresie IP. Taki mechanizm dostarcza nam [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) oraz [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).

#### Kroki

1. Zainstaluj ingress-nginx na klastrze
1. Stwórz dwa pliki YAML z konfiguracją obiektu Ingress dla aplikacji w obu Namespace. Wdróż konfigurację na klaster.
1. Sprawdź czy aplikacje odpowiadają na skonfigurowanych przez Ciebie endpoint.

#### Podpowiedzi

Wykorzystaj Ingress Controller [ingress-nginx](https://kubernetes.github.io/ingress-nginx/) w celu udostępnienia aplikacji React oraz API poprzez jeden publiczny adres IP. 
Jeśli podzieliłeś rozwiązanie na dwa Namespace powinieneś stworzyć dwa pliki YAML z konfiguracją obiektu Ingress dla usług w odpowiednich Namespace. Podział konfiguracji obiektu Ingress jest znaną i rekomendowaną praktyką konfigurowania Ingress dla usług w wielu Namespace.

Sposób instalacji i konfiguracji Ingress został przedstawiony w ćwiczeniu poświęconym **Ingress** - [Link do ćwiczenia](https://github.com/cloudstateu/otwarte-szkolenie-k8s/tree/main/labs/Kubernetes/13_ingress). Zauważ, że obie aplikacje znajdują się w jednym Namespace, dlatego w ćwiczeniu stworzono tylko jedną konfigurację obiektu Ingress. 

### 3. Stworzenie bazy danych

#### Po co?

Baza danych jest koniecznym elementem tworzonego systemu. Worker zapisuje zebrane dane do bazy danych. API z kolei został zaktualizowany by odczytywać dane z bazy danych.

#### Kroki

1. Utwórz serwer bazy danych w usłudze Azure Database for Postgres
1. Połącz się z serwerem bazy danych (np. przez `psql` z Cloud Shell)
1. Po podłączeniu się do serwera Postgres stwórz nową bazę danych oraz tabelę w której przechowasz dane pobrane z API NBP:

 - Stwórz bazę danych `bank`

    ```bash
    CREATE DATABASE bank;
    ```

- Połącz się z utworzoną bazą danych:

    ```
    \c bank
    ```

- Utwórz nową tabelę `exchangerates`

    ```
    CREATE TABLE exchangerates (
      pid SERIAL PRIMARY KEY,
      currency_from varchar(3),
      currency_to varchar(3),
      buy numeric,
      sell numeric,
      effective_date date,
      created_on timestamp DEFAULT CURRENT_TIMESTAMP
    );
    ```

#### Podpowiedzi

Sposób tworzenia usługi Azure Database for Postgres oraz podłączenia do serwera Postgres przedstawiony w ćwiczeniu poświęconym **integracji z bazą danych** :point_right: [link do ćwiczenia](https://github.com/cloudstateu/otwarte-szkolenie-k8s/tree/main/labs/Kubernetes/16_connect_to_azure_database).

### 4. Udostępnij Worker na klastrze za pomocą CronJob

#### Po co?

Worker w tworzonej aplikacji zajmuje się pobraniem danych o kursach walut z API NBP. Worker jest zaimplementowany jako [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/), co oznacza, że będzie uruchomiony na klastrze zgodnie z harmonogramem opisanym w konfiguracji obiektu CronJob.

#### Kroki

1. Otwórz plik [./files/4-worker-cronjob.yaml](./files/4-worker-cronjob.yaml) i uzupełnij wartości ConfigMap
1. Wdróż ConfigMap oraz Worker na klaster
1. Za pomocą debuggowego Pod (z Postgres) sprawdź czy Worker pobiera i zapisuje dane z API NBP do bazy danych.

#### Podpowiedzi

Po wdrożeniu Worker na klaster, sprawdź czy pobiera dane z API NBP i wstawia je do bazy. W razie potrzeby zmień harmonogram uruchomienia CronJob (na np. `* * * * *`, czyli uruchomienie Job co minutę).

CronJob powinien korzystać z ConfigMapy, które będzie zawierała informacje pozwalające połączyć się do bazy danych Postgres.

### 5. Podłączenie Key Vault w celu przechowywania w nim sekretów do bazy danych

#### Po co?

Usługa Key Vault pozowli na bezpieczne przechowywanie sekretów do bazy danych, które będą potrzebne aplikacji do łączenia się z bazą danych (API oraz Worker). W momencie uruchomienia kontenera wybrane sekrety z Azure Key Vault zostaną wstrzyknięte do kontenera.

#### Kroki

1. Stwórz instancję usługi Azure Key Vault w Azure Portal
1. Utwórz sekrety w Azure Key Vault z odpowiednimi wartościami dla Twojego środowiska:

    ```bash
    PGDATABASE: bank
    PGHOST: <hostname bazy danych w Azure>
    PGPASSWORD: <haslo użytkownika do bazy danych>
    PGPORT: "5432"
    PGUSER: <użytkownik do bazy danych>
    ```

1. Zainstaluj Secrets Store CSI Driver na klastrze:

    ```bash
    helm install csi csi-secrets-store-provider-azure/csi-secrets-store-provider-azure -n csi --set secrets-store-csi-driver.syncSecret.enabled=true
    ```

1. Uzupełnij plik [files/5-keyvault.yaml](files/5-keyvault.yaml) odpowiednimi wartościami i wdróż go na klaster:

    - `userAssignedIdentityID` - ID Managed Identity, którym przedstawia się AKS do innych usług
    - `keyvaultName` - nazwa utworzonego Key Vault
    - `tenantId`

    Informację jak zdobyć te wartości znajdziesz w ćwiczeniu [Key Valut (ćwiczenie)](https://github.com/cloudstateu/otwarte-szkolenie-k8s/tree/main/labs/Kubernetes/17_azure_key_vault_with_azure_database).

1. Stwórz Pod do debugging i sprawdź czy sekrety są pobierane z Key Vault

#### Podpowiedzi

Sposób tworzenia i wykorzystania Key Vault przedstawiono w ćwiczeniu poświęconym [Key Valut (ćwiczenie)](https://github.com/cloudstateu/otwarte-szkolenie-k8s/tree/main/labs/Kubernetes/17_azure_key_vault_with_azure_database).

W razie problemów z pobieraniem sekretów z Azure Key Vault do Pod możesz spróbować zdiagnozować problem za pomocą `kubectl describe pod`. Informacje zawarte w sekcji _"Events"_ z pewnością nakierują Cię gdzie szukać problemu.

**Uwaga**: Przedstawiony w ćwiczeniu YAML dla `SecretProviderClass` pozwalał na wstrzykiwanie sekretów do Podów jako Volumes. W tym ćwiczeniu użyjesz mechanizmu wstrzykiwania sekretów przez zmienne środowiskowe. Aby zmienić mechanizm użyj specjalnie przygotowanego YAML z `SecretProviderClass`, który znajdziesz w pliku [./files/5-keyvault.yaml](./files/5-keyvault.yaml).

**Uwaga**: Instalując Key Vault za pomocą Helm upewnij się, że przeciążysz jedną z domyślnych wartości Helm Chart. Ustaw wartość `secrets-store-csi-driver.syncSecret.enabled` na `true` lub wykorzystaj poniższą komendę do instalacji Azure csi-secrets-store-provider-azure:

```bash
helm install csi csi-secrets-store-provider-azure/csi-secrets-store-provider-azure -n csi --set secrets-store-csi-driver.syncSecret.enabled=true
```

### 6. Podłączenie API do bazy danych, tak aby odczytywać informacje o kursach walut

W tym momencie Worker jest usługą odpowiedzialną za pobranie informacji o kursach walut z API NBP. W związku z tym pierwotny kod usługi API został zmieniony, żeby odczytywać dane o kursach walut z bazy danych.

Zaktualizuj Deployment usługi API i sprawdź czy nadal poprawnie wyświetla dane. Do aktualizacji użyj [pliku YAML (`./files/6-deployment-api.yaml`)](./files/6-deployment-api.yaml).

**Uwaga**: Usługa API w nowej wersji łączy się z bazą danych. Do połączenia z bazą danych wymaga podania wartości uwierzytelniających przez zmienne środowiskowe. Przygotowany [plik YAML (`./files/6-deployment-api.yaml`)](./files/6-deployment-api.yaml) został przygotowany, aby odczytać wartości z Secret o nazwie `db-secrets` i wstawiać jego wartości do zmiennych środowiskowych w uruchamianych kontenerach. Secret `db-secrets` jest tworzony przez _initContainer_ `sync-secrets` na podstawie konfiguracji `SecretProviderClass`, którą znajdziesz w pliku [./files/5-keyvault.yaml](./files/5-keyvault.yaml).

**Uwaga**: W pliku [./files/6-worker-cronjob-secrets.yaml](./files/6-worker-cronjob-secrets.yaml) znajduje się zaktualizowany kod CronJob dla Worker, który również wykorzystuje wstrzykiwanie informacji uwerzytelniających do bazy danych przez Secret. Zaktualizuj CronJob dla Worker i sprawdź czy nadal działa poprawnie i zapisuje informacje do bazy danych.

#### Kroki

1. Zaktualizuj Deployment usługi API za pomocą pliku [./files/6-deployment-api.yaml](./files/6-deployment-api.yaml)
1. Sprawdź czy API nadal poprawnie zwraca dane
1. Usuń ConfigMap z nazwą `cm-azure-database-connection-details`. ConfigMap znajduje się w Namespace `backend`
1. Zaktualizuj CronJob za pomocą pliku [./files/6-worker-cronjob-secrets.yaml](./files/6-worker-cronjob-secrets.yaml)
1. Sprawdź czy Worker nadal poprawnie wstawia dane do tabeli `exchangerates`

### 7. Skomunikuj ze sobą Frontend i API

#### Po co?

Do tego momentu aplikacja Frontend nie pobierała danych z aplikacji API, ponieważ nigdzie nie skonfigurowaliśmy zmiennej środowiskowej `API_URL` w kontenerze z aplikacją Frontend. Czas to zmienić:

#### Kroki

1. Skonfiguruj Deployment aplikacji Frontend. Dodaj do kontenera z aplikacją Frontend wstrzyknięcie zmiennej środowiskowej `API_URL`. Po skonfigurowaniu Deployment Frontend wdróż go na klaster.
1. Sprawdź czy aplikacja uruchomiona w przeglądarce prezentuje dane o kursach walut

### 8. Wykorzystanie liveness i readiness probe

W aplikacji API zaimplementowano odpowiednie endpointy, do badania stanu życia aplikacji. Są to:

  - dla Liveness Probe: `/health`
  - dla Readiness Probe: `/ready`

Zadaniem endpoint `/health` jest zwrócenie odpowiedzi 200 OK. Zakładamy, że jeśli aplikacja ulegnie jakiejkolwiek awarii lub deadlock i nie będzie wstanie zwrócić odpowiedzi wykonując tak prosty kod to z pewnością nie działa ona poprawnie i wymaga restartu.

Zadaniem endpoint `/ready` jest sprawdzenie czy wszystkie zależności aplikacji są gotowe do pracy. W przypadku aplikacji API istnieje jedna zależność - baza danych. Aplikacja API wymaga, żeby baza danych była wstanie odpowiedzieć na zapytania. Jeśli baza nie będzie odpowiadała to aplikacja API nie zrealizuje swojego zadania, czyli nie zwróci informacji o kursach walut dla potencjalnego klienta.

Wykorzystaj mechanizm Probe w celu automatycznego sprawdzenia czy aplikacje działają poprawnie. Obraz aplikacji [macborowy/chmurobank-api:latest](https://hub.docker.com/r/macborowy/chmurobank-api) zawiera już kod aplikacji z zaimplementowanymi endpoint.

Sposób tworzenia i wykorzystania Liveness Probe został przedstawiony w ćwiczeniu poświęconym **Liveness Probe** - [Link do ćwiczenia](https://github.com/cloudstateu/otwarte-szkolenie-k8s/tree/main/labs/Kubernetes/12_probes). Natomiast informację dotyczące tworzenia Readiness Probe  znadują się w dokumentacji Kubernetes - [Link do dokumentacji dotyczącej Readiness Probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-readiness-probes)

W celu przetestowania działania Readiness Probe możesz wyłączyć bazę danych w Azure. W ten sposób proste zapytania do bazy danych, które jest wykonywane przez endpoint `/ready` nie powiedzie się i Readiness Probe powinien zwrócić porażkę i w ostatczności odłączyć repliki od Service i w ten sposób nie kierować ruchu do procesów, które nie są w stanie go obsłużyć.

<br><br>

<center><p>&copy; 2022 Chmurowisko Sp. z o.o.<p></center>
