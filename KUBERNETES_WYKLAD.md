# Aplikacja budżetu domowego w Kubernetes — kompletny wykład

Niniejszy dokument jest szczegółowym opisem implementacji aplikacji webowej do zarządzania budżetem domowym uruchomionej w lokalnym klastrze Kubernetes. Jego celem jest wyjaśnienie każdej decyzji architektonicznej, każdego zasobu Kubernetes i każdego mechanizmu działającego w projekcie — tak, aby po przeczytaniu można było nie tylko uruchomić aplikację, ale przede wszystkim rozumieć, dlaczego jest zbudowana tak, a nie inaczej.

---

## Rozdział 1. Aplikacja i jej architektura

### 1.1. Co robi aplikacja

Aplikacja pozwala użytkownikowi prowadzić prosty budżet domowy. Użytkownik może dodawać transakcje (przychody i wydatki), przypisywać je do kategorii, przeglądać miesięczny bilans, kontrolować limity wydatków dla kategorii oraz usuwać lub edytować wpisy.

Po uruchomieniu aplikacja zawiera dane demonstracyjne: wynagrodzenie, czynsz, zakupy spożywcze i bilet miesięczny z czerwca 2026 roku. Dzięki temu panel aplikacji po starcie pokazuje sensowne dane bez ręcznego wprowadzania czegokolwiek.

### 1.2. Trójwarstwowa architektura

Aplikacja jest zbudowana zgodnie z klasycznym wzorcem trójwarstwowym:

```
warstwa prezentacji   → Frontend (TypeScript + Nginx)
warstwa logiki        → Backend (Symfony + PHP-FPM + Nginx)
warstwa danych        → Baza danych (MySQL 8.0)
```

Każda warstwa jest fizycznie oddzielona: działa w osobnych kontenerach, jest zbudowana z osobnego obrazu Docker i uruchomiona jako osobny zasób Kubernetes. Warstwy komunikują się wyłącznie przez zdefiniowane interfejsy — frontend nie ma bezpośredniego dostępu do bazy danych.

### 1.3. Technologie

- **Backend**: PHP 8.3, Symfony 7.2, Doctrine ORM i Doctrine Migrations. Backend udostępnia API REST pod ścieżką `/api`. Jest uruchomiony jako PHP-FPM (FastCGI Process Manager), który nie obsługuje HTTP samodzielnie — do tego służy Nginx.
- **Frontend**: TypeScript bez ciężkiego frameworka, skompilowany do plików statycznych i serwowany przez Nginx. Adres backendu nie jest wkompilowany w pliki — jest wstrzykiwany w runtime.
- **Baza danych**: MySQL 8.0, uruchomiona jako kontener Kubernetes z trwałym wolumenem.
- **Infrastruktura**: Docker, minikube, kubectl, Kustomize, Helm, NGINX Ingress Controller.

---

## Rozdział 2. Kontenery i obrazy Docker

Zanim cokolwiek trafi do Kubernetes, każdy komponent musi stać się obrazem Docker. Projekt buduje trzy obrazy.

### 2.1. Obraz backendu PHP — `budget-backend-php:latest`

Backend używa wieloetapowego Dockerfile (`Backend/Dockerfile`). Wieloetapowe budowanie (multi-stage build) pozwala rozdzielić etap kompilacji od finalnego obrazu — w końcowym obrazie nie ma narzędzi deweloperskich, zależności build-time ani plików tymczasowych.

Docelowy etap `app_php_prod` zawiera:
- PHP-FPM z rozszerzeniami wymaganymi przez Symfony i Doctrine,
- skompilowane zależności composera bez pakietów deweloperskich,
- kod aplikacji.

Kontener uruchamia PHP-FPM nasłuchujący na porcie 9000. PHP-FPM nie jest serwerem HTTP — nie obsługuje połączeń HTTP, tylko protokół FastCGI. Żeby przeglądarka mogła się połączyć z backendem, potrzebny jest Nginx jako warstwa pośrednia.

### 2.2. Obraz backendu Nginx — `budget-backend-nginx:latest`

Drugi obraz backendu zawiera Nginx skonfigurowany jako odwrotny serwer proxy dla PHP-FPM. Konfiguracja (`Backend/docker/nginx/conf.d/default.conf`) przyjmuje żądania HTTP na porcie 80, a dla plików PHP przekazuje je do PHP-FPM przez protokół FastCGI pod adres `php:9000`.

`php` to nazwa Kubernetes Service, który w klastrze rozwiązuje się do poda backend-php. Nginx backendu „nie wie", że działa w Kubernetes — widzi tylko nazwę hosta `php`, która działa dzięki wewnętrznemu DNS klastra.

### 2.3. Obraz frontendu — `budget-frontend:latest`

Frontend też używa wieloetapowego Dockerfile (`Frontend/Dockerfile`):

1. Etap `build`: Node.js 24 kompiluje TypeScript do plików statycznych (`dist/`).
2. Etap finalny: Nginx 1.27 serwuje skompilowane pliki z `/usr/share/nginx/html`.

Do kontenera kopiowane są dwa dodatkowe pliki: `config.template.js` (szablon konfiguracji runtime) i `docker-entrypoint.sh` (skrypt startowy). Skrypt startowy uruchamia się przed Nginx i generuje plik `config.js` przez podstawienie zmiennych środowiskowych (`envsubst`):

```sh
envsubst '${API_BASE_URL} ${REQUEST_TIMEOUT_MS}' \
  < /usr/share/nginx/html/config.template.js \
  > /usr/share/nginx/html/config.js
```

Dzięki temu frontend w Kubernetes ma `API_BASE_URL=/api` (ścieżka względna), a ten sam obraz uruchomiony lokalnie może dostać `API_BASE_URL=http://localhost:8000/api`. Adres backendu nie jest wkompilowany w JavaScript — jest wstrzykiwany przez infrastrukturę.

### 2.4. Budowanie obrazów w kontekście minikube

Minikube uruchamia klaster w środku izolowanego środowiska (na macOS z Docker driverem: wewnątrz kontenera Docker). Ten klaster ma własny daemon Docker, oddzielony od systemowego. Żeby Kubernetes mógł uruchomić lokalnie zbudowane obrazy bez zewnętrznego rejestru, obrazy muszą być zbudowane w kontekście daeomna minikube:

```bash
eval $(minikube docker-env)
docker build -t budget-backend-php:latest --target app_php_prod ./Backend
docker build -t budget-backend-nginx:latest --target app_nginx ./Backend
docker build -t budget-frontend:latest ./Frontend
```

`eval $(minikube docker-env)` ustawia zmienne środowiskowe (`DOCKER_HOST`, `DOCKER_TLS_VERIFY` itd.), które kierują klienta Docker na daemon minikube zamiast systemowego. Po tym poleceniu `docker build` buduje obraz bezpośrednio w przestrzeni minikube.

### 2.5. imagePullPolicy: IfNotPresent

