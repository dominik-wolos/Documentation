# Pełny opis projektu budget-app

Ten dokument zbiera pełny opis projektu w jednym miejscu. Jego celem jest ułatwienie nauki przed prezentacją i przygotowanie odpowiedzi na pytania techniczne o aplikację, kontenery oraz konfigurację Kubernetes.

## Temat

Projekt to aplikacja webowa do zarządzania budżetem domowym uruchamiana w lokalnym klastrze Kubernetes.

Użytkownik może:

- przeglądać miesięczne przychody, wydatki i bilans,
- dodawać transakcje,
- edytować transakcje,
- usuwać transakcje,
- przypisywać wpisy do kategorii,
- sprawdzać limity wydatków dla kategorii.

## Najmniejsze elementy aplikacji

Najmniejszą częścią biznesową jest transakcja budżetowa. Transakcja ma tytuł, kwotę, typ (`income` albo `expense`), kategorię, datę i opcjonalną notatkę.

Drugim ważnym elementem jest kategoria. Kategoria ma kod, nazwę i opcjonalny miesięczny limit. Dzięki kategoriom aplikacja może policzyć, ile wydano na przykład na mieszkanie, jedzenie, transport albo inne wydatki.

Z transakcji i kategorii backend wylicza podsumowanie miesiąca: sumę przychodów, sumę wydatków, bilans, stopę oszczędności, największy wydatek i sumy w kategoriach.

## Backend

Backend znajduje się w katalogu `Backend/`. Jest napisany w PHP 8.3 z użyciem Symfony 7.2 i Doctrine ORM.

Najważniejsze pliki backendu:

- `Backend/src/Core/Controller/BudgetController.php` - główny kontroler API.
- `Backend/src/Entity/BudgetEntry.php` - encja transakcji.
- `Backend/src/Entity/Category.php` - encja kategorii.
- `Backend/src/Application/Budget/` - klasy pomocnicze do walidacji, dekodowania JSON, obsługi miesiąca i kategorii.
- `Backend/src/Infrastructure/Budget/DoctrineBudgetEntryRepository.php` - repozytorium Doctrine dla wpisów budżetowych.
- `Backend/migrations/` - migracje bazy danych.

Backend udostępnia API pod ścieżką `/api`:

- `GET /api/healthz` - prosty status działania aplikacji.
- `GET /api/readyz` - sprawdzenie gotowości aplikacji i połączenia z bazą.
- `GET /api/categories` - lista kategorii.
- `GET /api/transactions?month=YYYY-MM` - lista transakcji i podsumowanie miesiąca.
- `POST /api/transactions` - dodanie transakcji.
- `PUT /api/transactions/{id}` - edycja transakcji.
- `DELETE /api/transactions/{id}` - usunięcie transakcji.
- `GET /api/summary?month=YYYY-MM` - samo podsumowanie miesiąca.

## Frontend

Frontend znajduje się w katalogu `Frontend/`. Jest napisany w TypeScript bez ciężkiego frameworka.

Najważniejszy plik to `Frontend/src/main.ts`. Odpowiada za:

- pobranie kategorii z API,
- pobranie transakcji dla wybranego miesiąca,
- renderowanie podsumowania budżetu,
- obsługę formularza dodawania i edycji,
- obsługę usuwania transakcji,
- pokazywanie błędów i statusu połączenia.

Frontend w obrazie Docker jest serwowany przez Nginx. Adres backendu nie jest wpisany na stałe do zbudowanych plików. Kontener generuje plik `config.js` na podstawie zmiennej `API_BASE_URL`.

W Kubernetes `API_BASE_URL=/api`, więc frontend wysyła żądania względnie do tego samego hosta, a Ingress rozdziela ruch między frontend i backend.

## Baza danych

Baza danych to MySQL 8.0. Dane są tworzone przez migracje Doctrine.

W Kubernetes MySQL działa jako `StatefulSet`, ponieważ przechowuje stan. Dane są zapisane w `PersistentVolumeClaim` `mysql-data`. Dzięki temu dane bazy nie są związane wyłącznie z chwilowym życiem pojedynczego poda.

Sekret `mysql-secret` zawiera:

- nazwę bazy,
- użytkownika,
- hasło użytkownika,
- hasło roota,
- `DATABASE_URL` dla Symfony.

## Kontenery

Projekt buduje trzy obrazy:

- `budget-backend-php:latest` - Symfony uruchomione jako PHP-FPM.
- `budget-backend-nginx:latest` - Nginx dla backendu, który przekazuje PHP do service `php`.
- `budget-frontend:latest` - statyczny frontend serwowany przez Nginx.

W minikube obrazy najlepiej budować po wykonaniu:

```bash
eval $(minikube docker-env)
```

Dzięki temu obrazy trafiają do Dockera używanego przez minikube i Kubernetes może je uruchomić bez zewnętrznego registry.

## Kubernetes

Główne manifesty są w katalogu `k8s/`.

Najważniejsze zasoby:

- `Namespace budget-app` - izoluje zasoby projektu.
- `ConfigMap backend-config` - przechowuje niesekretne ustawienia Symfony.
- `Secret mysql-secret` - przechowuje dane połączenia z MySQL.
- `Secret backend-secret` - przechowuje `APP_SECRET`.
- `Secret budget-basic-auth` - przechowuje wpis htpasswd dla Basic Auth.
- `Service mysql` - daje stabilny adres `mysql:3306`.
- `Service mysql-headless` - wymagany przez StatefulSet jako stabilna usługa sieciowa.
- `StatefulSet mysql` - uruchamia bazę danych.
- `Deployment backend-php` - uruchamia PHP-FPM.
- `Service php` - pozwala backendowemu Nginx połączyć się z PHP-FPM przez `php:9000`.
- `Deployment backend-nginx` - obsługuje HTTP backendu.
- `Service backend` - wystawia backend wewnątrz klastra.
- `Job backend-migrations` - uruchamia migracje Doctrine.
- `Deployment frontend` - uruchamia frontend.
- `Service frontend` - wystawia frontend wewnątrz klastra.
- `Ingress budget-ingress` - kieruje ruch z `budget.local`.
- `ServiceAccount budget-viewer` - konto tylko do odczytu.
- `ServiceAccount budget-root` - konto administracyjne w namespace aplikacji.
- `Role` i `RoleBinding` - definiują autoryzację operacyjną w Kubernetes.

## Przepływ żądania

Typowe żądanie użytkownika przechodzi tak:

1. Użytkownik otwiera `http://budget.local`.
2. NGINX Ingress Controller sprawdza Basic Auth.
3. Ingress kieruje `/` do service `frontend`.
4. Service `frontend` kieruje ruch do poda frontendu.
5. Frontend pobiera dane z `/api`.
6. Ingress kieruje `/api` do service `backend`.
7. Service `backend` kieruje ruch do poda `backend-nginx`.
8. Backendowy Nginx przekazuje żądanie PHP do service `php`.
9. Service `php` kieruje ruch do poda `backend-php`.
10. Symfony wykonuje logikę aplikacji.
11. Doctrine odczytuje lub zapisuje dane w MySQL przez service `mysql`.

## Logowanie i autoryzacja infrastrukturalna

Projekt ma dwa poziomy zabezpieczeń infrastrukturalnych.

Pierwszy poziom to Basic Auth na Ingressie. Użytkownik wchodzący przez `budget.local` musi podać login i hasło:

- login: `budget-admin`
- hasło: `budget-demo`

To nie jest aplikacyjny system kont użytkowników. To zabezpieczenie wejścia HTTP na poziomie NGINX Ingress Controller.

Drugi poziom to RBAC w Kubernetes. Projekt definiuje dwa konta service account:

- `budget-viewer` - może podglądać zasoby aplikacji, ale nie może ich zmieniać.
- `budget-root` - może zarządzać zasobami w namespace `budget-app`.

RBAC służy do kontroli tego, kto lub co może wykonywać operacje administracyjne w klastrze.

## Helm

Helm Chart znajduje się w `helm/budget-app/`. Zawiera szablony tych samych głównych elementów aplikacji, które są opisane w manifestach `k8s/`.

Kustomize jest podstawowym sposobem uruchamiania projektu:

```bash
kubectl apply -k ./k8s
```

Helm można wykorzystać do renderowania albo alternatywnej instalacji:

```bash
helm template budget-app ./helm/budget-app
helm install budget-app ./helm/budget-app --namespace budget-app --create-namespace
```

## Makefile

`Makefile` skraca najczęstsze operacje:

- `make up-default` - pełne wdrożenie: minikube, obrazy, manifesty Kustomize, oczekiwanie na gotowość.
- `make check` - weryfikuje stan całego stosu (11 punktów: infrastruktura, pody, HTTP).
- `make build-images` - buduje obrazy w Dockerze minikube.
- `make apply` - aplikuje manifesty z `k8s/`.
- `make wait` - czeka na MySQL, migracje i deploymenty.
- `make status` - pokazuje zasoby w namespace.
- `make port-forward` - wystawia frontend lokalnie na `127.0.0.1:18081`.
- `make minikube-tunnel` - uruchamia tunel (wymagany na macOS + Docker driver dla Ingress).
- `make smoke-port-forward` - testuje API przez port-forward.
- `make smoke-k8s` - testuje API przez Ingress z Basic Auth.
- `make scale-frontend REPLICAS=2` - skaluje frontend.
- `make kill-frontend-pod` - usuwa jeden pod frontendu, żeby pokazać samonaprawianie.
- `make auth-info` - pokazuje dane demonstracyjne Basic Auth.
- `make rbac-check` - sprawdza konto tylko do odczytu.
- `make rbac-root-check` - sprawdza konto administracyjne w namespace.
- `make rerun-migrations` - uruchamia migracje ponownie.
- `make validate` - renderuje Kustomize i opcjonalnie Helm.

## Najważniejsze odpowiedzi techniczne

Frontend i backend są rozdzielone, bo frontend jest statyczny i może być serwowany przez Nginx, a backend wykonuje logikę aplikacji i komunikuje się z bazą.

MySQL działa jako StatefulSet, bo przechowuje dane. Deployment jest lepszy dla usług bezstanowych, takich jak frontend i backend.

Service jest potrzebny, bo pody mają zmienne adresy IP. Service daje stabilną nazwę DNS, na przykład `mysql`, `backend`, `frontend` albo `php`.

Ingress jest potrzebny, żeby użytkownik mógł wejść przez jeden adres HTTP, a Kubernetes rozdzielił ruch między frontend i backend.

Job migracji jest osobnym zasobem, bo migracje mają wykonać się raz i zakończyć. Nie powinny działać stale jak Deployment.

ConfigMap przechowuje konfigurację jawną, a Secret przechowuje hasła i dane wrażliwe.

Basic Auth na Ingressie zabezpiecza wejście HTTP przed aplikacją. Dzięki temu można pokazać logowanie bez przebudowy backendu.

RBAC kontroluje uprawnienia do operacji w Kubernetes. `budget-viewer` pokazuje zasadę minimalnych uprawnień, a `budget-root` pokazuje konto z pełnymi uprawnieniami w namespace projektu.
