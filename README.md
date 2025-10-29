# Laboratorium 4: Debugowanie i Zarządzanie Zasobami w Kubernetes

Repozytorium zawiera pliki konfiguracyjne YAML wykorzystane podczas Laboratorium nr 4 z przedmiotu **Programowanie Full-Stack w Chmurze Obliczeniowej**[cite: 5, 6].

[cite_start]Celem ćwiczeń było zapoznanie się z podstawowymi metodami postępowania w przypadku wystąpienia błędu w zasobach typu `Pod` oraz z mechanizmami zarządzania zasobami (CPU/RAM) w klastrze Kubernetes[cite: 8, 259].

## 🔬 Część 1: Postępowanie w przypadku wystąpienia błędu

Ta część laboratorium koncentrowała się na diagnozowaniu i rozwiązywaniu typowych problemów z Pod'ami.

### `incorrect.yaml`
* [cite_start]**Problem**: Pod po uruchomieniu przechodził w stan `CrashLoopBackOff`[cite: 90].
* [cite_start]**Diagnoza**: Użycie polecenia `kubectl logs incorrect-pod` wykazało przyczynę błędu[cite: 92]:
    ```bash
    /bin/sh: unknown: not found
    ```
* [cite_start]**Wniosek**: Kontener próbował wykonać nieistniejącą komendę `unknown`[cite: 108].

### `failing.yaml`
* [cite_start]**Problem**: Pod znajdował się w stanie `Running`[cite: 128], jednak aplikacja wewnątrz nie działała poprawnie.
* [cite_start]**Diagnoza**: Logi (`kubectl logs failing-pod`) ujawniły błąd[cite: 131]:
    ```bash
    /bin/sh: can't create /root/tmp/curr-date.txt: nonexistent directory
    ```
* [cite_start]**Naprawa**: Problem rozwiązano "na żywo" przez wejście do kontenera (`kubectl exec -it failing-pod -- /bin/sh`) [cite: 137-138] [cite_start]i ręczne utworzenie brakującego katalogu (`mkdir -p ~/tmp`)[cite: 139].

### `minimal.yaml`
* **Problem**: Trudność w debugowaniu kontenera typu **"distroless"** (minimalistycznego), który nie zawiera powłoki `/bin/sh`.
* **Diagnoza**: Standardowa próba wejścia (`kubectl exec`) zakończyła się niepowodzeniem i błędem `OCI runtime exec failed... no such file or directory`.
* **Rozwiązanie**: Użycie polecenia `kubectl debug` w celu dołączenia "efemerycznego" kontenera debugującego z obrazu `busybox`, co pozwoliło na inspekcję środowiska.

---

## 📊 Część 2: Zarządzanie zasobami (CPU/RAM)

Ta część dotyczyła alokacji i limitowania zasobów dla kontenerów.

### `appresources.yaml`
* [cite_start]**Cel**: Uruchomienie Poda z dwoma kontenerami (`db` i `wp`), z których każdy ma zdefiniowane `requests` (zasoby gwarantowane) oraz `limits` (limity zasobów)[cite: 261].
* [cite_start]**Scenariusz błędu**: Po modyfikacji pliku i zmianie żądania pamięci na `requests: memory: "64Gi"`[cite: 275], Pod "utknął" w stanie `Pending`.
* **Diagnoza**: Polecenie `kubectl describe pod myapp` wykazało w sekcji `Events` błąd `FailedScheduling` z powodu `Insufficient memory`. Klaster (Minikube) nie posiadał wystarczającej ilości wolnej pamięci, aby zagwarantować 64GiB.

### `ResourceQuota` (LimitRange)
* **Cel**: Ograniczenie całkowitego zużycia zasobów w obrębie `namespace`.
* **Kroki**:
    1.  Utworzono nowy `namespace` o nazwie `restricted`.
    2.  Utworzono obiekt `ResourceQuota` o nazwie `xlimits` (limit: `pods=3`, `cpu=2`, `memory=1G`).
* **Test 1 (Negatywny)**: Próba uruchomienia Poda `nginx` bez zdefiniowanych limitów (`kubectl run limiter1...`) zakończyła się błędem `Forbidden... must specify cpu, memory`. Dowodzi to, że Quota wymusza deklarowanie zasobów.
* **Test 2 (Pozytywny)**: Uruchomienie Poda `myapp` (z pliku `appresources.yaml`) w `namespace restricted` powiodło się, ponieważ posiadał on zdefiniowane zasoby, które mieściły się w ustalonych limitach.
