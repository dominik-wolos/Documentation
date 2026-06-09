# Instrukcja zarządzania budget-app w Kubernetes

Ten dokument opisuje codzienne operacje potrzebne do uruchomienia, sprawdzenia i utrzymania aplikacji `budget-app` w lokalnym klastrze Kubernetes. Skupia się na właściwej aplikacji: frontendzie, backendzie, bazie danych, migracjach, Ingressie, Basic Auth, RBAC, Helm Charcie i komendach z `Makefile`.

## Zakres

Projekt składa się z aplikacji webowej do zarządzania budżetem domowym oraz konfiguracji Kubernetes potrzebnej do jej uruchomienia. Główny wariant wdrożenia korzysta z Kustomize (`kubectl apply -k ./k8s`), a Helm Chart jest alternatywną, szablonowaną formą instalacji.

## Struktura

- `k8s/` - główne manifesty aplikacji budżetowej.
- `helm/budget-app/` - Helm Chart szablonujący aplikację.
- `Makefile` - komendy do budowania, wdrażania, skalowania, testowania i obsługi Helm.
- `scripts/curl-smoke.sh` - prosty test API wykonywany przez `curl`.

## Główna aplikacja

Główna aplikacja obejmuje:

- `Namespace budget-app` - izoluje zasoby projektu.
- `ConfigMap backend-config` - przechowuje konfigurację niesekretną.
- `Secret mysql-secret` - przechowuje dane bazy i `DATABASE_URL`.
- `Secret backend-secret` - przechowuje `APP_SECRET`.
- `Secret budget-basic-auth` - przechowuje wpis htpasswd dla logowania przez Ingress.
- `StatefulSet mysql` - uruchamia bazę MySQL.
- `Service mysql` - daje stabilny adres `mysql:3306`.
- `Service mysql-headless` - pokazuje mechanizm Headless Service.
- `Deployment backend-php` - uruchamia Symfony jako PHP-FPM.
- `Service php` - daje backendowemu Nginx adres `php:9000`.
- `Deployment backend-nginx` - obsługuje HTTP backendu.
- `Service backend` - udostępnia backend w klastrze.
- `Job backend-migrations` - wykonuje migracje Doctrine.
- `Deployment frontend` - uruchamia aplikację TypeScript przez Nginx.
- `Service frontend` - udostępnia frontend w klastrze.
- `Ingress budget-ingress` - kieruje `/api` do backendu, `/` do frontendu.
- `ServiceAccount budget-viewer` - konto operacyjne tylko do odczytu.
- `ServiceAccount budget-root` - konto operacyjne z pełnymi uprawnieniami w namespace `budget-app`.
- `Role` i `RoleBinding` - wiążą konta z uprawnieniami RBAC.

## Uruchomienie od zera

Najprostszy wariant — jedno polecenie robi wszystko:

```bash
make up-default
```

Alternatywnie krok po kroku:

1. Uruchom minikube:

```bash
make minikube-start
```

2. Włącz Ingress (skrypt `up-default` czeka automatycznie na gotowość kontrolera):

```bash
make minikube-ingress
```

3. Zbuduj obrazy w Docker daemon minikube:

```bash
make build-images
```

4. Zastosuj główne manifesty:

```bash
make apply
```

5. Poczekaj na start aplikacji:

```bash
make wait
```

6. Zweryfikuj stan wszystkich komponentów:

```bash
make check
```

7. Wystaw frontend przez port-forward:

```bash
make port-forward
```

8. Otwórz aplikację:

```bash
open http://127.0.0.1:18081/
```

Aby uzyskać dostęp przez `budget.local` zamiast port-forward (macOS + Docker driver):

```bash
make minikube-tunnel          # osobny terminal, wymaga sudo
echo "127.0.0.1 budget.local" | sudo tee -a /etc/hosts
open http://budget.local      # login: budget-admin, hasło: budget-demo
```

## Sprawdzenie działania

Status wszystkich zasobów:

```bash
make status
```

Lista podów:

```bash
make pods
```

Lista service:

```bash
make services
```

Smoke test API przez port-forward:

```bash
make smoke-port-forward
```

Ręczny healthcheck:

```bash
curl http://127.0.0.1:18081/api/healthz
```

Healthcheck przez Ingress z Basic Auth:

```bash
curl -u budget-admin:budget-demo http://budget.local/api/healthz
```

Oczekiwana odpowiedź:

```json
{"status":"ok","service":"budget-backend"}
```

## Logowanie przez Ingress

Ingress używa HTTP Basic Auth obsługiwanego przez NGINX Ingress Controller. Dane demonstracyjne:

- login: `budget-admin`
- hasło: `budget-demo`

Sprawdzenie danych:

```bash
make auth-info
```

Test przez Ingress:

```bash
curl -u budget-admin:budget-demo http://budget.local/api/healthz
```

Test bez loginu powinien zwrócić `401 Unauthorized`:

```bash
curl -i http://budget.local/api/healthz
```

To zabezpieczenie działa przed aplikacją. Ruch bez poprawnych danych logowania nie powinien trafić do frontendu ani backendu.

## Logi

Frontend:

```bash
make logs-frontend
```

Backend PHP:

```bash
make logs-backend-php
```

Backend Nginx:

```bash
make logs-backend-nginx
```

MySQL:

```bash
make logs-mysql
```

## Skalowanie

Zmiana liczby replik frontendu:

```bash
make scale-frontend REPLICAS=2
```

Zmiana liczby replik backendu PHP:

```bash
make scale-backend-php REPLICAS=2
```

Zmiana liczby replik backendowego Nginx:

```bash
make scale-backend-nginx REPLICAS=2
```

Powrót do jednej repliki:

```bash
make scale-down
```

Uwaga: MySQL jest bazą stanową i domyślnie działa jako jedna replika. Nie skalujemy go tymi komendami, bo replikacja bazy wymaga osobnej konfiguracji.

## Symulowanie awarii

Usunięcie poda frontendu:

```bash
make kill-frontend-pod
```

Usunięcie poda backendu PHP:

```bash
make kill-backend-php-pod
```

Usunięcie poda backendowego Nginx:

```bash
make kill-backend-nginx-pod
```

Usunięcie poda MySQL:

```bash
make kill-mysql-pod
```

Po usunięciu poda Deployment albo StatefulSet tworzy nowy pod, aby przywrócić zadany stan.

## Wejście do kontenera

Shell w podzie frontendu:

```bash
make exec-frontend
```

Shell w podzie backendu PHP:

```bash
make exec-backend-php
```

Ręczny przykład:

```bash
kubectl -n budget-app exec -it deploy/frontend -- sh
```

## Zasoby CPU i pamięci

Wyświetlenie requests i limits:

```bash
make describe-resources
```

Podgląd zużycia zasobów, jeśli metrics-server jest dostępny:

```bash
make top-pods
```

Requests mówią schedulerowi, ile zasobów kontener potrzebuje minimalnie. Limits ustawiają górny limit zużycia.

## RBAC

Projekt ma dwa konta service account do pokazania autoryzacji operacyjnej w Kubernetes:

- `budget-viewer` - konto tylko do odczytu.
- `budget-root` - konto administracyjne w namespace `budget-app`.

`budget-viewer` może listować i podglądać zasoby aplikacji, ale nie może ich usuwać ani modyfikować.

Sprawdzenie:

```bash
make rbac-check
```

Ręczne komendy:

```bash
kubectl auth can-i list pods -n budget-app --as=system:serviceaccount:budget-app:budget-viewer
kubectl auth can-i get pods/log -n budget-app --as=system:serviceaccount:budget-app:budget-viewer
kubectl auth can-i delete pods -n budget-app --as=system:serviceaccount:budget-app:budget-viewer
```

`budget-root` ma pełne uprawnienia do zasobów namespaced w `budget-app`. Nie jest administratorem całego klastra, ale może zarządzać zasobami projektu.

Sprawdzenie:

```bash
make rbac-root-check
```

Ręczne komendy:

```bash
kubectl auth can-i '*' '*' -n budget-app --as=system:serviceaccount:budget-app:budget-root
kubectl auth can-i delete pods -n budget-app --as=system:serviceaccount:budget-app:budget-root
kubectl auth can-i patch deployments -n budget-app --as=system:serviceaccount:budget-app:budget-root
```

## Migracje

Migracje Doctrine są uruchamiane jako Kubernetes `Job`, ponieważ mają wykonać jednorazową zmianę schematu i zakończyć pracę.

Ponowne uruchomienie migracji:

```bash
make rerun-migrations
```

Samo usunięcie istniejącego joba:

```bash
make clean-job
```

Po ponownym zastosowaniu manifestów job zostanie utworzony jeszcze raz.

## Helm

Helm Chart znajduje się w `helm/budget-app`.

Główne wdrożenie projektu działa przez `kubectl apply -k ./k8s`. Helm Chart jest alternatywną, szablonowaną formą wdrożenia. Jeżeli aplikacja jest już utworzona przez `kubectl apply`, przed instalacją przez Helm najlepiej wykonać reset namespace albo użyć osobnego środowiska, ponieważ Helm oczekuje, że będzie właścicielem instalowanych zasobów.

Jeżeli `helm` nie jest zainstalowany, na macOS najprościej zainstalować go przez Homebrew:

```bash
brew install helm
```

Walidacja chartu:

```bash
make helm-lint
```

Renderowanie manifestów bez instalacji:

```bash
make helm-template
```

Instalacja:

```bash
make helm-install
```

Pełna komenda bez Makefile:

```bash
helm install budget-app ./helm/budget-app --namespace budget-app --create-namespace
```

Aktualizacja:

```bash
make helm-upgrade
```

Pełna komenda bez Makefile:

```bash
helm upgrade budget-app ./helm/budget-app --namespace budget-app
```

Status release:

```bash
make helm-status
```

Lista release:

```bash
make helm-list
```

Usunięcie release:

```bash
make helm-uninstall
```

Spakowanie chartu:

```bash
make helm-package
```

Przykładowa zmiana wartości bez edycji pliku:

```bash
helm upgrade budget-app ./helm/budget-app --namespace budget-app --set replicas.frontend=2
```

Rollback po zmianie:

```bash
helm rollback budget-app 1 --namespace budget-app
```

## Reset środowiska

Usunięcie całego namespace:

```bash
make reset
```

To usuwa aplikację, bazę danych i zasoby namespaced projektu. Po resecie można ponownie wykonać `make apply` oraz `make wait`.

## Najkrótszy scenariusz prezentacji

1. Pokaż działającą aplikację i zweryfikuj stan:

```bash
make status
make check
make port-forward
```

2. Pokaż skalowanie:

```bash
make scale-frontend REPLICAS=2
make pods
make scale-down
```

3. Pokaż samonaprawianie:

```bash
make kill-frontend-pod
make pods
```

4. Pokaż Helm jako alternatywny sposób wyrenderowania manifestów:

```bash
make helm-template
```

5. Pokaż logowanie i RBAC:

```bash
make auth-info
curl -u budget-admin:budget-demo http://budget.local/api/healthz
make rbac-check
make rbac-root-check
```