Wszystkie deploymenty i joba mają ustawione `imagePullPolicy: IfNotPresent`. Oznacza to, że Kubernetes użyje istniejącego lokalnego obrazu zamiast próbować pobrać go z Docker Hub. To kluczowe ustawienie dla lokalnych obrazów bez tagu wersji — bez niego Kubernetes próbowałby pobrać obraz z zewnętrznego rejestru i kończyłby z błędem `ErrImagePull`.

---

## Rozdział 3. Namespace — izolacja zasobów

### 3.1. Po co namespace

Kubernetes Namespace to mechanizm logicznego podziału klastra. Zasoby w jednym namespace są oddzielone od zasobów w innym — mają osobne nazwy DNS, osobne polityki RBAC i osobne limity zasobów.

W projekcie wszystkie zasoby aplikacji istnieją w namespace `budget-app`. Dzięki temu:
- polecenia `kubectl -n budget-app get pods` pokazują tylko pody aplikacji,
- można usunąć całe środowisko jednym poleceniem `kubectl delete namespace budget-app`,
- RBAC można skonfigurować na poziomie jednego namespace bez wpływu na inne aplikacje w klastrze.

### 3.2. Definicja namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: budget-app
```

Namespace jest najprostszym zasobem Kubernetes. Nie ma specyfikacji — tylko metadane z nazwą. Wszystkie inne zasoby projektu w polu `metadata.namespace` mają wartość `budget-app`.

---

## Rozdział 4. Konfiguracja — ConfigMap i Secret

### 4.1. Zasada rozdziału konfiguracji od kodu

Dobrze zaprojektowana aplikacja kontenerowa nie ma konfiguracji wkompilowanej w obraz. Konfiguracja (adresy, hasła, tryb działania) jest wstrzykiwana z zewnątrz przez zmienne środowiskowe lub pliki. W Kubernetes do tego służą dwa zasoby: ConfigMap i Secret.

### 4.2. ConfigMap — konfiguracja jawna

ConfigMap przechowuje dane konfiguracyjne, które nie są wrażliwe. W projekcie ConfigMap `backend-config` zawiera:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: budget-app
data:
  APP_ENV: prod
  APP_DEBUG: "0"
  PHP_DATE_TIMEZONE: Europe/Warsaw
  CORS_ALLOW_ORIGIN: '^https?://(localhost|127\.0\.0\.1|budget\.local)(:[0-9]+)?$'
  MAILER_DSN: "null://null"
  SENTRY_DSN: ""
```

- `APP_ENV: prod` — uruchamia Symfony w trybie produkcyjnym (bez debug toolbar, z cache).
- `APP_DEBUG: "0"` — wyłącza debugowanie.
- `PHP_DATE_TIMEZONE: Europe/Warsaw` — ustawia strefę czasową dla PHP.
- `CORS_ALLOW_ORIGIN` — regex dopuszczający żądania z frontendu lokalnego i przez Ingress.
- `MAILER_DSN: "null://null"` — wyłącza wysyłkę maili (projekt ich nie używa).
- `SENTRY_DSN: ""` — puste, integracja Sentry nieaktywna.

### 4.3. Secret — dane wrażliwe

Secret przechowuje dane, których nie powinno być widać w logach, repozytoriach ani listingach zasobów. Kubernetes przechowuje wartości Secret zakodowane w Base64 (nie zaszyfrowane, ale oddzielone od zwykłej konfiguracji i możliwe do objęcia osobnymi politykami RBAC i szyfrowania w etcd).

Projekt ma trzy Secrety:

**`mysql-secret`** zawiera dane połączenia z bazą:

```yaml
stringData:
  MYSQL_DATABASE: budget
  MYSQL_USER: budget
  MYSQL_PASSWORD: budget_password
  MYSQL_ROOT_PASSWORD: root_password
  DATABASE_URL: mysql://budget:budget_password@mysql:3306/budget?serverVersion=8.0.32&charset=utf8mb4
```

`DATABASE_URL` to gotowy DSN dla Symfony/Doctrine. Adres hosta to `mysql` — nazwa Kubernetes Service, która rozwiązuje się do poda MySQL wewnątrz klastra.

**`backend-secret`** zawiera `APP_SECRET` Symfony:

```yaml
stringData:
  APP_SECRET: budget-demo-secret-change-me
```

`APP_SECRET` to klucz kryptograficzny Symfony używany do podpisywania tokenów CSRF, szyfrowania ciasteczek i innych mechanizmów frameworka.

**`budget-basic-auth`** zawiera wpis htpasswd dla NGINX Ingress Controller:

```yaml
stringData:
  auth: 'budget-admin:$apr1$XBFEc3xa$ZfG1lFjwCUETb6HP0loqi0'
```

Format `htpasswd` zawiera nazwę użytkownika i zahaszowane hasło (algorytm MD5-APR1). Ingress Controller odczytuje ten Secret i wymaga podania poprawnych danych przed przepuszczeniem ruchu do aplikacji.

### 4.4. Różnica między ConfigMap a Secret

ConfigMap jest przeznaczony do jawnej konfiguracji — wartości, które można bez obaw wyświetlić. Secret jest przeznaczony do haseł, kluczy i danych wrażliwych. Kubernetes traktuje je inaczej:
- Wartości Secret są zakodowane w Base64 w etcd.
- Można skonfigurować szyfrowanie w spoczynku (encryption at rest) dla Secretów, ale nie dla ConfigMap.
- RBAC pozwala nadać różne uprawnienia do odczytu ConfigMap i Secret.
- `kubectl get configmap -o yaml` pokaże wartości wprost; `kubectl get secret -o yaml` pokaże Base64, ale nie ukryje treści przed osobą mającą dostęp do kubectl.

### 4.5. Wstrzykiwanie konfiguracji do kontenerów

Wartości z ConfigMap i Secret trafiają do kontenerów przez `envFrom`:

```yaml
envFrom:
  - configMapRef:
      name: backend-config
  - secretRef:
      name: mysql-secret
  - secretRef:
      name: backend-secret
```

`envFrom` wstrzykuje wszystkie klucze z danego zasobu jako zmienne środowiskowe. Kontener PHP widzi `APP_ENV`, `DATABASE_URL`, `APP_SECRET` i wszystkie pozostałe wartości tak, jakby były ustawione ręcznie.

---

## Rozdział 5. Baza danych — StatefulSet i PersistentVolumeClaim

### 5.1. Dlaczego baza danych nie jest zwykłym Deploymentem

Deployment w Kubernetes jest przeznaczony dla aplikacji bezstanowych. Pody zarządzane przez Deployment mogą być tworzone, usuwane i zastępowane w dowolnej kolejności. Jeśli pod zostanie usunięty i Kubernetes stworzy nowy, nowy pod nie pamięta nic z poprzedniego.

Baza danych jest aplikacją stanową. Przechowuje dane na dysku. Jeśli pod MySQL zostałby usunięty i zastąpiony nowym, nowy pod zacząłby z pustą bazą — chyba że dane są przechowywane poza efemerycznym systemem plików kontenera.

