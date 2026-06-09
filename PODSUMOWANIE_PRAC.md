# Podsumowanie wykonanych prac

Data przygotowania: 02.06.2026

## Punkt wyjściowy

Projekt startował jako podstawowy szkielet aplikacji backendowej w Symfony umieszczony w katalogu `Backend/`.

Na początku w projekcie znajdowały się głównie:

- konfiguracja Symfony,
- pliki Dockerfile oraz `docker-compose.yml`,
- konfiguracja Doctrine, API Platform, CORS, Mailera i innych pakietów,
- pojedynczy testowy kontroler `GreetingController`, który zwracał prostą odpowiedź `Hello World!`,
- brak właściwej logiki biznesowej aplikacji,
- brak frontendu,
- brak manifestów Kubernetes,
- brak dokumentacji wymaganej pod projekt zaliczeniowy,
- brak kompletnego uruchomienia aplikacji jako spójnego systemu frontend + backend + baza danych.

Projekt nie był jeszcze aplikacją do zarządzania budżetem domowym. Był raczej przygotowanym szkieletem technologicznym, który wymagał doprojektowania tematu, funkcjonalności, warstwy prezentacji i infrastruktury.

## Co zostało wykonane

### 1. Przygotowanie właściwego tematu aplikacji

Projekt został przekształcony w aplikację webową na temat:

`Aplikacja do zarządzania budżetem domowym`

Aplikacja pozwala użytkownikowi:

- przeglądać miesięczne przychody i wydatki,
- dodawać transakcje,
- edytować istniejące transakcje,
- usuwać transakcje,
- przypisywać transakcje do kategorii,
- kontrolować limity wydatków dla kategorii,
- sprawdzać bilans miesiąca i stopę oszczędności.

### 2. Backend Symfony

W backendzie dodano właściwą logikę aplikacji budżetowej.

Najważniejsze wykonane elementy:

- dodano encję `BudgetEntry` reprezentującą transakcję budżetową,
- dodano encję `Category` reprezentującą kategorię budżetową,
- dodano kontroler `BudgetController`,
- dodano endpointy REST dla frontendu,
- dodano warstwę aplikacyjną dla walidacji, dekodowania JSON, kategorii i zakresu miesięcznego,
- dodano repozytorium domenowe dla wpisów budżetowych,
- usunięto testowy kontroler powitalny,
- przeniesiono wygenerowane endpointy API Platform pod `/api-platform`, aby nie kolidowały z głównym API aplikacji.

Najważniejsze endpointy aplikacji:

- `GET /api/healthz`
- `GET /api/readyz`
- `GET /api/categories`
- `GET /api/transactions?month=YYYY-MM`
- `POST /api/transactions`
- `PUT /api/transactions/{id}`
- `DELETE /api/transactions/{id}`
- `GET /api/summary?month=YYYY-MM`

### 3. Baza danych i migracje

Dodano migracje Doctrine, które przygotowują bazę danych dla aplikacji.

W migracjach utworzono:

- tabelę `budget_entries`,
- tabelę `categories`,
- relację transakcji z kategoriami,
- przykładowe dane demonstracyjne dla maja 2026,
- przykładowe dane demonstracyjne dla czerwca 2026, aby aktualny widok aplikacji po uruchomieniu od razu pokazywał sensowne dane.

Dzięki temu po uruchomieniu migracji aplikacja ma gotowy zestaw danych do prezentacji podczas obrony.

### 4. Frontend TypeScript

Obok backendu utworzono osobną aplikację frontendową w katalogu `Frontend/`.

Frontend został napisany w TypeScript bez ciężkiego frameworka, aby był prosty do uruchomienia i wyjaśnienia.

Frontend zawiera:

- panel podsumowania budżetu miesięcznego,
- formularz dodawania transakcji,
- obsługę edycji i usuwania transakcji,
- listę transakcji,
- listę kategorii i limitów miesięcznych,
- konfigurację adresu API,
- responsywny układ działający na mniejszych ekranach.

Frontend komunikuje się z backendem przez:

`http://localhost:8000/api`

W lokalnym środowisku frontend działa pod adresem:

`http://127.0.0.1:5173`

### 5. Uruchomienie lokalne

Projekt został połączony lokalnie jako działający zestaw:

- backend Symfony działa w kontenerach Docker Compose,
- backend Nginx jest wystawiony na porcie `8000`,
- MySQL działa w kontenerze i przechowuje dane budżetu,
- frontend TypeScript działa lokalnie na porcie `5173`,
- frontend pobiera dane z backendu przez `http://localhost:8000/api`.

Podczas prac naprawiono problemy uruchomieniowe:

- brakujący plik `vendor/autoload_runtime.php`,
- nieaktualny cache kontenera Symfony,
- brakujące migracje,
- konflikt tras między API Platform i ręcznym API aplikacji,
- rozbieżność między strukturą bazy danych a encjami Doctrine,
- konieczność używania portu `8000`, ponieważ port `80` był zajęty przez inną usługę.

