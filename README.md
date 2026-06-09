# Aplikacja do zarządzania budżetem domowym

Ten dokument jest pełnym opisem projektu przeniesionym do katalogu `docs/`. Główny `README.md` w katalogu repozytorium pełni teraz rolę krótkiej instrukcji uruchomienia infrastruktury przez `Makefile`.

Projekt zaliczeniowy: aplikacja webowa uruchamiana w lokalnym klastrze Kubernetes z wykorzystaniem minikube.

Aplikacja pozwala prowadzić prosty budżet domowy: dodawać przychody i wydatki, przypisywać je do kategorii, przeglądać miesięczny bilans, kontrolować limity kategorii oraz usuwać lub edytować transakcje.

## Technologie

- Backend: PHP 8.3, Symfony 7.2, Doctrine ORM, Doctrine Migrations
- Frontend: TypeScript, HTML, CSS, statyczny serwer Nginx
- Baza danych: MySQL 8.0
- Infrastruktura: Docker, Kubernetes, minikube, Ingress, Helm
- Bezpieczeństwo infrastruktury: Basic Auth na Ingressie, RBAC w namespace aplikacji

## Struktura projektu

- `Backend/` - aplikacja Symfony udostępniająca API REST
- `Frontend/` - aplikacja TypeScript prezentująca budżet użytkownikowi
- `k8s/` - główne manifesty Kubernetes
- `helm/budget-app/` - Helm Chart dla aplikacji
- `Makefile` - komendy do budowania, wdrażania, testowania i zarządzania aplikacją
- `docs/OPIS_APLIKACJI.md` - dokładny opis działania aplikacji i elementów Kubernetes
- `docs/INSTRUKCJA_ZARZADZANIA.md` - instrukcja zarządzania aplikacją w Kubernetes i Helm
- `docs/PROJECT_FULL_DESCRIPTION.md` - pełny opis techniczny projektu
- `docs/PROJECT_LEARNING_SUMMARY.md` - skrócona ścieżka nauki do prezentacji i pytań
- `docs/AUDYT_LABORATORIA.md` - notatka o zakresie po redukcji konfiguracji Kubernetes
- `docs/PODSUMOWANIE_PRAC.md` - historia wykonanych prac w projekcie
- `docs/KUBERNETES_WYKLAD.md` - kompletny wykład akademicki o implementacji Kubernetes na podstawie tej aplikacji
- `AI_CONTEXT.md` - opis użycia narzędzia LLM jako wsparcia przy pracy nad projektem

## API

Backend udostępnia następujące endpointy:

- `GET /api/healthz` - prosty healthcheck aplikacji
- `GET /api/readyz` - sprawdzenie połączenia z bazą danych
- `GET /api/categories` - lista kategorii budżetowych
- `GET /api/transactions?month=YYYY-MM` - transakcje i podsumowanie dla miesiąca
- `POST /api/transactions` - dodanie transakcji
- `PUT /api/transactions/{id}` - edycja transakcji
- `DELETE /api/transactions/{id}` - usunięcie transakcji
- `GET /api/summary?month=YYYY-MM` - samo podsumowanie miesiąca

## Dostęp do aplikacji

Ingress jest zabezpieczony przez HTTP Basic Auth. Dane demonstracyjne:

- login: `budget-admin`
- hasło: `budget-demo`

To logowanie działa na poziomie NGINX Ingress Controller, czyli przed wejściem ruchu do frontendu i backendu. Nie jest to system kont użytkowników w aplikacji.

## Uruchomienie w minikube

Wymagania lokalne:

- Docker
- minikube
- kubectl

1. Uruchom minikube i włącz ingress:

```bash
minikube start --driver=docker
minikube addons enable ingress
```

2. Przełącz terminal na Docker daemon minikube:

```bash
eval $(minikube docker-env)
```

3. Zbuduj obrazy aplikacji wewnątrz środowiska minikube:

```bash
docker build -t budget-backend-php:latest --target app_php_prod ./Backend
docker build -t budget-backend-nginx:latest --target app_nginx ./Backend
docker build -t budget-frontend:latest ./Frontend
```

4. Zastosuj manifesty Kubernetes:

```bash
kubectl apply -k ./k8s
```

5. Poczekaj na bazę danych i migracje:

```bash
kubectl -n budget-app wait --for=condition=ready pod -l app=mysql --timeout=180s
kubectl -n budget-app wait --for=condition=complete job/backend-migrations --timeout=240s
kubectl -n budget-app get pods
```

6. Uruchom tunel w osobnym terminalu (wymagane na macOS z Docker driverem):

