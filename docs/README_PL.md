# PiPup – nakładka na Android TV dla Home Assistant (pl)

<a href="https://www.buymeacoffee.com/yourusername" target="_blank">
  <img src="https://cdn.buymeacoffee.com/buttons/v2/default-orange.png" alt="Buy Me A Coffee" height="40" >
</a>

PiPup to usługa nakładki na Android TV, która może wyświetlać rozbudowane wyskakujące okienka na dowolnej aplikacji.
Ten fork rozszerza oryginalny projekt o **funkcje przyjazne dla Home Assistant**, takie jak:
- dźwięki powiadomień
- stałe kafelki
- przyciski akcji
- okienko sterowania kamerą z funkcją PTZ.

![PiPup demo](docs/images/pipup-demo.gif)

## Spis treści

- [Przegląd](#przegld)
- [Oryginalne funkcje](#oryginalne-funkcje)
- [Ulepszenia w tym forku](#ulepszenia-w-tym-forku)
- [Instalacja](#instalacja)
- [Integracja z Home Assistant](#integracja-z-home-assistant)
- [Podstawowe wyskakujące okienko](#podstawowe-wystakujące-okienko)
- [Panel stałych powiadomień](#panel-stałych-powiadomień)
- [Powiadomienie z akcjami](#powiadomienie-z-akcjami)
- [Okienko sterowania kamerą (PTZ)](#okienko-sterowania-kamerą)
- [Plan rozwoju](#plan-rozwoju)
- [Wsparcie](#wsparcie)

PiPup działa jako usługa pierwszoplanowa na Android TV i rysuje okna nakładkowe za pomocą `WindowManager`.
Możesz wywoływać wyskakujące okienka z Home Assistant (lub dowolnego klienta HTTP), wywołując prosty punkt końcowy JSON udostępniany przez wbudowany serwer WWW.

Domyślny port serwera: **7979**
Domyślny punkt końcowy: `POST /notify`

## Oryginalne funkcje

Oryginalny (źródło projektu PiPup: https://github.com/gmcmicken/PiPup) oferuje już:

- Wyskakujące okienka tekstowe (tytuł + wiadomość) z konfigurowalnymi kolorami i rozmiarami.
- Wyskakujące okienka z obrazkami/bitmapami ładowane z adresu URL lub przesyłane w postaci wielu części.
- Wyskakujące okienka wideo za pomocą `VideoView`.
- Wyskakujące okienka internetowe za pomocą `WebView` (`media.web.uri`).
- Pozycjonowanie (rogi / środek) i czas trwania w sekundach.
- Proste polecenia REST Home Assistant do wyświetlania i ukrywania wyskakujących okienek.

Jeśli potrzebujesz tylko podstawowych okienek z obrazami/tekstem, nadal obowiązują oryginalne przykłady README.

---

## Ulepszenia w tym forku

Ten fork koncentruje się na **korzystaniu z Home Assistant na Android TV** i dodaje:

### 1. Dźwięki powiadomień

- Opcjonalne pole `sound` w paśmie:
- ``default``, ``alarm``, ``doorbell``, ``none``.
- Dźwięki są odtwarzane po wyświetleniu okienka.

```json
{
  "title": "Doorbell",
  "message": "Someone is at the door",
  "sound": "doorbell"
}
```
### 2. Stały panel powiadomień

![Example popup](docs/images/screen12a.png)

Okna można oznaczyć jako stałe i będą one wyświetlane jako kafelki w górnym panelu.
Kafelki są odporne na ponowne uruchomienie aplikacji (przechowywane w SharedPreferences).
Każdy trwały wpis zawiera:
- notificationId
- czas trwania (opcjonalnie z automatycznym wygaśnięciem)
- znaczniki czasu, aby przywrócić pozostały czas po ponownym uruchomieniu.

### 3. Wyskakujące okienka (AKCJE)
![Example popup](docs/images/screen8a.png)
Wyskakujące okienka mogą zawierać listę akcji z etykietami i identyfikatorami.
Przyciski można aktywować i nawigować za pomocą krzyżaka.
Po naciśnięciu przycisku PiPup wywołuje callbackUrl z JSON:
```json
{
  "notificationId": "door_alert",
  "action": "open_gate"
}
```
Idealnie pasuje to do webhooków Home Assistant / automatyzacji REST.

### 4. Wyskakujące okienko sterowania kamerą (PTZ)
Specjalny tryb wyskakującego okienka z wbudowanym WebView dla strumienia kamery.

Używa pilota telewizora do wysyłania poleceń PTZ z powrotem do Home Assistant:

góra / dół / lewo / prawo / OK.

Rozmiar i położenie wyskakującego okienka można skonfigurować w HA.
Prawidłowe czyszczenie, aby zatrzymać dźwięk z kamery po zamknięciu wyskakującego okienka.

## Instalacja
Włącz instalowanie aplikacji z nieznanych źródeł na telewizorze z systemem Android.

[Pobierz APK](https://github.com/Controli-pl/Pipup-by-Controli/releases/latest/download/pipup-controli.apk)

Zainstaluj.

Udziel uprawnień do nakładki (SYSTEM_ALERT_WINDOW) — wyświetlanie na innych aplikacjach.

![Example popup](docs/images/uprawnienia.png)

Upewnij się, że PiPup może działać jako usługa na pierwszym planie (bez agresywnych programów wyczerpujących baterię).

## Integracja z Home Assistant
Poniżej znajdują się praktyczne przykłady różnych funkcji.
# Oczywiście zmień adres IP na poprawny! (dla Android TV i HA - do wywołania zwrotnego)

### Podstawowe wyskakujące okienko
Proste wyskakujące okienko JSON za pomocą polecenia rest_command:
```text
rest_command:
  pipup_basic_popup:
    url: http://192.168.1.231:7979/notify
    method: POST
    content_type: "application/json"
    payload: >
      {
        "title": "{{ title | default('PiPup') }}",
        "message": "{{ message | default('Hello from Home Assistant') }}",
        "duration": {{ duration | default(10) }},
        "position": {{ position | default(5) }}
      }
```

Użycie:
```text
service: rest_command.pipup_basic_popup
data:
  title: "Test"
  message: "This is a basic popup"
  duration: 8
  position: 5
```
### Panel stałych powiadomień
Każdy stały element jest przechowywany z identyfikatorem powiadomienia i opcjonalnie może wygasnąć po upływie określonej liczby sekund.
```text
rest_command:
  pipup_persistent:
    url: http://192.168.1.231:7979/notify
    method: POST
    content_type: "application/json"
    payload: >
      {
        "title": "{{ title | default('Status') }}",
        "message": "{{ message | default('Something happened') }}",
        "backgroundColor": "{{ color | default('#CC000000') }}",
        "notificationId": "{{ id | default('example_status') }}",
        "persistent": true,
        "duration": {{ duration | default(0) }}
      }
  pipup_persistent_clear_one:
    url: http://192.168.1.231:7979/notify
    method: POST
    content_type: "application/json"
    payload: >
      {
        "notificationId": "{{ id | default('example_status') }}"
      }
  pipup_persistent_clear_all:
    url: http://192.168.1.231:7979/notify
    method: POST
    content_type: "application/json"
    payload: >
      {
      }
  pipup_actionable:
    url: http://192.168.1.231:7979/notify
    method: POST
    content_type: "application/json"
    payload: "{{ payload }}"
```
# Użycie:

Utwórz trwały wpis:

```text
service: rest_command.pipup_persistent
data:
  title: "Test"
  message: "This is a basic popup"
  duration: 8
  position: 5
  duration: 60
  notificationId: 'test'
```

Usuń jeden trwały wpis:

```text
service: rest_command.pipup_persistent_clear_one
data:
  notificationId: 'test'
```

Wyczyść wszystko:

```text
  action: rest_command.pipup_persistent_clear_all
  metadata: {}
  data: {}
```

### Akcje wyskakujących okienek
akcje są kodowane jako "id:Label|id2:Label2|...".

```text
  pipup_actionable:
    url: http://192.168.1.231:7979/notify
    method: POST
    content_type: "application/json"
    payload: "{{ payload }}"
```

```text
action: rest_command.pipup_actionable
data:
  payload: |
    {
      "title": "DOORBELL RINGING",
      "message": "SOMEONE AT THE DOOR",
      "duration": 30,
      "titleSize": "20",
      "messageSize" : "14",
      "backgroundColor": "#CC000000",
      "sound": "doorbell",
      "actions": [
        {"id": "open_gate", "label": "Open Gate"},
        {"id": "camera", "label": "Camera"},
        {"id": "ignore", "label": "Ignore"}
      ],
      "callbackUrl": "http://192.168.1.122:8123/api/webhook/doorbell_action", #Home Assistant IP
      "notificationId": "doorbell_main"
    }
```

Przykładowa automatyzacja do obsługi akcji w HA:

```text
automation:
  - id: pipup_actions_webhook
    alias: "PiPup – actionable popup handler"
    trigger:
      - platform: webhook
        webhook_id: doorbell_action
    action:
      - choose:
          - conditions:
              - condition: template
                value_template: "{{ trigger.json.action == 'open_gate' }}"
            sequence:
              - service: switch.turn_on
                target:
                  entity_id: switch.gate_relay

          - conditions:
              - condition: template
                value_template: "{{ trigger.json.action == 'ignore' }}"
            sequence:
              - service: logbook.log
                data:
                  name: "PiPup"
                  message: "User ignored the alert"
```

### Okienko sterowania kamerą (PTZ)
Wyświetl strumień z kamery i steruj PTZ zdalnie.

1. Polecenie REST
```text
rest_command:
  pipup_camera_control:
    url: http://192.168.1.231:7979/notify #Android IP
    method: POST
    content_type: "application/json"
    payload: >
      {
        "title": "{{ title | default('Camera View') }}",
        "duration": {{ duration | default(120) }},
        "position": {{ position | default(1) }},
        "cameraControl": true,
        "controlCallbackUrl": "http://192.168.1.122:8123/api/webhook/camera_control", #HA IP
        "notificationId": "camera_view",
        "media": {
          "web": {
            "uri": "{{ url }}",
            "width": {{ width | default(1920) | int }},
            "height": {{ height | default(1080) | int }}
          }
        }
      }
```
Pozycje okienek pop-up:
| position | Pozycja ekranu |
| -------- | --------------- |
| 0 | Prawy górny róg |
| 1 | Lewy górny róg |
| 2 | Prawy dolny róg |
| 3 | Lewy dolny róg |
| 4 | Środek |

Przykładowe rozmiary dla 1920×1080:
| Nazwa | szerokość | wysokość |
| ---------- | ----- | ------ |
| Pełny ekran | 1920 | 1080 |
| 3/4 ekranu | 1440 | 810 |
| 1/2 ekranu | 960 | 540 |
| 1/4 ekranu | 480 | 270 |

Przykład użycia (3/4 ekranu, prawy górny róg):
```text
service: rest_command.pipup_camera_control
data:
  title: "PTZ Camera"
  url: "http://192.168.1.122:1984/stream.html?src=PTZ_R_SD" #camera stream
  width: 1440
  height: 810
  position: 4
  duration: 120
```
2. Webhook PTZ w Home Assistant
```text
automation:
  - id: pipup_camera_ptz_control
    alias: "PiPup – camera PTZ control"
    trigger:
      - platform: webhook
        webhook_id: camera_control
    action:
      - choose:
          - conditions:
              - condition: template
                value_template: "{{ trigger.json.direction == 'up' }}"
            sequence:
              - service: onvif.ptz #or other action
                target:
                  entity_id: camera.ptz_camera
                data:
                  move_mode: RelativeMove
                  tilt: 0.1

          - conditions:
              - condition: template
                value_template: "{{ trigger.json.direction == 'down' }}"
            sequence:
              - service: onvif.ptz
                target:
                  entity_id: camera.ptz_camera
                data:
                  move_mode: RelativeMove
                  tilt: -0.1

          - conditions:
              - condition: template
                value_template: "{{ trigger.json.direction == 'left' }}"
            sequence:
              - service: onvif.ptz
                target:
                  entity_id: camera.ptz_camera
                data:
                  move_mode: RelativeMove
                  pan: -0.1

          - conditions:
              - condition: template
                value_template: "{{ trigger.json.direction == 'right' }}"
            sequence:
              - service: onvif.ptz
                target:
                  entity_id: camera.ptz_camera
                data:
                  move_mode: RelativeMove
                  pan: 0.1

          - conditions:
              - condition: template
                value_template: "{{ trigger.json.direction == 'ok' }}"
            sequence:
              - service: onvif.ptz
                target:
                  entity_id: camera.ptz_camera
                data:
                  move_mode: GotoHome
```
Mapowanie zdalne:
| Przycisk zdalny | Ładunek kierunkowy |
| ------------- | ----------------- |
| Góra | "góra" |
| Dół | "dół" |
| Lewo | "lewo" |
| Prawo | "prawo" |
| Środek / OK | "ok" |

### Plan rozwoju
Planowane pomysły dla tego forka:

- Widok z wielu kamer
- Siatka 2×2 / 3×3 z wieloma kamerami.
- Nawigacja krzyżakiem między kafelkami, OK = powiększenie wybranej kamery.
- Mapowanie przybliżania/oddalania (np. zwiększanie/zmniejszanie głośności) tam, gdzie obsługuje to PTZ.
- Eksportowalne plany Home Assistant dla typowych scenariuszy (dzwonek do drzwi, ruch, watchdog).
- Opcjonalny interfejs sterowania MQTT.
- Trwała animacja powiadomienia (przypomnienie)

### Opinia

Znalazłeś błąd lub masz pomysł?

- Zgłoś problem w zakładce **Issues** na GitHubie.
- Podaj model swojego urządzenia, wersję Androida i wersję aplikacji, a także kroki umożliwiające odtworzenie problemu.

Repozytorium: https://github.com/Controli-pl/Pipup-by-Controli

### Wsparcie

Czy moja praca była pomocna? Rozważ wsparcie. Dziękuję!
<a href="https://www.buymeacoffee.com/yourusername" target="_blank">
  <img src="https://cdn.buymeacoffee.com/buttons/v2/default-orange.png" alt="Buy Me A Coffee" height="40" >
</a>

