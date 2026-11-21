# ThingSpeak — Block Diagram

High-level architecture of the ThingSpeak IoT application (Rails 4 / Ruby).

```mermaid
flowchart TB
    subgraph Clients["Clients"]
        SENSOR["IoT Sensor / Device<br/>(HTTP client)"]
        BROWSER["Web Browser<br/>(Dashboard / Charts)"]
        READER["Data Consumer App<br/>(JSON / XML / CSV)"]
    end

    subgraph App["ThingSpeak Rails Application"]
        ROUTER["Rails Router<br/>config/routes.rb"]

        subgraph Controllers["Controllers"]
            CH_CTRL["ChannelsController<br/>(post_data, show, realtime)"]
            FEED_CTRL["FeedController<br/>(public/private feeds)"]
            USER_CTRL["Users / Sessions / Registrations<br/>(Devise auth)"]
            API_CTRL["ApiKeys / Status / CORS"]
        end

        subgraph Models["Models (ActiveRecord)"]
            CHANNEL["Channel"]
            FEED["Feed / DailyFeed"]
            USER["User"]
            APIKEY["ApiKey"]
            COMMENT["Comment / Tag / Watching"]
        end

        SERVICES["Services<br/>Mailer · Realtime IO · Twitter"]
    end

    DB[("Database<br/>MySQL")]
    STORE[("Feed Storage<br/>per-channel data tables")]

    SENSOR -- "GET/POST /update?key=API_KEY" --> ROUTER
    BROWSER -- "HTTPS pages / charts" --> ROUTER
    READER -- "/channels/:id/feed.json|xml|csv" --> ROUTER

    ROUTER --> CH_CTRL & FEED_CTRL & USER_CTRL & API_CTRL

    CH_CTRL --> CHANNEL
    FEED_CTRL --> FEED
    USER_CTRL --> USER
    API_CTRL --> APIKEY

    CHANNEL --> FEED
    CHANNEL --> APIKEY
    USER --> CHANNEL
    CHANNEL --> COMMENT

    CHANNEL & FEED & USER & APIKEY --> DB
    FEED --> STORE
    CH_CTRL --> SERVICES

    SERVICES -- "realtime push / email" --> BROWSER
```

## Key Blocks

| Block | Responsibility | Location |
|---|---|---|
| Router | Maps `/update`, `/channels`, `/feeds` URLs | `config/routes.rb` |
| ChannelsController | Data ingest (`post_data`), channel CRUD, realtime | `app/controllers/channels_controller.rb` |
| FeedController | Serves stored data as JSON/XML/CSV | `app/controllers/feed_controller.rb` |
| Channel model | 8 data fields + location + elevation per entry | `app/models/channel.rb` |
| Feed / DailyFeed | Entry storage & retrieval, timescaling, averaging | `app/models/feed.rb` |
| ApiKey | Read/write key management per channel | `app/models/api_key.rb` |
| User + Devise | Accounts, sessions, registrations | `app/models/user.rb` |