StatefulSet rozwiązuje ten problem w kilku wymiarach:
1. **Stabilne nazwy**: pody StatefulSet mają przewidywalne nazwy (`mysql-0`, `mysql-1`...) zamiast losowych sufiksów jak w Deployment.
2. **Stabilna tożsamość sieciowa**: każdy pod StatefulSet ma stabilną nazwę DNS przez headless Service.
3. **Trwałe wolumeny**: StatefulSet może automatycznie tworzyć PersistentVolumeClaim dla każdego poda, który przeżywa usunięcie poda.

### 5.2. PersistentVolumeClaim

PersistentVolumeClaim (PVC) to żądanie przydzielenia trwałego miejsca na dysku. Kubernetes (a konkretnie StorageClass) realizuje to żądanie przez PersistentVolume (PV) — konkretny kawałek dysku.

W projekcie PVC jest tworzony automatycznie przez `volumeClaimTemplates` w StatefulSet:

```yaml
volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```

- `accessModes: ReadWriteOnce` — wolumen może być montowany do zapisu przez jeden węzeł na raz. Dla bazy danych działającej jako jedna replika to wystarczające i właściwe.
- `storage: 1Gi` — żądamy jednego gigabajta miejsca.

PVC `mysql-data` jest montowany w kontenerze MySQL pod ścieżką `/var/lib/mysql`, gdzie MySQL przechowuje pliki bazy danych. Nawet po usunięciu poda i ponownym uruchomieniu StatefulSet dane pozostają na tym wolumenie.

### 5.3. Headless Service dla StatefulSet

StatefulSet wymaga tzw. Headless Service — Service bez wirtualnego adresu IP (z `clusterIP: None`). Headless Service nie load-balancuje ruchu ani nie ma jednego adresu; zamiast tego DNS klastra zwraca bezpośrednio adresy IP podów.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
  namespace: budget-app
spec:
  clusterIP: None
  ports:
    - name: mysql
      port: 3306
      targetPort: 3306
  selector:
    app: mysql
```

Dzięki headless Service pod StatefulSet otrzymuje stabilną nazwę DNS: `mysql-0.mysql-headless.budget-app.svc.cluster.local`. Ta nazwa jest używana przez samego MySQL do komunikacji w klastrze (przydatne przy replikacji master-slave, choć projekt nie konfiguruje replikacji).

Obok headless Service istnieje zwykły Service `mysql`, który daje aplikacji (Symfony) stabilną nazwę `mysql:3306` bez konieczności znania numeru poda. Backend używa `mysql` jako hosta w `DATABASE_URL`.

### 5.4. StatefulSet — pełna specyfikacja

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: budget-app
spec:
  serviceName: mysql-headless
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3306
              name: mysql
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 1Gi
          envFrom:
            - secretRef:
                name: mysql-secret
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

- `serviceName: mysql-headless` — wiąże StatefulSet z headless Service.
- `replicas: 1` — jedna instancja MySQL. Replikacja MySQL wymaga dodatkowej konfiguracji (np. MySQL InnoDB Cluster), która wykracza poza zakres projektu.
- Kontener MySQL dostaje zmienne środowiskowe z `mysql-secret`: `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_ROOT_PASSWORD`. Obraz MySQL przy pierwszym uruchomieniu tworzy bazę i użytkownika na podstawie tych zmiennych.

---

## Rozdział 6. Backend — Deployment, Service i komunikacja PHP-FPM

### 6.1. PHP-FPM i Nginx jako para kontenerów

Backend Symfony nie jest jednym kontenerem — to dwa osobne kontenery działające jako dwa osobne pody z dwoma osobnymi Deploymentami, połączone Kubernetes Service.

Ta separacja wynika z podziału odpowiedzialności:
- **backend-php** (PHP-FPM): wykonuje kod PHP, obsługuje logikę biznesową, komunikuje się z bazą danych. Nasłuchuje na porcie 9000, ale tylko na protokół FastCGI — nie HTTP.
- **backend-nginx**: przyjmuje żądania HTTP, serwuje pliki statyczne z katalogu `public/` Symfony, a dla plików `.php` przekazuje żądania przez FastCGI do `php:9000`.

### 6.2. Deployment backend-php i Service php

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-php
  namespace: budget-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend-php
  template:
    metadata:
      labels:
        app: backend-php
    spec:
      containers:
        - name: php
          image: budget-backend-php:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 9000
              name: fpm
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          envFrom:
            - configMapRef:
                name: backend-config
            - secretRef:
                name: mysql-secret
            - secretRef:
                name: backend-secret
```

Service `php` daje backendowemu Nginx stabilny adres:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: php
  namespace: budget-app
spec:
  ports:
    - name: fpm
      port: 9000
      targetPort: 9000
  selector:
    app: backend-php
```

Dzięki temu Nginx backendu może używać `php:9000` jako adresu PHP-FPM, niezależnie od tego, na którym węźle działa pod i jaki ma adres IP.

### 6.3. Deployment backend-nginx i Service backend

Nginx backendu dostaje własny Deployment i Service `backend`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: budget-app
spec:
  ports:
    - name: http
      port: 80
      targetPort: 80
  selector:
    app: backend-nginx
```

Service `backend` jest tym, na co wskazuje Ingress dla ścieżek `/api` i `/api-platform`. Kiedy żądanie trafia przez Ingress do ścieżki `/api/transactions`, Ingress przekazuje je do Service `backend` na port 80, a Service przekazuje je do jednego z podów backend-nginx.

### 6.4. Dlaczego backend-nginx i backend-php są osobnymi Deploymentami?

Można by uruchomić Nginx i PHP-FPM jako dwa kontenery w jednym podzie (sidecar pattern). Projekt celowo tego nie robi — każdy komponent to osobny Deployment. Powody:
- Niezależne skalowanie: można dodać więcej replik PHP-FPM bez zmiany liczby Nginx i odwrotnie.
- Osobne restarty: crash PHP-FPM nie restartu Nginx.
- Czystszy model: każdy Deployment ma jedną odpowiedzialność.

---

## Rozdział 7. Frontend — Deployment, Service i runtime config

### 7.1. Deployment frontend

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: budget-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: budget-frontend:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
              name: http
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi
          env:
            - name: API_BASE_URL
              value: /api
            - name: REQUEST_TIMEOUT_MS
              value: "8000"
