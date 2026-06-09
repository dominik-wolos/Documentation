# Opis aplikacji do zarządzania budżetem domowym

Ten plik opisuje, co znajduje się w projekcie, jak elementy są ze sobą połączone i co warto umieć wyjaśnić podczas obrony.

## Punkt wyjścia

Punktem wyjścia był backend PHP/Symfony uruchamiany w kontenerach. Rozszerzyłem go o kompletną aplikację budżetu domowego: API REST, model danych, migracje bazy, frontend TypeScript oraz manifesty Kubernetes dla lokalnego klastra minikube.

## Co robi aplikacja

Aplikacja służy do prowadzenia prostego budżetu domowego. Użytkownik widzi miesięczny bilans, przychody, wydatki, stopę oszczędności, limity kategorii oraz listę transakcji. Może dodać nową transakcję, edytować istniejącą pozycję, usunąć ją i odświeżyć dane z backendu.

Domyślne dane demonstracyjne obejmują wpisy z czerwca 2026 roku: wynagrodzenie, czynsz i media, zakupy spożywcze oraz bilet miesięczny. Dzięki temu po uruchomieniu projektu od razu widać działający panel bez ręcznego wprowadzania danych.

## Architektura

Projekt składa się z czterech głównych warstw:

- Frontend w `Frontend/` to aplikacja napisana w TypeScript, serwowana przez Nginx.
- Backend w `Backend/` to aplikacja Symfony z API REST.
- Baza danych to MySQL 8 uruchomiony jako StatefulSet w Kubernetes.
- Infrastruktura w `k8s/` opisuje namespace, konfigurację, sekrety, deploymenty, service, job migracji, ingress, Basic Auth, RBAC, limity zasobów i bazę danych.
- Helm Chart w `helm/budget-app/` pozwala wyrenderować lub zainstalować aplikację przez Helm.

Przepływ żądania wygląda tak:

1. Użytkownik otwiera frontend.
2. NGINX Ingress Controller sprawdza Basic Auth.
3. Frontend pobiera konfigurację runtime z `config.js`.
4. Frontend wysyła żądania do `/api`.
5. Nginx frontendu lub Ingress przekazuje `/api` do service `backend`.
6. Service `backend` kieruje ruch do poda `backend-nginx`.
7. Backendowy Nginx przekazuje żądania PHP do service `php` na port `9000`.
8. Service `php` kieruje ruch do poda `backend-php`.
9. Symfony obsługuje endpoint, korzystając z Doctrine i połączenia `DATABASE_URL`.
10. Doctrine czyta lub zapisuje dane w service `mysql`, który wskazuje na pod MySQL.

## Backend

Backend udostępnia API REST pod ścieżką `/api`. Najważniejsze endpointy:

- `GET /api/healthz` zwraca prosty status działania backendu.
- `GET /api/readyz` sprawdza gotowość aplikacji, w tym dostęp do bazy.
- `GET /api/categories` zwraca kategorie budżetowe i ich limity.
- `GET /api/transactions?month=YYYY-MM` zwraca transakcje i podsumowanie miesiąca.
- `POST /api/transactions` dodaje nową transakcję.
- `PUT /api/transactions/{id}` aktualizuje istniejącą transakcję.
- `DELETE /api/transactions/{id}` usuwa transakcję.
- `GET /api/summary?month=YYYY-MM` zwraca samo podsumowanie miesiąca.

Backend korzysta z Doctrine ORM. Dane są modelowane przez encje transakcji i kategorii. Migracje tworzą tabele, wypełniają kategorie oraz dodają dane demonstracyjne.

## Frontend

Frontend jest statyczną aplikacją TypeScript. Nie ma zaszytego adresu backendu w kodzie obrazu. Adres API jest wstrzykiwany w runtime przez zmienną `API_BASE_URL`, a skrypt startowy kontenera generuje plik `config.js`.

W Kubernetes `API_BASE_URL` ma wartość `/api`, dzięki czemu frontend działa zarówno przez Ingress `budget.local`, jak i przez port-forward na przykład `http://127.0.0.1:18081/`.

Ekran aplikacji pokazuje:

- kafelki z przychodami, wydatkami, bilansem i stopą oszczędności,
- formularz dodawania transakcji,
- limity miesięczne kategorii,
- listę transakcji z przyciskami edycji i usuwania,
- informację o runtime API.

## Baza danych

