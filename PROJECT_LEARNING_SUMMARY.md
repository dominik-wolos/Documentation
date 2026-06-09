# Skrócona ścieżka nauki projektu

Ten dokument prowadzi od najmniejszych elementów do pełnej architektury `budget-app`. Można go potraktować jako checklistę przed prezentacją i pytaniami technicznymi.

## 1. Transakcja

Najmniejszy element aplikacji to transakcja budżetowa. Ma tytuł, kwotę, typ, kategorię, datę i notatkę.

Typ transakcji:

- `income` - przychód,
- `expense` - wydatek.

## 2. Kategoria

Kategoria grupuje transakcje. Ma kod, nazwę i opcjonalny limit miesięczny. Dzięki temu aplikacja może pokazać, ile wydano w danej kategorii i czy limit został przekroczony.

## 3. Podsumowanie miesiąca

Backend liczy:

- sumę przychodów,
- sumę wydatków,
- bilans,
- stopę oszczędności,
- największy wydatek,
- sumy według kategorii.

## 4. API

Frontend nie łączy się bezpośrednio z bazą. Rozmawia z backendem przez API:

- `GET /api/categories`
- `GET /api/transactions?month=YYYY-MM`
- `POST /api/transactions`
- `PUT /api/transactions/{id}`
- `DELETE /api/transactions/{id}`
- `GET /api/summary?month=YYYY-MM`
- `GET /api/healthz`
- `GET /api/readyz`

## 5. Backend

Backend to Symfony. Najważniejszy plik do omówienia to `Backend/src/Core/Controller/BudgetController.php`.

Warto umieć powiedzieć:

- kontroler odbiera żądania HTTP,
- waliduje dane wejściowe,
- korzysta z Doctrine,
- zapisuje i odczytuje encje,
- zwraca JSON dla frontendu.

## 6. Frontend

Frontend to TypeScript serwowany jako pliki statyczne. Najważniejszy plik to `Frontend/src/main.ts`.

Warto umieć powiedzieć:

- frontend pobiera dane z `/api`,
- renderuje formularz, listę transakcji i podsumowanie,
- adres API jest konfigurowany w runtime przez `config.js`,
- w Kubernetes `API_BASE_URL` ma wartość `/api`.

## 7. Baza danych

Baza to MySQL. Dane są tworzone przez migracje Doctrine.

W Kubernetes baza działa jako `StatefulSet`, bo przechowuje dane. Dane są zapisywane na PVC `mysql-data`.

## 8. Obrazy Docker

Projekt używa trzech obrazów:

- backend PHP-FPM,
- backend Nginx,
- frontend Nginx.

Najprostsza odpowiedź: PHP-FPM wykonuje kod Symfony, Nginx przyjmuje HTTP, a frontend Nginx serwuje statyczną aplikację.

## 9. Kubernetes

Najważniejsze pojęcia w projekcie:

- `Namespace` - oddziela zasoby projektu.
- `ConfigMap` - konfiguracja jawna.
- `Secret` - hasła i sekrety.
- `Deployment` - uruchamia bezstanowe aplikacje.
- `StatefulSet` - uruchamia bazę danych ze stanem.
- `Service` - stabilny adres dla podów.
- `Ingress` - wejście HTTP do aplikacji.
- `Basic Auth` - proste logowanie na wejściu przez Ingress.
- `RBAC` - kontrola uprawnień do operacji w Kubernetes.
- `Job` - jednorazowe migracje.
- `PVC` - trwałe miejsce na dane MySQL.

## 10. Przepływ ruchu

Najważniejszy przepływ do zapamiętania:

```text
użytkownik
  -> Ingress budget.local
  -> Basic Auth
  -> frontend albo backend
  -> backend-nginx
  -> php service
  -> backend-php
  -> mysql service
  -> MySQL StatefulSet
```

## 11. Komendy, które warto znać

Uruchomienie (jedno polecenie):

```bash
make up-default
```

Lub krok po kroku:

```bash
make minikube-start
make minikube-ingress
make build-images
make apply
make wait
```

Sprawdzenie stanu:

```bash
make check            # pełna weryfikacja 11 punktów
make status
make pods
make services
```

Dostęp i testy:

```bash
make port-forward              # osobny terminal → http://127.0.0.1:18081
make smoke-port-forward        # test API przez port-forward
make minikube-tunnel           # osobny terminal (macOS) → http://budget.local
make smoke-k8s                 # test API przez Ingress
```

Skalowanie:

```bash
make scale-frontend REPLICAS=2
make scale-down
```

Samonaprawianie:

```bash
make kill-frontend-pod
make pods
```

Helm:

```bash
make helm-template
```

Logowanie i RBAC:

```bash
make auth-info
curl -u budget-admin:budget-demo http://budget.local/api/healthz
make rbac-check
make rbac-root-check
```

## 12. Logowanie i autoryzacja