```

Zmienne `API_BASE_URL=/api` i `REQUEST_TIMEOUT_MS=8000` są wstrzyknięte do kontenera. Skrypt startowy (`docker-entrypoint.sh`) użyje ich do wygenerowania `config.js`, który frontend załaduje w przeglądarce jako pierwszy skrypt.

### 7.2. Frontend jako reverse proxy dla /api

Konfiguracja Nginx frontendu (`Frontend/nginx/default.conf`) zawiera coś ważnego:

```nginx
location /api/ {
    proxy_pass http://backend/api/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}

location / {
    try_files $uri $uri/ /index.html;
}
```

Frontend Nginx nie tylko serwuje pliki statyczne — przekazuje też żądania `/api/` do Service `backend`. Dzięki temu dostęp przez `make port-forward` (który łączy się bezpośrednio z Service `frontend`) działa kompletnie: frontend dostaje pliki HTML/CSS/JS, a JavaScript wywołujący `/api/transactions` trafia do backendu przez Nginx frontendu.

W trybie Ingress to znaczenie jest mniejsze — Ingress sam rozdziela `/api` do backendu i `/` do frontendu — ale dla port-forward to kluczowy mechanizm.

### 7.3. API_BASE_URL = /api — dlaczego ścieżka względna?

Użycie `/api` zamiast `http://backend.local/api` sprawia, że frontend zawsze wysyła żądania do tego samego hosta, z którego jest serwowany. W środowisku Kubernetes:
- przez port-forward: `http://127.0.0.1:18081/api/...` → frontend Nginx proxy → backend
- przez Ingress: `http://budget.local/api/...` → Ingress → backend

Ten sam obraz Docker działa w obu scenariuszach bez zmian.

---

## Rozdział 8. Migracje bazy danych — Kubernetes Job

### 8.1. Po co Job zamiast skryptu startowego

Migracje Doctrine tworzą tabele, wstawiają kategorie i dane demonstracyjne. To operacja jednorazowa: powinna wykonać się raz, zakończyć sukcesem i nie działać permanentnie.

Alternatywy i ich wady:
- **Skrypt w kontenerze PHP przy starcie**: jeśli pod się zrestartuje, migracje uruchomią się ponownie. `doctrine:migrations:migrate --no-interaction` jest idempotentne (nie robi nic, jeśli nie ma nowych migracji), ale restartowanie bazy i uruchamianie migracji w każdym restarcie jest niepotrzebnym ryzykiem i spowalnia start.
- **InitContainer**: uruchamia się przed głównym kontenerem, ale jest powiązany z cyklem życia poda. Przy każdym restarcie poda init container uruchamia się ponownie.

**Job** jest właściwą abstrakcją: Kubernetes gwarantuje, że Job wykona się do skutku (zadana liczba udanych uruchomień), a potem przestaje działać. Zakończony Job pozostaje w klastrze do ręcznego usunięcia lub automatycznego sprzątania.

### 8.2. Definicja Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backend-migrations
  namespace: budget-app
spec:
  backoffLimit: 10
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrations
          image: budget-backend-php:latest
          imagePullPolicy: IfNotPresent
          command:
            - sh
            - -c
            - bin/console doctrine:migrations:migrate --no-interaction
          resources:
            requests:
              cpu: 50m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          envFrom:
            - configMapRef:
                name: backend-config
            - secretRef:
                name: mysql-secret
            - secretRef:
                name: backend-secret
```

- `backoffLimit: 10` — Kubernetes może ponowić pod Joba do 10 razy przy błędach. To ważne, bo MySQL może nie być jeszcze gotowy w momencie startu Joba. Przy `backoffLimit: 10` Job będzie próbował ponownie, aż MySQL uruchomi się i migracje zakończą sukcesem.
- `restartPolicy: OnFailure` — pod jest restartowany tylko przy błędzie (nie zawsze). Opcja `Never` spowodowałaby tworzenie nowych podów zamiast restartowania, co prowadziłoby do wielu nieudanych podów zajmujących zasoby.
- Job używa tego samego obrazu co backend-php, bo ten obraz zawiera plik `bin/console` Symfony i wszystkie klasy migracji.

---

## Rozdział 9. Ingress — zewnętrzny dostęp HTTP

### 9.1. Problem: jak użytkownik trafia do aplikacji?

Wszystkie Services w projekcie są typu ClusterIP — mają adresy IP widoczne tylko wewnątrz klastra. Użytkownik nie może się z nimi połączyć bezpośrednio.

Kubernetes rozwiązuje ten problem przez kilka mechanizmów:
- **NodePort**: wystawia port na każdym węźle klastra.
- **LoadBalancer**: (dla chmury) przydziela zewnętrzny adres IP.
- **Ingress**: routing HTTP/HTTPS na poziomie warstwy 7, z jednym punktem wejścia i regułami dla wielu usług.

Ingress jest najbardziej elastyczny dla aplikacji webowych: jeden adres IP, wiele ścieżek, wiele usług, obsługa SSL, Basic Auth i inne funkcje HTTP.

### 9.2. NGINX Ingress Controller

Ingress to tylko reguła routingu — sam w sobie nic nie robi. Potrzebny jest Ingress Controller, który tę regułę realizuje. W projekcie to **NGINX Ingress Controller**, włączany jako addon minikube:

```bash
minikube addons enable ingress
```

Controller działa jako pod w namespace `ingress-nginx`. Obserwuje zasoby Ingress w całym klastrze i dynamicznie aktualizuje konfigurację Nginx, żeby ruch był routowany zgodnie z regułami.

### 9.3. ingressClassName: nginx

W Kubernetes 1.18+ Ingress musi jawnie wskazywać kontroler przez `spec.ingressClassName`. Bez tego pola Controller może zignorować zasób.

```yaml
spec:
  ingressClassName: nginx
```

`nginx` to nazwa IngressClass tworzonej przez addon minikube. Tak NGINX Ingress Controller identyfikuje, które zasoby Ingress należy przetwarzać.

To jeden z najczęstszych błędów przy wdrożeniu Ingress na nowszych wersjach Kubernetes — brak tego pola skutkuje pustą kolumną `ADDRESS` w `kubectl get ingress` i brakiem ruchu.

### 9.4. Routing ścieżek

```yaml
spec:
  ingressClassName: nginx
  rules:
    - host: budget.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80
          - path: /api-platform
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

`pathType: Prefix` oznacza, że reguła pasuje do wszystkich ścieżek zaczynających się podanym prefiksem. Kolejność reguł ma znaczenie — bardziej szczegółowe ścieżki (`/api`) powinny być przed ogólniejszymi (`/`).

- `/api/*` → Service `backend` → pody backend-nginx → FastCGI → pody backend-php → Symfony
- `/api-platform/*` → Service `backend` → API Platform (wygenerowane endpointy Doctrine)
- `/*` → Service `frontend` → pody frontend → pliki statyczne

### 9.5. Basic Auth na Ingressie

NGINX Ingress Controller obsługuje HTTP Basic Auth przez adnotacje i Secret:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: budget-basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "budget-app"
```

- `auth-type: basic` — włącza Basic Auth.
- `auth-secret: budget-basic-auth` — wskazuje Secret z formatem htpasswd.
- `auth-realm: "budget-app"` — nazwa wyświetlana w oknie dialogowym przeglądarki.

Adnotacje są odczytywane przez Controller i przekształcane w dyrektywę `auth_basic` konfiguracji Nginx. Każde żądanie HTTP do `budget.local` musi zawierać nagłówek `Authorization: Basic ...` z poprawnymi danymi. Bez niego Controller zwraca `401 Unauthorized`.

To nie jest aplikacyjny system kont użytkowników — to zabezpieczenie wejścia HTTP na poziomie infrastruktury. Symfony nie wie o tym uwierzytelnieniu.

### 9.6. Dostęp z macOS — tunel i /etc/hosts

Na macOS z Docker driverem minikube działa jako kontener Docker, a jego IP (`192.168.49.2`) jest osiągalne tylko wewnątrz sieci Docker — nie z systemu macOS. Rozwiązaniem jest `minikube tunnel`:

```bash
minikube tunnel
```

`minikube tunnel` uruchamia trwały proces, który tworzy route sieciowy między systemem macOS a siecią klastra. Dzięki temu ruch do klastra jest tunelowany przez localhost.

Po uruchomieniu tunelu `/etc/hosts` musi wskazywać na `127.0.0.1`, nie na `192.168.49.2`:

```
127.0.0.1 budget.local
```

Przeglądarka wysyła żądania do `budget.local`, DNS rozwiązuje do `127.0.0.1`, tunel przekierowuje ruch do NGINX Ingress Controller w klastrze, który obsługuje żądanie zgodnie z regułami Ingress.

---

## Rozdział 10. Service — stabilne adresy dla podów

### 10.1. Problem zmiennych adresów IP podów

Pod w Kubernetes ma efemeryczny adres IP. Kiedy pod jest usuwany i tworzony od nowa (przez restart, skalowanie, aktualizację), dostaje nowy adres IP. Gdyby komponenty komunikowały się bezpośrednio przez adresy IP podów, każda zmiana poda wymagałaby aktualizacji konfiguracji wszystkich klientów.

Kubernetes Service rozwiązuje ten problem: daje stabilną nazwę DNS i wirtualny adres IP (ClusterIP), który nie zmienia się przez cały czas życia Service. DNS klastra rozwiązuje nazwę Service do jego ClusterIP, a kube-proxy na każdym węźle przekierowuje ruch z ClusterIP do aktualnie zdrowych podów.

### 10.2. Typy Service

Projekt używa wyłącznie `ClusterIP` (domyślny typ). ClusterIP jest widoczny tylko wewnątrz klastra — to właściwy wybór dla komunikacji między komponentami aplikacji, które nie powinny być bezpośrednio dostępne z zewnątrz.

Inne typy (dla kontekstu):
- **NodePort**: wystawia port na każdym węźle klastra, dostępny z zewnątrz przez `<node-ip>:<node-port>`.
- **LoadBalancer**: (chmurowy) tworzy zewnętrzny load balancer z publicznym IP.
- **ExternalName**: DNS alias dla zewnętrznej usługi.

### 10.3. Selector i etykiety

Service łączy się z podami przez etykiety (labels) i selektor (selector):

```yaml
# Service:
spec:
  selector:
    app: backend-nginx

# Pod (Deployment template):
metadata:
  labels:
    app: backend-nginx
```

Kiedy Service dostaje żądanie, kube-proxy wybiera jeden z podów pasujących do selektora i przekazuje mu ruch (load balancing, domyślnie round-robin).

### 10.4. Discovery przez DNS

W klastrze Kubernetes każdy Service dostaje wpis DNS:

```
<service-name>.<namespace>.svc.cluster.local
```

Kontener w tym samym namespace może używać samej nazwy Service (np. `mysql`, `php`, `backend`, `frontend`). Pełna nazwa DNS jest potrzebna tylko przy komunikacji między namespace'ami.

---

## Rozdział 11. RBAC — kontrola dostępu

### 11.1. Co to jest RBAC

Role-Based Access Control (RBAC) w Kubernetes kontroluje, kto (lub co) może wykonywać jakie operacje na jakich zasobach. W projekcie RBAC dotyczy kont Service Account — tożsamości Kubernetes przypisywanych podom i narzędziom operacyjnym.

### 11.2. Service Account

Service Account to tożsamość Kubernetes dla procesów działających w klastrze. Każdy pod automatycznie dostaje Service Account `default`, chyba że jest wskazany inny.

Projekt definiuje dwa Service Account:
- `budget-viewer` — konto tylko do odczytu,
- `budget-root` — konto z pełnymi uprawnieniami w namespace `budget-app`.

### 11.3. Role

Role definiuje zestaw dozwolonych operacji na określonych zasobach w namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: budget-viewer
  namespace: budget-app
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "services", "configmaps", "events", "persistentvolumeclaims"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch"]
```

Role `budget-viewer` pozwala na odczyt (get, list, watch) zasobów aplikacji, ale nie na modyfikację ani usuwanie.

Role `budget-admin` (przypisana do `budget-root`) używa wildcardu:

```yaml
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
```

To pełne uprawnienia do wszystkich zasobów w namespace `budget-app`. Nie jest to administrator całego klastra — nie może tworzyć namespace'ów ani zmieniać zasobów klastrowych.

### 11.4. RoleBinding

RoleBinding wiąże Role z Service Account:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: budget-viewer-binding
  namespace: budget-app
subjects:
  - kind: ServiceAccount
    name: budget-viewer
    namespace: budget-app
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: budget-viewer
```

Weryfikacja uprawnień:

```bash
# budget-viewer może listować pody:
kubectl auth can-i list pods -n budget-app \
  --as=system:serviceaccount:budget-app:budget-viewer
# → yes

# budget-viewer nie może usuwać podów:
kubectl auth can-i delete pods -n budget-app \
  --as=system:serviceaccount:budget-app:budget-viewer
# → no

# budget-root może wszystko w namespace:
kubectl auth can-i '*' '*' -n budget-app \
  --as=system:serviceaccount:budget-app:budget-root
# → yes
```

---

## Rozdział 12. Zasoby CPU i pamięci

### 12.1. Requests i Limits

Każdy kontener w projekcie ma zdefiniowane `resources.requests` i `resources.limits`. To nie jest opcjonalne — to ważna część konfiguracji produkcyjnej.

**Requests** określają minimalne zasoby, które scheduler Kubernetes zarezerwuje dla kontenera. Scheduler szuka węzła z wystarczającymi wolnymi zasobami. Jeśli węzeł nie ma wystarczającej ilości CPU lub pamięci dla requests, pod nie zostanie na nim umieszczony.

**Limits** określają maksymalne zasoby, jakie kontener może wykorzystać. Jeśli kontener przekroczy limit CPU, zostanie spowalniany (throttling). Jeśli przekroczy limit pamięci, zostanie zabity przez Out-Of-Memory Killer (OOM Kill) i zrestartowany.

### 12.2. Wartości w projekcie

| Komponent | CPU request | CPU limit | RAM request | RAM limit |
|-----------|-------------|-----------|-------------|-----------|
| frontend | 50m | 250m | 64Mi | 128Mi |
| backend-nginx | 50m | 250m | 64Mi | 128Mi |
| backend-php | 100m | 500m | 128Mi | 512Mi |
| migrations (Job) | 50m | 500m | 128Mi | 512Mi |
| mysql | 250m | 1000m | 512Mi | 1Gi |

CPU jest mierzony w milirdzeniach (m): `1000m = 1 rdzeń`. MySQL dostaje `250m` gwarantowanego CPU i może użyć maksymalnie `1` rdzenia. PHP dostaje `100m` gwarantowanego CPU z limitem `500m`.

### 12.3. Dlaczego to ważne?

Bez requests i limits jeden kontener mógłby zająć całe zasoby węzła i zagłodzić inne. W lokalnym minikube to mniej krytyczne, ale w środowisku produkcyjnym z wieloma aplikacjami na jednym klastrze to kluczowe dla stabilności.

---

## Rozdział 13. Przepływ żądania — od przeglądarki do bazy

Poniżej szczegółowy opis tego, co dzieje się, gdy użytkownik otwiera aplikację przez `http://budget.local` i przegląda listę transakcji.

### 13.1. Otwarcie strony głównej

1. **Przeglądarka** wysyła `GET http://budget.local/`.
2. **System operacyjny** rozwiązuje DNS: `budget.local → 127.0.0.1` (wpis w `/etc/hosts`).
3. **Połączenie TCP** trafia do portu 80 na `127.0.0.1`, gdzie nasłuchuje tunel minikube.
4. **Tunel minikube** przekazuje połączenie do NGINX Ingress Controller w klastrze (port NodePort lub przez routing sieci).
5. **NGINX Ingress Controller** sprawdza nagłówek `Authorization`. Brak danych → `401 Unauthorized`.
6. **Przeglądarka** pokazuje okno dialogowe Basic Auth. Użytkownik wpisuje `budget-admin` / `budget-demo`.
7. **NGINX Ingress Controller** weryfikuje dane przez wpis htpasswd z Secret `budget-basic-auth`. Dane poprawne → kontynuacja.
8. **Ingress** dopasowuje ścieżkę `/` do reguły frontendu i przekazuje żądanie do Service `frontend`.
9. **Service `frontend`** (ClusterIP, kube-proxy) przekazuje ruch do jednego z podów frontendu.
10. **Pod frontend** (Nginx) szuka pliku `/` w `/usr/share/nginx/html`. Pasuje `index.html` → zwraca HTML.
11. **Przeglądarka** parsuje HTML. Ładuje `/assets/styles.css`, `/config.js`, `/assets/main.js`.
12. `/config.js` zawiera `window.__APP_CONFIG__ = { apiBaseUrl: "/api", requestTimeoutMs: 8000 }`.
13. `/assets/main.js` to skompilowany TypeScript — logika aplikacji.

### 13.2. Pobranie kategorii i transakcji

14. **JavaScript** wywołuje `GET /api/categories` i `GET /api/transactions?month=2026-06`.
15. **Przeglądarka** wysyła żądanie do `http://budget.local/api/categories` (z nagłówkiem `Authorization: Basic ...` — przeglądarka pamięta dane z kroku 6).
16. **Ingress** dopasowuje `/api` do reguły backendu → Service `backend`.
17. **Service `backend`** przekazuje do poda backend-nginx.
18. **Backend Nginx** (`docker/nginx/conf.d/default.conf`) obsługuje żądanie: ścieżka kończy się `.php` (przez FastCGI) lub jest obsługiwana przez `index.php` Symfony.
19. **FastCGI** do Service `php:9000` → pod backend-php.
20. **PHP-FPM** uruchamia `public/index.php` Symfony.
21. **Symfony** routuje żądanie do `BudgetController::getCategories()`.
22. **BudgetController** wywołuje `DoctrineBudgetEntryRepository` przez interfejs domenowy.
23. **Doctrine ORM** buduje zapytanie SQL i wysyła je do Service `mysql:3306`.
24. **Service `mysql`** przekazuje do poda `mysql-0` StatefulSet.
25. **MySQL** wykonuje zapytanie i zwraca wyniki.
26. **Doctrine** mapuje rekordy na obiekty PHP (`Category[]`).
27. **BudgetController** serializuje wyniki do JSON i zwraca odpowiedź HTTP 200.
28. Odpowiedź wraca przez PHP-FPM → backend-nginx → Service backend → Ingress → przeglądarka.
29. **JavaScript** parsuje JSON i renderuje kategorie w DOM.

### 13.3. Przepływ przez port-forward

Przy dostępie przez `make port-forward` przepływ jest inny:
- Brak Ingress — połączenie TCP do `127.0.0.1:18081` trafia bezpośrednio do poda frontendu.
- Brak Basic Auth — Ingress jest omijany.
- Frontend Nginx obsługuje i pliki statyczne, i `/api/` (przez proxy_pass do Service `backend` wewnątrz klastra).

---

## Rozdział 14. Kustomize — zarządzanie manifestami

### 14.1. Problem zarządzania wieloma plikami YAML

Aplikacja składa się z wielu zasobów Kubernetes: namespace, 3 Secret, 1 ConfigMap, 1 StatefulSet, 3 Deployment, 5 Service, 1 Job, 1 Ingress, 2 ServiceAccount, 2 Role, 2 RoleBinding. Zarządzanie każdym plikiem osobno przez `kubectl apply -f` jest żmudne i podatne na błędy. Kustomize rozwiązuje ten problem.

### 14.2. kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - mysql-secret.yaml
  - backend-configmap.yaml
  - ingress-auth-secret.yaml
  - rbac.yaml
  - mysql.yaml
  - backend.yaml
  - frontend.yaml
  - ingress.yaml
```

`kustomization.yaml` to manifest Kustomize — lista plików YAML, które razem tworzą pełne środowisko. Kolejność ma znaczenie: namespace musi istnieć przed innymi zasobami, Secret przed Deploymentami, które je montują.

### 14.3. Polecenia Kustomize

Aplikowanie:
```bash
kubectl apply -k ./k8s
```

Renderowanie bez aplikowania (dry-run):
```bash
kubectl kustomize ./k8s
```

Kustomize wbudowany w `kubectl` od wersji 1.14 nie wymaga instalacji osobnego narzędzia.

### 14.4. Możliwości Kustomize

Projekt używa Kustomize w podstawowym trybie — jako zagregowany manifest. Kustomize oferuje też:
- **patches**: nadpisywanie wartości w manifestach bez edycji oryginałów,
- **namePrefix / nameSuffix**: dodawanie prefiksów do nazw zasobów,
- **bases i overlays**: tworzenie środowisk (dev/staging/prod) przez nadpisywanie bazowej konfiguracji.

---

## Rozdział 15. Helm — szablonowane wdrożenia

### 15.1. Helm vs Kustomize

Kustomize aplikuje zasoby YAML z opcjonalnymi patchami. Helm idzie dalej — szablonuje zasoby i parametryzuje wdrożenie przez plik wartości (`values.yaml`). Różnica jest podobna do różnicy między statycznym HTML a szablonami Jinja.

Kustomize jest prostszy i lepszy dla ścisłej kontroli manifestów. Helm jest lepszy dla parametryzowanych wdrożeń — można zmienić liczbę replik, adres hosta, limity zasobów przez jeden plik wartości lub przez `--set` bez edycji manifestów.

### 15.2. Struktura Helm Charta

```
helm/budget-app/
├── Chart.yaml          # metadane charta
├── values.yaml         # domyślne wartości parametrów
└── templates/
    ├── _helpers.tpl    # pomocnicze funkcje szablonów
    ├── NOTES.txt       # instrukcja po instalacji
    ├── namespace.yaml
    ├── config.yaml     # ConfigMap + Secrets
    ├── backend.yaml    # PHP, Nginx, Service, Job
    ├── frontend-deployment.yaml
    ├── frontend-service.yaml
    ├── mysql.yaml
    ├── ingress.yaml
    ├── ingress-auth-secret.yaml
    └── rbac.yaml
```

### 15.3. values.yaml — parametryzacja

```yaml
ingress:
  enabled: true
  className: nginx
  host: budget.local
  basicAuth:
    enabled: true
    secretName: budget-basic-auth
    realm: budget-app

replicas:
  frontend: 1
  backendPhp: 1
  backendNginx: 1

mysql:
  storage: 1Gi
```

Zmiana parametru bez edycji pliku:
```bash
helm template budget-app ./helm/budget-app --set replicas.frontend=2
```

### 15.4. Szablony Helm

Szablon `ingress.yaml` w Helm:

```yaml
{{- if .Values.ingress.enabled }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className | quote }}
  {{- end }}
  rules:
    - host: {{ .Values.ingress.host | quote }}
