# Audyt zakresu konfiguracji Kubernetes

Ten dokument zostaje w projekcie jako notatka porządkująca zakres po redukcji konfiguracji. Nie opisuje osobnej prezentacji materiału laboratoryjnego. Jego celem jest pokazanie, które elementy Kubernetes są realnie potrzebne do działania `budget-app`, a które zostały usunięte z aktywnego wdrożenia jako nadmiarowe.

## Aktualny zakres

Aktywna konfiguracja Kubernetes znajduje się w `k8s/` i obejmuje tylko zasoby potrzebne aplikacji budżetowej:

- namespace projektu,
- sekrety i konfigurację backendu,
- bazę MySQL,
- backend PHP-FPM,
- backend Nginx,
- frontend,
- migracje Doctrine jako Job,
- service dla komunikacji wewnątrz klastra,
- Ingress dla ruchu HTTP,
- Basic Auth na Ingressie,
- RBAC dla kont operacyjnych,
- requests i limits dla kontenerów.

## Elementy zostawione w aktywnym wdrożeniu

`Namespace` został, bo oddziela zasoby projektu.

`ConfigMap` została, bo przechowuje jawną konfigurację Symfony i środowiska.

`Secret` został, bo aplikacja potrzebuje haseł i `DATABASE_URL`.

`Deployment` został dla frontendu i backendu, ponieważ są to elementy bezstanowe.

`StatefulSet` został dla MySQL, ponieważ baza danych przechowuje stan.

`PersistentVolumeClaim` został, ponieważ MySQL musi mieć trwały katalog danych.

`Service` został, ponieważ pody mają zmienne adresy IP i potrzebują stabilnych nazw DNS.

`Ingress` został, ponieważ aplikacja ma być dostępna jako jedna usługa HTTP pod `budget.local`.

`Basic Auth` został dodany, ponieważ temat pracy dotyczy infrastruktury wokół aplikacji. Logowanie na Ingressie zabezpiecza wejście do systemu bez rozbudowywania backendu o konta użytkowników.

`RBAC` został dodany, ponieważ pozwala pokazać autoryzację operacyjną w Kubernetes. Konto `budget-viewer` ma uprawnienia tylko do odczytu, a `budget-root` ma pełne uprawnienia w namespace aplikacji.

`Job` został, ponieważ migracje bazy są zadaniem jednorazowym.

`resources.requests` i `resources.limits` zostały, ponieważ pomagają kontrolować użycie CPU i pamięci.

## Elementy usunięte z aktywnego wdrożenia

Z aktywnej konfiguracji usunięto elementy, które nie były potrzebne do działania aplikacji:

- osobne manifesty demonstracyjne,
- dodatkowe service typu NodePort,
- dodatkowe przykładowe Ingressy,
- DaemonSet niezwiązany z aplikacją,
- ręczne PV/PVC tworzone wyłącznie jako przykład.

Po redukcji projekt jest łatwiejszy do wyjaśnienia: każdy aktywny zasób ma bezpośredni związek z uruchomieniem, zabezpieczeniem albo utrzymaniem `budget-app`.

## Stan dokumentacji

Pliki PDF pozostają w repozytorium jako materiały źródłowe, ale aktywna dokumentacja projektu opisuje tylko działającą aplikację i jej konfigurację.

Najważniejsze dokumenty:

- `README.md` - uruchomienie i główne informacje.
- `docs/OPIS_APLIKACJI.md` - opis architektury.
- `docs/INSTRUKCJA_ZARZADZANIA.md` - operacje administracyjne.
- `docs/PROJECT_FULL_DESCRIPTION.md` - pełne wyjaśnienie techniczne.
- `docs/PROJECT_LEARNING_SUMMARY.md` - krótka ścieżka nauki.

## Wniosek

Aktualny projekt jest skupiony na jednej rzeczy: działającej aplikacji budżetowej uruchamianej w Kubernetes. Konfiguracja jest lżejsza, bo nie zawiera zasobów niezwiązanych z aplikacją, ale obejmuje elementy ważne dla infrastruktury: Ingress, Basic Auth, RBAC, Helm, trwałą bazę danych i migracje.