Baza danych działa jako StatefulSet `mysql`. Jest to lepszy wybór niż zwykły Deployment, ponieważ baza przechowuje stan. Dane MySQL są zapisane w PersistentVolumeClaim `mysql-data`, montowanym w kontenerze pod ścieżką `/var/lib/mysql`.

Sekret `mysql-secret` zawiera nazwę bazy, użytkownika, hasła i `DATABASE_URL` dla Symfony.

## Kubernetes

Najważniejsze zasoby:

- `Namespace budget-app` izoluje zasoby projektu.
- `ConfigMap backend-config` przechowuje niesekretne ustawienia Symfony, CORS i strefę czasową.
- `Secret mysql-secret` przechowuje konfigurację bazy.
- `Secret backend-secret` przechowuje `APP_SECRET`.
- `Secret budget-basic-auth` przechowuje wpis htpasswd dla Basic Auth na Ingressie.
- `StatefulSet mysql` uruchamia bazę danych z trwałym wolumenem.
- `Deployment backend-php` uruchamia Symfony PHP-FPM.
- `Service php` daje backendowemu Nginx stabilną nazwę `php:9000`.
- `Deployment backend-nginx` obsługuje HTTP dla backendu i przekazuje PHP do service `php`.
- `Service backend` kieruje ruch HTTP do backendowego Nginx.
- `Job backend-migrations` wykonuje migracje Doctrine po starcie środowiska.
- `Deployment frontend` uruchamia statyczny frontend przez Nginx.
- `Service frontend` kieruje ruch do frontendu.
- `Ingress budget-ingress` rozdziela ruch: `/api` i `/api-platform` do backendu, a `/` do frontendu.
- `ServiceAccount`, `Role` i `RoleBinding` tworzą RBAC dla kont `budget-viewer` i `budget-root`.
- `resources.requests` i `resources.limits` ograniczają oraz rezerwują CPU i pamięć dla kontenerów.
- `Service mysql-headless` pokazuje mechanizm headless service dla aplikacji stanowych.

## Jak sprawdzić, że działa

Pełna weryfikacja jednym poleceniem:

```bash
make check
```

Sprawdza minikube, kontroler Ingress, adres Ingress, MySQL, migracje, wszystkie deploymenty i dostępność HTTP. Wynik 11/11 PASS oznacza, że cały stos działa.

Ręczny podstawowy status:

```bash
kubectl -n budget-app get pods
kubectl -n budget-app get deployments,svc,job
kubectl auth can-i list pods -n budget-app --as=system:serviceaccount:budget-app:budget-viewer
kubectl auth can-i delete pods -n budget-app --as=system:serviceaccount:budget-app:budget-viewer
kubectl auth can-i '*' '*' -n budget-app --as=system:serviceaccount:budget-app:budget-root
```

Test przez port-forward:

```bash
make port-forward   # osobny terminal
curl http://127.0.0.1:18081/api/healthz
make smoke-port-forward
```

Test przez Ingress (wymaga tunelu i wpisu w /etc/hosts):

```bash
make minikube-tunnel   # osobny terminal
curl -u budget-admin:budget-demo http://budget.local/api/healthz
make smoke-k8s
```

Oczekiwana odpowiedź:

```json
{"status":"ok","service":"budget-backend"}
```

## Co powiedzieć na obronie

Najważniejsza myśl: aplikacja jest podzielona na frontend, backend, bazę danych i manifesty Kubernetes. Frontend nie łączy się bezpośrednio z podami, tylko używa ścieżki `/api`. Kubernetes Service zapewnia stabilne nazwy i porty, a Deploymenty pilnują replik. Baza danych jest StatefulSetem, bo przechowuje dane. Migracje są osobnym Jobem, bo mają wykonać się raz i zakończyć. Dostęp przez Ingress jest zabezpieczony Basic Auth, a operacje w klastrze są rozdzielone przez RBAC.

Warto też pokazać komendy z `Makefile`, na przykład skalowanie frontendu:

```bash
make scale-frontend REPLICAS=2
make pods
make scale-down
```

Oraz symulację awarii:

```bash
make kill-frontend-pod
make pods
```

Po usunięciu poda Kubernetes automatycznie utworzy nowy, ponieważ Deployment pilnuje zadanej liczby replik.

Na końcu można pokazać Helm jako alternatywny sposób renderowania tych samych zasobów aplikacji:

```bash
make helm-template
```