```

`{{- if .Values.ingress.enabled }}` — renderuje zasób tylko jeśli Ingress jest włączony.
`.Values.ingress.host | quote` — wstawia wartość z values.yaml z cudzysłowem.

### 15.5. Polecenia Helm

```bash
helm lint ./helm/budget-app             # walidacja szablonów
helm template budget-app ./helm/budget-app  # renderowanie bez instalacji
helm install budget-app ./helm/budget-app \
  --namespace budget-app --create-namespace  # instalacja
helm upgrade budget-app ./helm/budget-app \
  --namespace budget-app                     # aktualizacja
helm rollback budget-app 1 --namespace budget-app  # cofnięcie do wersji 1
helm uninstall budget-app --namespace budget-app   # usunięcie
```

---

## Rozdział 16. Skalowanie i samonaprawianie

### 16.1. Kontroler Deploymentu

Deployment nie tylko tworzy pody — zarządza nimi przez kontroler, który stale porównuje pożądany stan (desired state) z aktualnym (actual state). Jeśli pod zostanie usunięty, kontroler automatycznie tworzy nowy.

### 16.2. Skalowanie replik

```bash
make scale-frontend REPLICAS=3
```

To wywołuje:
```bash
kubectl -n budget-app scale deployment/frontend --replicas=3
```

Kubernetes tworzy 2 dodatkowe pody frontendu. Service `frontend` automatycznie zaczyna load-balansować ruch między 3 podami.

Frontendem i backendem można skalować swobodnie. MySQL nie — jedna replika StatefulSet z PVC, bo MySQL nie obsługuje wielomasterowej replikacji out-of-the-box.

### 16.3. Symulacja awarii

```bash
make kill-frontend-pod
```

To wywołuje:
```bash
kubectl -n budget-app delete pod <pod-name>
```

Deployment wykrywa, że liczba działających podów jest niższa od pożądanej i natychmiast tworzy nowy. W ciągu kilku sekund:

```
frontend-59989b9754-lf2bg   0/1   Terminating   ...
frontend-59989b9754-abc12   0/1   ContainerCreating   ...
frontend-59989b9754-abc12   1/1   Running   ...
```

To demonstruje self-healing: Kubernetes sam przywraca środowisko do stanu pożądanego bez ręcznej interwencji.

### 16.4. Rolling Update

Przy aktualizacji obrazu Deployment domyślnie wykonuje rolling update: stopniowo zastępuje stare pody nowymi, zachowując dostępność serwisu. Strategia rolling update jest konfigurowana przez `spec.strategy`:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

Domyślnie: maksymalnie 1 pod niedostępny i 1 pod nadmiarowy podczas aktualizacji.

---

## Rozdział 17. Skrypty operacyjne i Makefile

### 17.1. scripts/up.sh — orchestracja wdrożenia

Skrypt `scripts/up.sh` jest głównym skryptem wdrożenia. Obsługuje tryby `default` (Kustomize) i `helm`.

Kroki w trybie `default`:
1. Uruchomienie minikube (`minikube start --driver=docker`).
2. Włączenie addonu Ingress (`minikube addons enable ingress`).
3. **Oczekiwanie na gotowość kontrolera Ingress** — kluczowy krok zapobiegający błędowi admission webhooka:
   ```bash
   kubectl -n ingress-nginx wait --for=condition=ready pod \
     -l app.kubernetes.io/component=controller --timeout=120s
   ```
   Bez tego kroku `kubectl apply` może trafić na niegotowy webhook walidacyjny NGINX Ingress Controller i zakończyć się błędem `Internal error occurred: failed calling webhook`.
4. Budowanie obrazów Docker w kontekście minikube.
5. Renderowanie manifestów do `/tmp` (dry-run).
6. Aplikowanie manifestów przez `kubectl apply -k ./k8s`.
7. Oczekiwanie na gotowość MySQL, ukończenie Joba migracji i rollout wszystkich Deploymentów.
8. Wyświetlenie statusu i instrukcji dostępu.

### 17.2. scripts/check.sh — weryfikacja stanu

`make check` uruchamia `scripts/check.sh`, który weryfikuje 11 punktów kontrolnych z kolorowym wyjściem PASS/FAIL/SKIP:

1. minikube running
2. Ingress controller pod ready
3. Ingress address assigned
4. MySQL pod ready
5. Migrations job completed
6. backend-php deployment ready
7. backend-nginx deployment ready
8. frontend deployment ready
9. API `/healthz` reachable (HTTP)
10. API `/readyz` reachable (HTTP)
11. Frontend HTML served

Punkty 9-11 wymagają uruchomionego port-forward. Jeśli port-forward nie działa, check automatycznie wyświetla SKIP zamiast FAIL.

```
==> budget-app health check

