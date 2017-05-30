# aplikacje.gov.pl – Eksperyment Ansible
## Cel eksperymentu
Eksperyment polega na stworzeniu architektury (playbooka i ról) [Ansible](https://www.ansible.com/) umożliwiającej uruchomienie dockerowego kontenera z serwerem Nginx, który serwuje plik index.html z treścią konfigurowalną przez użytkownika.

**Celem eksperymentu jest wydanie rekomendacji dotyczącej wykorzystania programu Ansible w procesie konfiguracji Instancji.**


#### Pytania eksperymentalne:

1. Jak będzie wyglądać intergracja Ansible z webowym UI konfiguratora?
    - Jakie są różnice pomiędzy powyższym sposobem integracji a modelem aktualnie realizowanym (który jest nakładką na docker/docker-compose)?
    - Czy i jak zmienią się aktualnie używane w konfiguratorze modele?
2. Czy Ansible będzie się umiał zająć zarządzaniem użytkownikami w BD?
3. Wykorzystując docker-compose mamy problem z tym, że modyfikacja jednego serwisu wymaga restartu wszystkich. (docker-compose down; create new compose file; docker-compose up; ) Czy Ansible jest pod tym względem lepsze?
4. Jak można się komunikować z Ansible? Pisze sie skrypty w plikach czy jest jakieś inne API?
5. Czy plik parametryzacyjny (*parameters.json*) można zastąpić odpowiednim Playbookiem bądź Rolą?

#### Efekty:

1. Rekomendacja odnoście użycia programu Ansible przy tworzeniu konfiguratora instancjii.
2. Zarys części architektury konfiguratora wykorzystującej Ansible (w przypadku pozytywnej decyzji).

## Przebieg eksperymentu

### O Ansible (teoretycznie)

Ansible jest narzędziem umożliwiającym automatyzację i orchiestryzację zadań związanych ze zdalnym zarządzaniem serwerami.

Szczególnie warto zwrócić uwagę na dwie jego cechy:

- Deklaratywne definiowanie pożądanego stanu systemu (*CO* zamiast *JAK*)
- Idempotencję (wiele wywołań idempotentnej funkcji ma taki sam efekt jak jej pojedyncze wywołanie). Oznacza to, że jeżeli oczekiwany przez użytkownika stan jest już na serwerze osiągnięty, stan tego serwera nie ulegnie zmianie.

Zalety Ansible:

- Czystość:
    - Brak agentów
    - Brak bazy danych
    - Brak dodatkowego oprogramowania
    - Brak skomplikowanych uaktualnień
- YAML
    - Przyjazna i przejrzysta składnia (*YAML Ain't a Markup Language*)
    - Nie wymaga doświadczenia programistycznego
- Wbudowane zabezpieczenia ([Ansible Vault](http://docs.ansible.com/ansible/playbooks_vault.html))
- Modularność => rozszerzalność

#### Architektura

Warto w tym miejscu wprowadzić rozróżnienie na serwer kontrolujący i serwer zdalny. Pierwszy jest maszyną, która zawiera katalog z plikami i inicjuję Ansible; drugi jest maszyną, na której zachodzą procesy kontrolowane przez Ansible.

**Wymagania systemowe:**

-  serwer kontrolujący:
    - Python 2.6+
    - System operacyjny typu \*NIX (Linux/Unix/Mac). 
- serwer zdalny:
    - *NIX: 
        - Python2.5+ 
        - SSH
    - Windows:
        - Remote Powershell


#### Proces egzekucji

Pliki `.yml` definiujące zadania, które mają być wykonane na wyspecyfikowanych serwerach zdalnych tłumaczone są na paczkę języka Python która następnie transferowana jest po SSH na serwer zdalny tam zapisywane w folderze /tmp i wykonywana. Po wykonaniu usuwane są wszystkie Ansiblowe pliki z serwera zdalnego i zwracany jest raport z egzekucji. W przypadku wystąpienia błędów egzekucja paczki jest przerywana.

Ansible wywoływać można także z wiersza poleceń.

**Python 3**

W przypadku, gdy serwer lokalny domyślnie wykonuje Pythona 3+ można wyspecyfikować w definicji *hosta* ścieżkę do odpowiedniego interpreta.
Warto też dodać, że w wersji 2.2 prezentowany jest już wstępny support dla Pythona 3, za czym można przewidywać, że niedługo Ansible będzie Python *agnostic*.

#### Inwentarz

Inwenatarz to plik (typowe nazywany `hosts` lub `inventory`), w którym specyfikuje się zdalne maszyny, na których wykonywane będą operacje.

Przewidziana przez Ansible syntaktyka pozwala na złożone grupowanie serwerów i pzypisywanie im zmiennych.

#### Konfiguracja

W pliku konfiguracyjnym `ansible.cfg` wyspecyfikować można globalne ustawienia np. wskzać sćieżkę do Inwentarza. 

#### Moduły

Moduły wykorzystywane są do realizacji konkretnych zadań na serwerze zdalnym. Np. moduł `apt` służy do zarządzania programami na serwerze debianowym. Np. aby upewnić się, że serwer Apache jest zainstalowany:


```yaml
tasks:
  - name: Ensure Apache is installed
    apt: name=httpd state=present
```

Wbudowanych w Ansible jest ponad 1000 modułów. Można także pisać własne. 

#### Playbooki

Klucz `tasks` powyżej jest częścią większego pliku, który nazywany jest *Playbookiem*. Playbook składa się z zagrywek (ang. *plays*), z których każda zaadresowana jest do wybranych *hostów* (serwerów zdalnych) i składa się z jednego bądź więcej zadań (ang. *tasks*). Każde zadanie korzysta z jednego modułu, który przyjmuje pewne argumenty. Oprócz tego *taski* mogą powiadamiać *handlery*, które wykonają się pod koniec playbooka. Np. gdy nasza zagrywka składa się z wprowadzania zmian w plikach konfiguracyjnych, chcielibyśmy upewnić się, że Apache zostanie zrestartowane. 

Przykładowy *playbook* składający się z jednej zagrywki i czterech zadań wygląda tak:

```yaml
- name: Deploy site with Apache
  hosts: webservers
  become: yes
  become_metho: sudo
  vars:
    doc_dir: /ansible/
    doc_root: /var/www/html/
    http_port: 80
    max_clients: 5
    user_name: Guest
  
  tasks:
  - name: Ensure Apache is installed 
    yum: name=httpd state=present 
    when: ansible_os_family == "RedHat" # This will run conditionally based on
                                        # the Linux version.

  - name: Start Apache Services
    service: name=httpd enabled=yes state=started

  - name: Deploy configuration File                                   # template module copies files after 
    template: src=templates/httpd.j2 dest=/etc/httpd/conf/httpd.conf  # proccessing them with Jinja2
    notify:
      - Restart Apache  # Notify the handler

  - name: Copy Site Files
    template: src=templates/index.j2 dest={{ doc_root }}/index.html
  
  handlers:
      - name: Restart Apache
        service: name=httpd state=restarted 

```

Warto zwrócić uwagę na klucz `vars`, który specyfikuje 4 wartości. Jedna z nich używana jest w samym playbooku (`doc_root`) a trzy pozostałe używane są w plikach żródłowych przetwarzanych przez moduł `template`. 

Wartościami zmiennych mogą być stringi, inty, listy, a także słowniki.

#### Role

Celem ról jest wyabstrachowanie zadań i handlerów składających się na konkretne funkcjonalności w celu zwiększenia ponownego użycia tych samych kawałków kodu.

[Ansible Galaxy](https://galaxy.ansible.com/) jest odpowiednikiem DockerHuba -- internetowym repozytorium do pobierania i dzielenia się rolami.

po wykonaniu komendy `ansible-galaxy init moja_rola` w katalogu `moja_rola` zostanie utowrzona następująca struktura:

```
├── README.md  
├── defaults
│ └── main.yml
├── files
├── handlers
│ └── main.yml
├── meta
│ └── main.yml
├── tasks
│ └── main.yml
├── templates
└── vars
└── main.yml
```

Jak widać Role mogą mieć własne szablony, pliki i zmienne. 

Może to być wykorzystane w architekturze konfiguratora: może każda paczka powinna zawierać ansiblową rolę, w której to dostawca mógłby zadbać o wszelkie kontekstualne zadania, a od administratora instancji pobrać tylko odpowiednie zmienne predefiniowane w pliku `vars/main.yml`?

#### Podsumowanie

- Inwenatrz mapuje serwery zdalne
- Konfiguracja ustawia parametry Ansible
- Moduły definiują akcję
- Playbooki koordynują wiele zadań
- Python używany do zbudowania egzekucji
- SSH do wykonania zadań

![graph/ansible schema](graph/ansible_schema.png)

źródło: https://app.pluralsight.com/library/courses/hands-on-ansible

### Struktura repozytorium

W tej sekcji omówione są wybrane pliki składające się na repozytorium.

- Katalog `ansible-eksperyment`:
    - `ansible.cfg` -- plik konfiguracyjny ansible
    - `build_site.yml` -- playbook uruchamiający kontener z dockerem i odpowiednio przetworzonym plikiem `index.html`
    - `hosts` -- Inwenatrz specyfikujący cztery maszyny postawione przez Vagranta
    - `install_docker.yml` -- playbook instalujący dockera na hoście `test1`
    - `requirements.yml` -- analogicznie do pipa tylko dla ansible-galaxy
    - `snippets.yml` -- przykładowe taski ansible
    - `Vagrantfile --` plik konfiguracyjny dla programu Vagrant specyfikujący wirtualne maszyny.
- Katalog `dockerizing`:
    - przykładowe proste playbooki do zarządzania kontenerami
- Katalog `no-docker`:
    - przykładowa struktura do odpalenia serwera Apache bez dockera i zarządzanie serwerem bazodanowym z MySQL.
    - `playbook_example.yml` -- czysty playbook nie zawierający ról z promptem do użytkownika o zmienną.
    - `site.yml` - playbook wykorzystujący role

### Wybrane źródła

- https://docs.docker.com/
- https://www.ansible.com/docker
- https://www.ansible.com/blog/six-ways-ansible-makes-docker-compose-better
- https://docs.ansible.com/ansible/docker_container_module.html
- https://linuxacademy.com/howtoguides/posts/show/topic/13750-managing-docker-containers-with-ansible
- https://opensolitude.com/2015/05/26/building-docker-images-with-ansible.html
- https://www.udemy.com/ansible-essentials-simplicity-in-automation/learn/v4/overview
- https://app.pluralsight.com/library/courses/hands-on-ansible

## Wnioski

1. Zaletą oparcia się na produkcyjnym module takim jak Ansible jest zapewnienie sobie aktualizacji do najnowszych wersji. W przypadku oparcia się tylko o API Dockera należałoby przepisywać kod w przypadku sporych zmian w API.

2. Ansible udostępnia [pythonowe API](http://docs.ansible.com/ansible/dev_guide/developing_api.html) jednak zaznacza, że głównie spełniać ma ono potrzeby CLI i w związku z tym nie ma gwarancji, że w przyszłości nie dojdzie do znaczących zmian.
    - Powyższe kieruje mnie w stronę propozycji by webowy UI konfiguratora na naciśnięcie np. przycisku *INSTALUJ* uruchamiał skrypty (naszego autorstwa), które będą odpowiedzialne za wykonywanie ansibla z odpowiednimi parametrami, zarządzanie rolami, a także wszelkimi potrzebnymi zadaniami. Te skrypty mogłyby spełniać rolę "zarządcy instancji", o którym mowa jest w pliku *Funkcje backendu konfiguratora*.

3. Dodatkowo wydaje się, że rozwiązanie *Aplikacje jako Role* będzie łatwo testowalne. NASK może utrzymywać testową instancję w chmurze i wydawać certyfikacje tym aplikacjom, które pomyślnie przeszły proces konfiguracji na instancji testowej (dodajmy, że można by w przyszłości stworzyć Playbooki do wykonywania takich testów automatycznie).

4. Otwarte pozostaje pytanie co zrobić z sekcją `provides` pliku `parameters.json`, na którą nie ma domyślnie miejsca w strukturze katalogowej ansiblowej roli.
    - Moją propozycją byłoby w tym wypadku umieszczenie jej w zmiennych (`vars/main.yml`) jako słownik (taki sam jak w paramters.json) z domyślnymi wartościami (które nie ulegną zmianie, zostaną po prostu sparsowane przez Django, przy tworzeniu obiektów).

5. Architektura Ansible przewiduje zarządzanie hostami, które są
   maszynami fizycznymi lub wirtualnymi. Takie założenie nie do końca
   pasuje do architektury platformy, w której konfigurator zarządza
   Docker Swarmem abstrahując od niższych warstw (np. pojęcia maszyny).
    - Wykorzystanie Anible nakłada na nas obciążenie w postaci wymogu
      zainstalowanego SSH i Pythona na jednym managerze w Docker
      Swarmie. W niektórych przypadkach może to ograniczać możliwości
      uruchomienia instancji. (Np. trudnym będzie użycie Dockera
      uruchomionego wewnątrz kontenera Dockera.)

### Odpowiedzi na pytania eksperymentalne

1. *Jak będzie wyglądać intergracja Ansible z webowym UI konfiguratora?*
    - *Jakie są różnice pomiędzy powyższym sposobem integracji a modelem aktualnie realizowanym (który jest nakładką na docker/docker-compose)?*
        - ...
    - *Czy i jak zmienią się aktualnie używane w konfiguratorze modele?*
        - możliwe, że należałoby dodać obiekt Role, bądź Var, do reprezentacji ansiblowej roli.
2. *Czy Ansible będzie się umiał zająć zarządzaniem użytkownikami w BD?*
    - Tak 
3. *Wykorzystując docker-compose mamy problem z tym, że modyfikacja jednego serwisu wymaga restartu wszystkich. (docker-compose down; create new compose file; docker-compose up; ) Czy Ansible jest pod tym względem lepsze?*
    - Wydaje się, że ansible będzie zmieniał automatycznie stan gdy dojdzie nowy kontener, nie mam jednak pewności co do samego restartu platformy.
4. *Jak można się komunikować z Ansible? Pisze sie skrypty w plikach czy jest jakieś inne API?*
    - Można korzystać z wiersza poleceń, pisać skrypty, API jest choć niestabilne.
5. *Czy plik parametryzacyjny (*parameters.json*) można zastąpić odpowiednim Playbookiem bądź Rolą?*
    - Da się. W mojej opini zastąpienie go Rolą jest efektywnym i eleganckim rozwiązaniem. 


### Rekomendacja

Na pewno trzeba wykonać jeszcze dwie rzeczy:

- *ad* pytanie eksperymentalne 3): zapoznać się z dokumentacją docker_service
- *ad* wniosek 5): Zapytać NASK jak są elastyczni w temacie maszyn dla Dockera.
 
Skłaniam się ku pozytywnej rekomendacji.


## Odtworzenie eksperymentu

#### Wymagania

- [Ansible](http://docs.ansible.com/ansible/intro_installation.html)
- [Vagrant](https://www.vagrantup.com/intro/index.html)

#### Proces

- Setup:
    1. Sklonować repozytorium:
    2. przejść do katalogu: `cd /sciezka/do/repo`
    3. w konsoli wpisać: `vagrant up` (zbootowanie maszyn wirtualnych)
    4. w konsoli wpisać: `ansible-galaxy install -r requirements.yml` (pobranie roli odpowiedzialnej za instalację Dockera)
    5. w konsoli wpisać: `ansible-playbook install_docker.yml` (upewnia się, że Docker jest zainstalowany na hoście)

- Eksperyment:
    1. w konsoli wpisać: `ansible-playbook build_site.yml`
    2. Zostanie się poproszonym o wpisanie swojego imienia
    3. W przypadku zapytania o hasło (*TASK [upload the site directory to the docker host]*) wpisać: `vagrant`
    4. Na końcu w ramach tasku *get the port to go to* wyświetla się komunikat pod jaki adres się udać np: `To see the page, visit 192.168.33.40:32773`
    5. Udać się pod wskazany adres. Powinno się zobaczyć tam stronę z imieniem podanym w kroku 7.