### 6. Kubernetes i minikube

Dodano komplet manifestów Kubernetes w katalogu `k8s/`.

Manifesty obejmują:

- `Namespace`,
- `Secret` dla danych MySQL,
- `ConfigMap` dla konfiguracji backendu,
- `StatefulSet` i `Service` dla MySQL,
- `Deployment` i `Service` dla backendu PHP,
- `Deployment` i `Service` dla backendu Nginx,
- `Job` wykonujący migracje Doctrine,
- `Deployment` i `Service` dla frontendu,
- `Ingress` kierujący ruch do frontendu i backendu,
- Basic Auth na Ingressie,
- RBAC dla kont operacyjnych `budget-viewer` i `budget-root`,
- Helm Chart jako alternatywny sposób wdrożenia.

Routing w Kubernetes:

- `/` kieruje do frontendu,
- `/api` kieruje do backendowego API aplikacji,
- `/api-platform` kieruje do wygenerowanego API Platform.

Dostęp przez `budget.local` jest zabezpieczony przez Basic Auth na poziomie NGINX Ingress Controller. Konto `budget-viewer` ma uprawnienia tylko do odczytu zasobów w namespace, a konto `budget-root` ma pełne uprawnienia administracyjne w namespace `budget-app`.

### 7. Dokumentacja

Dodano główny plik `README.md` w katalogu głównym projektu.

README zawiera:

- krótki opis aplikacji,
- opis technologii,
- strukturę projektu,
- listę endpointów,
- instrukcję uruchomienia w minikube,
- instrukcję budowania obrazów Docker,
- opis Basic Auth, RBAC i Helm,
- opis użycia sztucznej inteligencji,
- opis obszarów konsultowanych z pomocą AI,
- uwagi przydatne podczas obrony projektu.

Dodano również plik:

`AI_CONTEXT.md`

Plik ten pełni rolę kontekstu rozmowy z LLM, zgodnie z wymaganiem projektu dotyczącym udokumentowania użycia sztucznej inteligencji.

## Aktualny stan projektu

Aplikacja działa w pełni w lokalnym klastrze Kubernetes (minikube). Pojedyncze polecenie `make up-default` buduje obrazy, aplikuje manifesty i czeka na gotowość całego stosu.

Dostęp do aplikacji:

- przez port-forward: `make port-forward`, następnie `http://127.0.0.1:18081`,
- przez Ingress z tunelem: `make minikube-tunnel` (osobny terminal), następnie `http://budget.local` (login: `budget-admin`, hasło: `budget-demo`).

Weryfikacja stanu środowiska:

```bash
make check
```

Komenda sprawdza 11 punktów: minikube, kontroler Ingress, adres Ingress, MySQL, migracje, wszystkie deploymenty oraz dostępność HTTP frontendu i API.

Smoke test API:

```bash
make smoke-port-forward   # przez port-forward
make smoke-k8s            # przez Ingress z Basic Auth
```

Sprawdzono i zweryfikowano:

- kompletne wdrożenie przez `make up-default`,
- działanie endpointów API (healthz, readyz, categories, transactions, summary),
- działanie frontendu w przeglądarce przez port-forward,
- dostęp przez Ingress z Basic Auth przez `minikube tunnel`,
- skalowanie deploymentów (`make scale-frontend REPLICAS=2`),
- samonaprawianie po `make kill-frontend-pod`,
- działanie RBAC (`make rbac-check`, `make rbac-root-check`),
- renderowanie manifestów Kustomize i Helm (`make validate`, `make helm-template`),
- automatyczny test `make check` — wynik 11/11 PASS.

## Najważniejsze pliki dodane lub zmienione

- `Backend/src/Core/Controller/BudgetController.php`
- `Backend/src/Entity/BudgetEntry.php`
- `Backend/src/Entity/Category.php`
- `Backend/src/Application/Budget/`
- `Backend/src/Domain/Budget/`
- `Backend/src/Infrastructure/Budget/`
- `Backend/migrations/`
- `Frontend/src/main.ts`
- `Frontend/src/styles.css`
- `Frontend/Dockerfile`
- `Frontend/nginx/default.conf`
- `k8s/`
- `helm/budget-app/`
- `README.md`
- `AI_CONTEXT.md`

## Podsumowanie

Punktem wyjściowym był prosty szkielet Symfony z jednym testowym endpointem. W ramach prac przygotowano kompletną aplikację webową do zarządzania budżetem domowym, składającą się z backendu, frontendu, bazy danych, manifestów Kubernetes oraz dokumentacji wymaganej w projekcie zaliczeniowym.

Projekt działa end-to-end w Kubernetes: `make up-default` uruchamia cały stos, `make check` weryfikuje jego stan, a `make smoke-port-forward` lub `make smoke-k8s` przeprowadzają pełny test API. Projekt jest gotowy do prezentacji.