[PASS] minikube is running
[PASS] Ingress controller pod ready
[PASS] Ingress address assigned (192.168.49.2)
[PASS] MySQL pod ready
[PASS] Migrations job completed
[PASS] backend-php deployment ready
[PASS] backend-nginx deployment ready
[PASS] frontend deployment ready
[PASS] API /healthz reachable at http://127.0.0.1:18081
[PASS] API /readyz reachable at http://127.0.0.1:18081
[PASS] Frontend HTML served at http://127.0.0.1:18081

==> Results: 11/11 passed, 0 failed, 0 skipped
```

### 17.3. scripts/curl-smoke.sh — smoke test API

Smoke test weryfikuje pełen cykl życia transakcji przez 9 kroków:

1. `GET /api/healthz` — healthcheck
2. `GET /api/readyz` — readiness
3. `GET /api/categories` — lista kategorii
4. `GET /api/transactions?month=YYYY-MM` — lista transakcji
5. `POST /api/transactions` — tworzenie transakcji
6. `PUT /api/transactions/{id}` — aktualizacja
7. `GET /api/summary?month=YYYY-MM` — podsumowanie miesiąca
8. `DELETE /api/transactions/{id}` — usuwanie
9. `GET /api/transactions?month=YYYY-MM` — weryfikacja po usunięciu

Skrypt automatycznie parsuje `id` nowo utworzonej transakcji z JSON odpowiedzi (przez `jq` albo `sed` jako fallback) i używa go w krokach 6, 8.

### 17.4. Makefile — interfejs operacyjny

Makefile jest głównym interfejsem operacyjnym projektu. Zamiast pamiętać długie komendy kubectl, operator używa semantycznych targetów:

```bash
make up-default          # pełne wdrożenie
make check               # weryfikacja stanu
make port-forward        # dostęp lokalny
make smoke-port-forward  # test API
make scale-frontend REPLICAS=3  # skalowanie
make kill-frontend-pod   # symulacja awarii
make rbac-check          # weryfikacja RBAC
make helm-template       # renderowanie Helm
make reset               # czyszczenie środowiska
```

Każdy target przekazuje do skryptów zmienne środowiskowe przez prefiksy (`NS=$(NS) KUBECTL=$(KUBECTL) ...`), dzięki czemu wszystkie parametry są konfigurowane w jednym miejscu — na górze Makefile.

---

## Rozdział 18. Namespace i izolacja zasobów — głębsza analiza

### 18.1. Jak namespace wpływa na DNS

Wewnątrz klastra każdy Service dostaje pełną nazwę DNS:
```
<service>.<namespace>.svc.cluster.local
```

Kontener w namespace `budget-app` może połączyć się z MySQL przez:
- `mysql` (w tym samym namespace — wystarczy nazwa)
- `mysql.budget-app` (skrócona forma z namespace)
- `mysql.budget-app.svc.cluster.local` (pełna forma)

Kontener w innym namespace musiałby używać pełnej formy z namespace.

### 18.2. NetworkPolicy

Projekt nie używa NetworkPolicy, ale warto wiedzieć, że Kubernetes umożliwia też sieciową izolację na poziomie namespace przez NetworkPolicy. Bez NetworkPolicy wszystkie pody w klastrze mogą się ze sobą komunikować — nawet między namespace'ami. W środowisku produkcyjnym NetworkPolicy izolowałaby namespace `budget-app` od innych aplikacji.

---

## Rozdział 19. Wdrożenie end-to-end — pełna sekwencja

Poniżej kompletna sekwencja od pustego środowiska do działającej aplikacji:

```bash
# 1. Uruchom minikube (jeśli nie działa)
minikube start --driver=docker