```bash
minikube tunnel
```

7. Dodaj wpis hosta dla Ingress (wskazując na `127.0.0.1`, bo tunel wystawia ruch lokalnie):

```bash
echo "127.0.0.1 budget.local" | sudo tee -a /etc/hosts
```

8. Zweryfikuj stan środowiska:

```bash
make check
```

9. Otwórz aplikację:

```bash
open http://budget.local
```

Alternatywnie — bez tunelu — przez port-forward:

```bash
make port-forward   # osobny terminal
open http://127.0.0.1:18081
```

## Szybkie sprawdzenie API

```bash
curl -u budget-admin:budget-demo http://budget.local/api/healthz
curl -u budget-admin:budget-demo "http://budget.local/api/transactions?month=2026-05"
```

## Validate it works through curl

Below are ready-to-use curl commands and a small smoke-test script that validate the API end-to-end. Adjust BASE_URL depending on where you run the backend.

- Local Symfony/Nginx (outside Kubernetes): BASE_URL="http://localhost:8000/api"
- In minikube via Ingress: BASE_URL="http://budget.local/api"

Quick manual checks

```bash
# Set your base URL once for the shell
BASE_URL="http://budget.local/api"
AUTH="-u budget-admin:budget-demo"

# Health and readiness through Ingress Basic Auth
curl -sS $AUTH "$BASE_URL/healthz" | jq .
curl -sS $AUTH "$BASE_URL/readyz" | jq .

# List categories
curl -sS $AUTH -H 'Accept: application/json' "$BASE_URL/categories" | jq .

# List transactions for a specific month
MONTH=$(date +%Y-%m)
curl -sS $AUTH -H 'Accept: application/json' "$BASE_URL/transactions?month=$MONTH" | jq .

# Create a transaction
TODAY=$(date +%F)
curl -sS $AUTH -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{
    "title": "Test purchase",
    "amount": 12.34,
    "type": "expense",
    "category": "other",
    "occurredAt": "'"$TODAY"'",
    "note": "created via curl"
  }' \
  "$BASE_URL/transactions" | jq .

# Update a transaction (replace <ID> with the id returned above)
ID=<ID>
curl -sS $AUTH -X PUT \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{
    "title": "Test purchase (updated)",
    "amount": 15.00,
    "type": "expense",
    "category": "other",
    "occurredAt": "'"$TODAY"'",
    "note": "updated via curl"
  }' \
  "$BASE_URL/transactions/$ID" | jq .

# Summary for the month
curl -sS $AUTH -H 'Accept: application/json' "$BASE_URL/summary?month=$MONTH" | jq .

# Delete the transaction
curl -sS $AUTH -X DELETE -H 'Accept: application/json' "$BASE_URL/transactions/$ID" -i
```

Automated smoke test

You can also run a one-shot script that performs the whole flow automatically (health, categories, list, create, update, summary, delete).

```bash
# Option A: from repo root (wrapper script)
./curl-smoke.sh

# Option B: directly call the script in scripts/
./scripts/curl-smoke.sh

# Option C: via Makefile helpers
make smoke            # uses BASE_URL if set
make smoke-local      # BASE_URL=http://localhost:8000/api
make smoke-k8s        # BASE_URL=http://budget.local/api with Basic Auth

# You can always override the target API explicitly:
BASE_URL="http://budget.local/api" CURL_AUTH="budget-admin:budget-demo" ./curl-smoke.sh
```

The script uses jq if available to pretty-print JSON. If jq is not installed, it falls back to raw output.

## Reset środowiska

Usunięcie namespace usuwa również dane MySQL zapisane w PVC:

```bash
kubectl delete namespace budget-app
```

Po resecie można ponownie wykonać kroki z sekcji uruchamiania.

## Jak przebudować aplikację (How to rebuild the app)

Poniższe kroki zakładają, że aplikacja działa w minikube tak jak w sekcji „Uruchomienie w minikube”.

1) Przełącz Docker na daemon minikube (za każdym nowym otwarciem terminala):

```bash
eval $(minikube docker-env)
```

2) Zbuduj ponownie obrazy z aktualnym kodem (tagi muszą pozostać takie same jak w manifestach Kubernetes):

```bash
# Backend (PHP FPM)
docker build -t budget-backend-php:latest --target app_php_prod ./Backend

# Backend (Nginx serwujący /public)
docker build -t budget-backend-nginx:latest --target app_nginx ./Backend

# Frontend (Nginx + statyczne pliki)
docker build -t budget-frontend:latest ./Frontend
```

