# Deti FM Time-Shift Audio Relay (`detifm_relay`)

A containerized, timezone-aware audio streaming relay built so my child could listen to morning stories before school, bedtime tales, songs, and live broadcasts from **Deti FM (Детское радио)** in their native language, synchronized to our local US Eastern Time (`America/New_York`) schedule.

## 📋 Story & Overview

Living in the United States while wanting a child to grow up listening to Russian-language children's radio presents a 7-to-8-hour timezone problem: Moscow's morning shows and pre-school stories air in the middle of the American night, while evening bedtime tales (*"Сказки на ночь"*) play in the middle of the afternoon.

**`detifm_relay`** solves this by continuously buffering the live Moscow AAC broadcast (`64 kbps`, `44.1 kHz`) into a rolling `700 MB` ring buffer via **FFmpeg / FFserver** and serving it through **Apache HTTP Server (`httpd`)**. Using a custom dynamic `RewriteMap` script (`httpd_prg_delay.sh`), Apache calculates the real-time UTC offset between Moscow and New York, automatically accounting for Daylight Saving Time (DST) and fine-tuning both morning and evening windows so pre-school stories play at breakfast and bedtime tales play right before sleep.

## ✨ Key Features

* **Automatic Timezone & DST Synchronization:** Dynamically calculates the exact hour difference between `Europe/Moscow` and `America/New_York`, adjusting seamlessly for US Daylight Saving Time transitions.
* **Morning & Bedtime Schedule Alignment:** Applies smart hourly offsets during early morning (`00:00-09:00` EST/EDT) and evening (`17:00-20:45` EST/EDT) windows so children's programming blocks (morning stories before school, daytime music, and evening bedtime tales) match local family routines, while also supporting custom manual offset overrides.
* **Rolling Circular Buffer (`700 MB`):** Captures up to 24 hours of continuous 64 kbps AAC audio in `/tmp/detifm.ffm`, enabling reliable historical playback without growing disk usage.
* **Private Stream Protection:** Secures the relay behind Apache HTTP Basic Authentication (`.htpasswd`) so only authorized family devices can connect.
* **Containerized Architecture:** Orchestrated via Docker Compose with isolated `ffserver` and `httpd` containers for effortless deployment on any home server or VPS.

## 🛠 Tech Stack

* **Streaming & Transcoding:** FFmpeg & FFserver (ring-buffer live capture and `?date=` time-shifted AAC streaming)
* **Web Server & Reverse Proxy:** Apache HTTP Server (`httpd:2.4.54`) with `mod_rewrite` (`RewriteMap prg:`) and `mod_proxy`
* **Time-Shift Engine:** Bash / GNU `date` timezone calculation script
* **Infrastructure:** Docker & Docker Compose

## 🏗 Project Structure

```text
detifm_relay/
├── ffserver/
│   ├── Dockerfile          # Custom FFmpeg / FFserver container build
│   └── ffserver.conf       # 700 MB rolling feed (detifm.ffm) & AAC stream config
├── web/
│   ├── httpd.conf          # Apache reverse proxy, Basic Auth & RewriteMap config
│   └── etc/
│       ├── httpd_prg_delay.sh  # Dynamic Moscow-to-New York time-shift calculator
│       └── .htpasswd           # HTTP Basic Authentication credentials
└── docker-compose.yml      # Multi-container orchestration (exposes port 80)
```

## 🔄 How It Works

```text
Upstream Radio (101.ru AAC) --> FFmpeg --> FFserver Rolling Buffer (/tmp/detifm.ffm, 700MB)
                                                              ^
Home Player / Smart Speaker --> Apache httpd (:80, Basic Auth) +
                                     |
                                     +--> httpd_prg_delay.sh (RewriteMap)
                                          Calculates offset (Moscow vs. NY + DST)
                                          Appends ?date=YYYY-MM-DDTHH:MM:SS
```

1. **Ingest:** FFserver automatically launches FFmpeg to pull the live AAC stream (`64 kbps`, stereo) from Moscow and writes it continuously into `/tmp/detifm.ffm`.
2. **Request Routing:** When a home media player or smart speaker connects to the Apache web server (port `80`) and authenticates, Apache passes the request through `httpd_prg_delay.sh`.
3. **Time-Shift Rewrite:** The script computes the current offset between Moscow and New York (adjusting for DST and time-of-day rules) and outputs the target timestamp (`YYYY-MM-DDTHH:MM:SS`), allowing FFserver to stream the exact historical broadcast slice.

## 🚀 Getting Started

### Prerequisites

* [Docker](https://docker.com)
* [Docker Compose](https://docker.com)

### Installation & Deployment

1. **Clone the repository:**
   ```bash
   git clone https://github.com
   cd detifm_relay
   ```

2. **Set your own Basic Auth credentials (Recommended):**
   Replace the default credentials in `web/etc/.htpasswd` using `htpasswd`:
   ```bash
   htpasswd -cb web/etc/.htpasswd <username> <your_password>
   ```

3. **Build and start the containers:**
   ```bash
   docker-compose up -d --build
   ```

4. **Listen to the stream:**
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
  *(Note: Upon initial startup, the rolling buffer `/tmp/detifm.ffm` needs 7 to 9 hours to accumulate historical audio before time-shifted playback becomes available.)*