# 2. Włącz NGINX Ingress Controller
minikube addons enable ingress

# 3. Poczekaj na kontroler Ingress
kubectl -n ingress-nginx wait \
  --for=condition=ready pod \
  -l app.kubernetes.io/component=controller \
  --timeout=120s

# 4. Przełącz Docker na daemon minikube
eval $(minikube docker-env)

# 5. Zbuduj obrazy
docker build -t budget-backend-php:latest --target app_php_prod ./Backend
docker build -t budget-backend-nginx:latest --target app_nginx ./Backend
docker build -t budget-frontend:latest ./Frontend

# 6. Zastosuj manifesty
kubectl apply -k ./k8s

# 7. Poczekaj na gotowość
kubectl -n budget-app wait \
  --for=condition=ready pod -l app=mysql --timeout=240s
kubectl -n budget-app wait \
  --for=condition=complete job/backend-migrations --timeout=300s
kubectl -n budget-app rollout status deployment/backend-php --timeout=240s
kubectl -n budget-app rollout status deployment/backend-nginx --timeout=240s
kubectl -n budget-app rollout status deployment/frontend --timeout=240s

# Albo jednym poleceniem:
make up-default

# 8. Dostęp przez port-forward
kubectl -n budget-app port-forward svc/frontend 18081:80
# → http://127.0.0.1:18081