W razie potrzeby wymuś pełne przebudowanie bez cache:

```bash
docker build --no-cache -t budget-frontend:latest ./Frontend
```

3) Zrestartuj deploymenty, aby podciągnęły świeże obrazy z lokalnego rejestru minikube:

```bash
kubectl -n budget-app rollout restart deployment/backend-php
kubectl -n budget-app rollout restart deployment/backend-nginx
kubectl -n budget-app rollout restart deployment/frontend

# Opcjonalnie obserwuj postęp
kubectl -n budget-app rollout status deployment/backend-php
kubectl -n budget-app rollout status deployment/backend-nginx
kubectl -n budget-app rollout status deployment/frontend
```

Alternatywnie możesz ponownie zastosować manifesty (Kustomize):

```bash
kubectl apply -k ./k8s
```

4) Jeżeli zmienił się model/baza danych – uruchom migracje:

```bash
# Jeśli Job już się kiedyś wykonał, usuń go i uruchom ponownie
kubectl -n budget-app delete job/backend-migrations --ignore-not-found
kubectl apply -k ./k8s

# Poczekaj aż migracje zakończą się sukcesem
kubectl -n budget-app wait --for=condition=complete job/backend-migrations --timeout=240s
```

5) Sprawdź stan i zdrowie aplikacji:

```bash
kubectl -n budget-app get pods
curl -u budget-admin:budget-demo http://budget.local/api/healthz
```

### Szybkie scenariusze przebudowy

- Tylko frontend:
  - `eval $(minikube docker-env)`
  - `docker build -t budget-frontend:latest ./Frontend`
  - `kubectl -n budget-app rollout restart deployment/frontend`

- Tylko backend (kod PHP):
  - `eval $(minikube docker-env)`
  - `docker build -t budget-backend-php:latest --target app_php_prod ./Backend`
  - `kubectl -n budget-app rollout restart deployment/backend-php`

- Zmiany w statycznych plikach backendu (konfiguracja Nginx w Backend/docker/nginx):
  - `eval $(minikube docker-env)`
  - `docker build -t budget-backend-nginx:latest --target app_nginx ./Backend`
  - `kubectl -n budget-app rollout restart deployment/backend-nginx`

### Najczęstsze problemy i wskazówki

- Upewnij się, że bieżący terminal „widzi” Docker daemona minikube (`eval $(minikube docker-env)`).
- Jeśli deployment nie pobiera nowego obrazu, zwykle pomaga `rollout restart` lub podbicie tagu obrazu w manifeście.
- Frontend nie wymaga ponownego budowania aplikacji przy zmianie adresu API – adres jest wstrzykiwany w runtime przez `config.js` na podstawie zmiennych środowiskowych kontenera.
- Sprawdź logi przy problemach:

```bash
kubectl -n budget-app logs deploy/backend-php
kubectl -n budget-app logs deploy/backend-nginx
kubectl -n budget-app logs deploy/frontend
```

## Wsparcie AI

Projekt był konsultowany z pomocą narzędzia OpenAI Codex. Szczegóły znajdują się w `AI_CONTEXT.md`.

Obszary, przy których AI pełniło rolę wsparcia technicznego:

- model Doctrine `Backend/src/Entity/BudgetEntry.php`
- kontroler API `Backend/src/Core/Controller/BudgetController.php`
- migracja `Backend/migrations/Version20260530000100.php`
- aplikacja frontendowa w `Frontend/src/`
- skrypty uruchamiania i budowania frontendu w `Frontend/scripts/`
- manifesty Kubernetes w `k8s/`
- Ingress Basic Auth i RBAC
- dokumentacja projektu

Zakres projektu, nazwy, endpointy, konfiguracja runtime frontendu oraz manifesty pod lokalne uruchomienie w minikube były sprawdzane i dopasowywane do wymagań projektu.

## Uwagi do obrony

Najważniejsze elementy do omówienia:

- Ingress kieruje `/api` do backendu, a `/` do frontendu.
- Basic Auth na Ingressie dodaje proste logowanie przed wejściem do aplikacji.
- RBAC rozdziela konto tylko do odczytu `budget-viewer` i konto administracyjne `budget-root`.
- Backend Nginx przekazuje żądania PHP do usługi `php:9000`.
- Migracje bazy danych wykonuje Job `backend-migrations`.
- Frontend nie ma zaszytego adresu backendu w obrazie; odczytuje `API_BASE_URL` z runtime config.
- Dane aplikacji są przechowywane w MySQL StatefulSet z PVC.
