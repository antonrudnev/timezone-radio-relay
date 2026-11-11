# Timezone Radio Relay (`timezone-radio-relay`)

A containerized, timezone-aware audio streaming relay that captures live internet radio broadcasts from one timezone and re-streams them with dynamic **time-shifting** aligned to a listener's local schedule.

## 📋 Story & Overview

Broadcasting live radio across distant time zones creates an obvious scheduling mismatch: morning shows air in the middle of the night, while evening programs play in the afternoon.

I originally built this project for my family in the United States so my child could listen to morning stories before school, bedtime tales, songs, and live broadcasts from **Deti FM (Детское радио)** in their native language, synchronized to our local US Eastern Time (`America/New_York`) schedule. While pre-configured for Moscow-to-New York streaming, the architecture is completely adaptable to any upstream audio stream and any pair of time zones.

**`timezone-radio-relay`** continuously buffers the upstream AAC broadcast (`64 kbps`, `44.1 kHz`) into a rolling `700 MB` ring buffer via **FFmpeg / FFserver** and serves it through **Apache HTTP Server (`httpd`)**. Using a custom dynamic `RewriteMap` script (`httpd_prg_delay.sh`), Apache calculates the real-time UTC offset between the source and destination time zones, automatically accounting for Daylight Saving Time (DST) and fine-tuning morning and evening windows so time-sensitive shows always play at the right moment of the day.

## ✨ Key Features

* **Automatic Timezone & DST Synchronization:** Dynamically calculates the exact hour difference between the source timezone (`Europe/Moscow`) and local timezone (`America/New_York`), adjusting seamlessly for Daylight Saving Time transitions.
* **Smart Morning & Evening Schedule Alignment:** Applies fine-tuned hourly offsets during early morning (`00:00-09:00`) and evening (`17:00-20:45`) windows so daily programming blocks (morning shows before school, daytime music, and evening bedtime stories) match local listener routines, while also supporting custom manual offset overrides.
* **Rolling Circular Buffer (`700 MB`):** Captures up to 24 hours of continuous 64 kbps AAC audio in `/tmp/detifm.ffm`, enabling reliable historical playback without growing disk usage.
* **Private Stream Protection:** Secures the relay behind Apache HTTP Basic Authentication (`.htpasswd`) so only authorized devices can connect.
* **Containerized Architecture:** Orchestrated via Docker Compose with isolated `ffserver` and `httpd` containers for effortless deployment on any home server or VPS.

## 🛠 Tech Stack

* **Streaming & Transcoding:** FFmpeg & FFserver (ring-buffer live capture and `?date=` time-shifted AAC streaming)
* **Web Server & Reverse Proxy:** Apache HTTP Server (`httpd:2.4.54`) with `mod_rewrite` (`RewriteMap prg:`) and `mod_proxy`
* **Time-Shift Engine:** Bash / GNU `date` timezone calculation script
* **Infrastructure:** Docker & Docker Compose

## 🏗 Project Structure

```text
timezone-radio-relay/
├── ffserver/
│   ├── Dockerfile          # Custom FFmpeg / FFserver container build
│   └── ffserver.conf       # 700 MB rolling feed & AAC stream configuration
├── web/
│   ├── httpd.conf          # Apache reverse proxy, Basic Auth & RewriteMap config
│   └── etc/
│       ├── httpd_prg_delay.sh  # Dynamic source-to-local time-shift calculator
│       └── .htpasswd           # HTTP Basic Authentication credentials
└── docker-compose.yml      # Multi-container orchestration (exposes port 80)
```

## 🔄 How It Works

```text
Upstream Radio Stream (AAC) --> FFmpeg --> FFserver Rolling Buffer (/tmp/detifm.ffm, 700MB)
                                                            ^
Media Player / Smart Speaker --> Apache httpd (:80, Auth) --+
                                      |
                                      +--> httpd_prg_delay.sh (RewriteMap)
                                           Calculates offset (Source TZ vs. Local TZ + DST)
                                           Appends ?date=YYYY-MM-DDTHH:MM:SS
```

1. **Ingest:** FFserver automatically launches FFmpeg to pull the live AAC stream (`64 kbps`, stereo) from the upstream URL and writes it continuously into `/tmp/detifm.ffm`.
2. **Request Routing:** When a media player or smart speaker connects to the Apache web server (port `80`) and authenticates, Apache passes the request through `httpd_prg_delay.sh`.
3. **Time-Shift Rewrite:** The script computes the current offset between the source and target time zones (adjusting for DST and time-of-day rules) and outputs the target timestamp (`YYYY-MM-DDTHH:MM:SS`), allowing FFserver to stream the exact historical broadcast slice.

## 🚀 Getting Started

### Prerequisites

* [Docker](https://docker.com)
* [Docker Compose](https://docker.com)

### Installation & Deployment

1. **Clone the repository:**
   ```bash
   git clone https://github.com
   cd timezone-radio-relay
   ```

2. **Customize Stream & Timezones (Optional):**
   * Update the upstream radio URL in `ffserver/ffserver.conf` (`Launch ./ffmpeg -i <stream_url>`).
   * Adjust the source and target time zones in `web/etc/httpd_prg_delay.sh` (`Europe/Moscow` and `America/New_York` by default).

3. **Set your own Basic Auth credentials (Recommended):**
   Replace the default credentials in `web/etc/.htpasswd` using `htpasswd`:
   ```bash
   htpasswd -cb web/etc/.htpasswd <username> <your_password>
   ```

4. **Build and start the containers:**
   ```bash
   docker-compose up -d --build
   ```

5. **Listen to the stream:**
   * Open `http://<server-ip>/detifm.aac` in any media player (VLC, mpv, browser, or smart speaker audio bridge) and enter your Basic Auth credentials.

## 🔧 Operations & Maintenance

* **View real-time logs (FFserver & Apache):**
  ```bash
  docker-compose logs -f
  ```
* **Restart services after configuration changes:**
  ```bash
  docker-compose restart
  ```
* **Stop the relay:**
  ```bash
  docker-compose down
  ```
  *(Note: Upon initial startup, the rolling buffer `/tmp/detifm.ffm` needs several hours to accumulate historical audio before time-shifted playback becomes available.)*
