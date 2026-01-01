# Pipup-by-Controli
Pipup modifed by Controli.pl

# PiPup – Android TV overlay + Home Assistant camera control

PiPup shows overlay popups on Android TV and integrates tightly with Home Assistant.  
This fork adds a **camera control popup**: live camera view + PTZ control from the TV remote.

![Camera control popup](docs/images/camera-popup.png)

## Features

- Popup notifications with text, images, video and web content.
- **Camera Control popup** – live camera view with PTZ from the TV remote.
- Persistent notification panel – always-visible tiles at the top of the screen.
- Home Assistant integration (REST, webhooks, ADB conditions).
- Configurable popup size and position (fullscreen, 3/4, 1/2, corners).
- (Planned) **Multi‑camera view** – show several cameras at once.

---

## Camera Control overview

The Camera Control popup lets you:

- Open any camera stream that can be shown in WebView (e.g. go2rtc, MotionEye, Blue Iris web player).
- Control PTZ using the TV remote (up/down/left/right/OK).
- Choose the popup size (e.g. 3/4 of the screen) and screen corner.
- Auto‑close the popup after a configurable timeout.

Typical use cases:

- Doorbell events – auto show the door camera with PTZ.
- Motion detection – quickly peek at the backyard and move the camera.
- Manual on‑demand camera view from Home Assistant.

### Remote mapping

| Remote button       | Action sent to HA            |
|---------------------|-----------------------------|
| ⬆️ D‑pad Up         | PTZ: move camera up         |
| ⬇️ D‑pad Down       | PTZ: move camera down       |
| ⬅️ D‑pad Left       | PTZ: move camera left       |
| ➡️ D‑pad Right      | PTZ: move camera right      |
| ⭕ D‑pad Center / OK | PTZ: home / preset          |
| 🔙 Back             | Close the camera popup      |

---

## Installation

1. Enable **“Install unknown apps”** on your Android TV.
2. Install the PiPup APK (e.g. via `adb install`).
3. Grant overlay permission (`SYSTEM_ALERT_WINDOW`) if required by your device.
4. Configure the Home Assistant integration as shown below.

---

## Home Assistant integration

### 1. REST command – open camera popup

```yaml
rest_command:
  pipup_camera_control:
    url: http://192.168.1.231:7979/notify
    method: POST
    content_type: "application/json"
    payload: >
      {
        "title": "{{ title | default('Camera View') }}",
        "duration": {{ duration | default(120) }},
        "position": {{ position | default(1) }},
        "cameraControl": true,
        "controlCallbackUrl": "http://192.168.1.122:8123/api/webhook/camera_control",
        "notificationId": "camera_view",
        "media": {
          "web": {
            "uri": "{{ url }}",
            "width": {{ width | default(1920) | int }},
            "height": {{ height | default(1080) | int }}
          }
        }
      }

Parameters:

title – title shown above the camera.

url – camera page/stream URL.

duration – display time in seconds (0 = no auto‑close).

width / height – popup size in pixels.

position – popup position (1–5, see table below).

Popup positions
position	Screen position
1	Top‑right
2	Top‑left
3	Bottom‑right
4	Bottom‑left
5	Center
Example sizes for 1920×1080 TV
Name	width	height
Fullscreen	1920	1080
3/4 screen	1440	810
1/2 screen	960	540
1/4 screen	480	270
2. Webhook – PTZ control in Home Assistant
text
automation:
  - id: pipup_camera_ptz_control
    alias: "PiPup – Camera PTZ control"
    trigger:
      - platform: webhook
        webhook_id: camera_control
    action:
      - choose:
          # Up
          - conditions:
              - condition: template
                value_template: "{{ trigger.json.direction == 'up' }}"
            sequence:
              - service: onvif.ptz
                target:
                  entity_id: camera.ptz_camera
                data:
                  move_mode: RelativeMove
                  tilt: 0.1

          # Down
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

          # Left
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

          # Right
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

          # OK – go to home preset
          - conditions:
              - condition: template
                value_template: "{{ trigger.json.direction == 'ok' }}"
            sequence:
              - service: onvif.ptz
                target:
                  entity_id: camera.ptz_camera
                data:
                  move_mode: GotoHome
Adjust the camera.ptz_camera entity and PTZ parameters to match your setup.

3. Example automations
Doorbell – fullscreen popup
text
automation:
  - alias: "Doorbell – show camera"
    trigger:
      - platform: state
        entity_id: binary_sensor.doorbell
        to: "on"
    action:
      - service: rest_command.pipup_camera_control
        data:
          title: "🚪 Someone at the door"
          url: "http://192.168.1.122:1984/stream.html?src=doorbell"
          width: 1920
          height: 1080
          position: 5
          duration: 60
3/4 popup in the top‑right corner
text
service: rest_command.pipup_camera_control
data:
  title: "PTZ Camera"
  url: "http://192.168.1.122:1984/stream.html?src=PTZ_R_SD"
  width: 1440
  height: 810
  position: 1        # top‑right
  duration: 120
ADB conditions – detecting active popup
If you use the Android TV integration with ADB, you can prevent other automations (like changing inputs) while PiPup is on top.

Example command:

text
service: androidtv.adb_command
data:
  command: adb shell dumpsys window windows | grep -i pipup && echo 1 || echo 0
target:
  entity_id: media_player.android_tv_192_168_1_231
Template condition – true when PiPup is not in the foreground:

text
condition:
  - condition: template
    value_template: >-
      {{ not ('mCurrentFocus=Window{' in state_attr('media_player.android_tv_192_168_1_231', 'adb_response')
         and 'pipup' in state_attr('media_player.android_tv_192_168_1_231', 'adb_response')) }}
Code changes (summary)
PopupProps.kt

Added:

cameraControl: Boolean

controlCallbackUrl: String?

PiPupService.kt

createCameraControlPopup(popup: PopupProps) – creates camera overlay with size from media.width/height, focuses the view and handles remote keys.

sendControlCallback(popup, direction) – sends JSON {direction, notificationId} to the Home Assistant webhook.

removePopup() – additionally searches for R.id.camera_webview inside the overlay, stops and destroys the WebView to avoid audio continuing in the background.

PopupView.kt

PopupView.Web:

in destroy() now calls:

webView.stopLoading()

webView.loadUrl("about:blank")

webView.destroy()

res/layout/popup_camera_control.xml

New layout containing:

Title TextView

WebView for the camera stream.

Roadmap / ideas
Multi‑camera view:

grid layout (2×2 / 3×3) with multiple WebViews,

focus switching between tiles with the D‑pad,

pressing OK opens the selected camera in a bigger popup.

Zoom in/out mapped to Volume Up/Down (where supported by PTZ).

Exportable Home Assistant blueprints for common scenarios (doorbell, motion, watchdog).

Screenshots
text


Made for Home Assistant & Android TV.