# Lub przez Ingress (macOS):
minikube tunnel  # osobny terminal
echo "127.0.0.1 budget.local" | sudo tee -a /etc/hosts
# → http://budget.local (login: budget-admin, hasło: budget-demo)

# 9. Weryfikacja
make check
make smoke-port-forward
```

---

## Podsumowanie

Aplikacja budżetu domowego w Kubernetes jest przykładem kompletnego wdrożenia aplikacji trójwarstwowej z użyciem nowoczesnych narzędzi infrastruktury. Projekt demonstruje:

- **Konteneryzację**: wieloetapowe buildy Docker, runtime config przez envsubst, imagePullPolicy dla lokalnych obrazów.
- **Separację konfiguracji od kodu**: ConfigMap dla jawnych ustawień, Secret dla danych wrażliwych, wstrzyknięcie przez envFrom.
- **Właściwe typy zasobów**: StatefulSet dla bazy danych ze stanem i PVC, Deployment dla bezstanowych komponentów, Job dla jednorazowych zadań.
- **Sieć wewnętrzną**: ClusterIP Service jako stabilny endpoint DNS, headless Service dla StatefulSet, komunikacja przez nazwy Service.
- **Zewnętrzny dostęp**: NGINX Ingress Controller z ingressClassName, routing ścieżek, Basic Auth przez adnotacje i Secret htpasswd.
- **Bezpieczeństwo operacyjne**: RBAC z zasadą minimalnych uprawnień (budget-viewer) i kontem administracyjnym (budget-root) ograniczonym do jednego namespace.
- **Zasoby CPU i pamięci**: requests i limits dla każdego kontenera zapewniające przewidywalne zachowanie schedulera.
- **Zarządzanie manifestami**: Kustomize jako główna metoda, Helm jako parametryzowana alternatywa.
- **Automatyzację**: Makefile jako interfejs operacyjny, skrypty wdrożenia z obsługą błędów i oczekiwaniem na gotowość, health check i smoke test.
- **Samonaprawianie**: Deployment controller automatycznie przywraca zadaną liczbę replik po awarii poda.

Każda decyzja projektowa — od wyboru StatefulSet dla MySQL, przez runtime config frontendu, po oczekiwanie na webhook Ingress przed aplikowaniem manifestów — wynika z konkretnych właściwości Kubernetes i praktycznych doświadczeń z lokalnym klastrem minikube na macOS.