W projekcie logowanie jest rozwiązane infrastrukturalnie przez Basic Auth na Ingressie. To znaczy, że użytkownik musi podać login i hasło zanim ruch trafi do frontendu albo backendu.

Autoryzacja operacyjna jest rozwiązana przez Kubernetes RBAC. Konto `budget-viewer` ma tylko odczyt, a `budget-root` może zarządzać zasobami w namespace `budget-app`.

## 13. Krótka odpowiedź na prezentacji

Projekt to aplikacja budżetu domowego podzielona na frontend, backend i bazę danych. Frontend jest statyczny i komunikuje się z backendem przez `/api`. Backend Symfony udostępnia REST API i korzysta z Doctrine. MySQL działa jako StatefulSet z trwałym wolumenem. Kubernetes Service daje stabilne adresy dla podów, Ingress wystawia aplikację pod `budget.local`, Basic Auth zabezpiecza wejście HTTP, RBAC rozdziela uprawnienia administracyjne, a migracje bazy uruchamia osobny Job.

## 14. Checklista obrony

Na obronie można zostać poproszonym o wyjaśnienie aplikacji, manifestów, drobną modyfikację i uruchomienie projektu. Najbezpieczniej trzymać się kolejności: co robi aplikacja, z czego się składa, jak ruch przechodzi przez Kubernetes, jak ją uruchomić, jak ją zmienić.

### Co pokazać jako pierwsze

```bash
make status
make pods
make services
```

Następnie pokaż aplikację przez port-forward (najprostsze, bez tunelu):

```bash
make port-forward        # osobny terminal
make smoke-port-forward
```

Lub przez Ingress (wymaga tunelu na macOS):

```bash
make minikube-tunnel     # osobny terminal
make smoke-k8s
```

### Pliki manifestów do omówienia

- `k8s/namespace.yaml` - tworzy namespace projektu.
- `k8s/mysql-secret.yaml` - dane bazy i `DATABASE_URL`.
- `k8s/backend-configmap.yaml` - jawna konfiguracja backendu.
- `k8s/mysql.yaml` - MySQL jako StatefulSet, service i PVC.
- `k8s/backend.yaml` - PHP-FPM, backend Nginx, service i Job migracji.
- `k8s/frontend.yaml` - frontend jako Deployment i Service.
- `k8s/ingress.yaml` - routing HTTP i Basic Auth.
- `k8s/rbac.yaml` - konta `budget-viewer` i `budget-root`.
- `helm/budget-app/templates/` - szablony Helm dla tych samych zasobów.

### Drobne modyfikacje, które można zrobić na żywo

Zmiana liczby replik frontendu:

```bash
make scale-frontend REPLICAS=2
make pods
make scale-down
```

Zmiana wartości przez Helm bez edycji pliku:

```bash
helm template budget-app ./helm/budget-app --set replicas.frontend=2
```

Restart deploymentów po przebudowie obrazu:

```bash
make restart
make rollout
```

Ponowne uruchomienie migracji:

```bash
make rerun-migrations
```

Sprawdzenie RBAC:

```bash
make rbac-check
make rbac-root-check
```

### Pytania techniczne i krótkie odpowiedzi

Dlaczego MySQL jest StatefulSetem?
Bo baza danych przechowuje stan i potrzebuje stabilnego wolumenu. Deployment pasuje lepiej do usług bezstanowych.

Po co jest Service?
Pody mogą się zmieniać i dostawać nowe adresy IP. Service daje stabilną nazwę DNS i port.

Po co jest Ingress?
Ingress wystawia aplikację przez jeden adres HTTP i rozdziela ścieżki `/`, `/api`, `/api-platform`.

Po co jest Job migracji?
Migracje mają wykonać się raz i zakończyć. Dlatego Job jest lepszy niż Deployment.

Czym różni się ConfigMap od Secret?
ConfigMap przechowuje konfigurację jawną, a Secret dane wrażliwe, takie jak hasła i `APP_SECRET`.

Czym jest RBAC?
RBAC kontroluje, kto może wykonywać operacje na zasobach Kubernetes. W projekcie `budget-viewer` może czytać, a `budget-root` zarządza namespace.

Czy Basic Auth to logowanie użytkowników aplikacji?
Nie. To zabezpieczenie wejścia HTTP na poziomie Ingress NGINX. Aplikacja nie ma osobnego systemu kont użytkowników.

Czym różni się Kustomize od Helm?
Kustomize aplikuje zestaw zwykłych manifestów z katalogu `k8s/`. Helm używa szablonów i wartości z `values.yaml`, więc łatwiej zmieniać parametry wdrożenia.

### Minimalny scenariusz, gdy jest mało czasu

```bash
make validate
make status
curl -u budget-admin:budget-demo http://budget.local/api/healthz
make scale-frontend REPLICAS=2
make pods
make scale-down
make helm-template
make rbac-check
```
