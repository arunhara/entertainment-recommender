# Entertainment Recommender - Development Plan

## Project Overview

A lightweight, notification-based entertainment recommendation service that runs in the background. It researches and consolidates links to quality movies and TV series, sends personalized recommendations via Slack/Email/WhatsApp, and maintains a watched list through simple message commands.

## Requirements Summary

### Core Features
- **Background Recommendation Engine**: Researches and consolidates links to quality content
- **Notification Delivery**: Sends top recommendations via Slack, Email, or WhatsApp
- **Message-Driven Watched List**: Update watched status by sending a message (e.g., "watched: The Bear")
- **Hidden Gems Focus**: Prioritize lesser-known quality content over mainstream titles
- **Link Consolidation**: Aggregate streaming links, reviews, and trailers in recommendations

### Filtering Criteria
- **Minimum IMDB Rating**: 7.0 and above
- **Minimum Votes**: 3,000+ votes (ensures reliable ratings)
- **Languages** (in priority order):
  1. English
  2. Malayalam
  3. Tamil
  4. Hindi
  5. Kannada
  6. Telugu

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|------------|
| Backend | Python (background service) |
| Database | SQLite (simple, file-based) |
| Notifications | Telegram Bot API (python-telegram-bot) |
| Data Sources | IMDB Datasets, TMDB API, OMDb API, JustWatch |
| Scheduler | APScheduler or cron |

### System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Background Service                            │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Scheduler   │  │  Research    │  │  Notifier    │          │
│  │  (cron)      │──│  Engine      │──│  Service     │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│         │                │                   │                  │
│         ▼                ▼                   ▼                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   SQLite     │  │ External APIs│  │   Telegram   │          │
│  │   Database   │  │ (IMDB/TMDB)  │  │     Bot      │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ Two-way
                              ▼
                    ┌─────────────────┐
                    │   You on        │
                    │   Telegram      │
                    │ "watched: X"    │
                    └─────────────────┘
```

---

## Database Schema

### Tables

#### `titles`
| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| imdb_id | VARCHAR(15) | IMDB identifier (e.g., tt1234567) |
| title | VARCHAR(500) | Original title |
| type | ENUM | 'movie' or 'series' |
| year | INTEGER | Release year |
| imdb_rating | DECIMAL(3,1) | IMDB rating (0.0-10.0) |
| imdb_votes | INTEGER | Number of IMDB votes |
| language | VARCHAR(50) | Primary language |
| genres | JSON | Array of genres |
| runtime_minutes | INTEGER | Runtime in minutes |
| poster_url | TEXT | Poster image URL |
| overview | TEXT | Plot summary |
| hidden_gem_score | DECIMAL(5,2) | Calculated hidden gem score |
| review_sentiment | DECIMAL(3,2) | Aggregated review sentiment (-1 to 1) |
| created_at | TIMESTAMP | Record creation time |
| updated_at | TIMESTAMP | Last update time |

#### `watched_list`
| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| user_id | UUID | User identifier |
| title_id | UUID | Foreign key to titles |
| watched_date | DATE | Date marked as watched |
| rating | INTEGER | User's personal rating (1-10) |
| notes | TEXT | Optional user notes |
| created_at | TIMESTAMP | Record creation time |

#### `reviews`
| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| title_id | UUID | Foreign key to titles |
| source | VARCHAR(100) | Review source (e.g., 'rotten_tomatoes') |
| review_text | TEXT | Review content |
| sentiment_score | DECIMAL(3,2) | Sentiment analysis score |
| fetched_at | TIMESTAMP | When review was fetched |

---

## Hidden Gem Algorithm

The algorithm scores content to surface quality hidden gems:

```python
def calculate_hidden_gem_score(title):
    """
    Higher scores = more likely to be a hidden gem
    """
    score = 0.0
    
    # Rating quality (7.0-10.0 range, normalized)
    rating_factor = (title.imdb_rating - 7.0) / 3.0 * 25  # Max 25 points
    
    # Popularity sweet spot (3k-200k votes is ideal for hidden gems)
    if 3000 <= title.imdb_votes <= 50000:
        popularity_factor = 30  # Ideal hidden gem range
    elif 50000 < title.imdb_votes <= 200000:
        popularity_factor = 20  # Still relatively unknown
    elif 200000 < title.imdb_votes <= 500000:
        popularity_factor = 10  # Getting mainstream
    else:
        popularity_factor = 0  # Very popular, not a hidden gem
    
    # Age factor (older quality content often overlooked)
    current_year = 2026
    age = current_year - title.year
    if age > 20:
        age_factor = 15  # Classic hidden gem
    elif age > 10:
        age_factor = 10
    elif age > 5:
        age_factor = 5
    else:
        age_factor = 0
    
    # Review sentiment boost
    sentiment_factor = (title.review_sentiment + 1) / 2 * 20  # Max 20 points
    
    # Language priority (slight boost for non-English quality content)
    language_priority = {
        'English': 0, 'Malayalam': 5, 'Tamil': 4,
        'Hindi': 3, 'Kannada': 2, 'Telugu': 1
    }
    language_factor = language_priority.get(title.language, 0)
    
    score = rating_factor + popularity_factor + age_factor + sentiment_factor + language_factor
    
    return min(score, 100)  # Cap at 100
```

---

## Telegram Commands

### Watched List Updates
```
watched: The Bear                    → Mark "The Bear" as watched
watched: Inception (2010)            → Mark specific title as watched
finished: Breaking Bad               → Same as "watched:"
rating: The Bear 9/10                → Add personal rating
unwatched: The Bear                  → Remove from watched list
list watched                         → Get your watched list
```

### Recommendation Requests
```
recommend                            → Get today's top recommendations
recommend movies                     → Movies only
recommend series                     → TV series only
recommend malayalam                  → Filter by language
hidden gems                          → Get extra obscure recommendations
```

### Notification Format
Each recommendation notification includes:
- Title, Year, IMDB Rating
- Why it's recommended (hidden gem score reason)
- Consolidated links (streaming platforms, IMDB, trailers)
- One-line review summary
- Reply "watched: [title]" to mark as complete

---

## Development Phases

### Phase 1: Foundation (Week 1)
- [ ] Set up Python project structure
- [ ] Set up SQLite database with SQLAlchemy ORM
- [ ] Create database models (titles, watched_list, recommendations_sent)
- [ ] Basic configuration management (API keys, notification preferences)

### Phase 2: Data Integration (Week 2)
- [ ] Download and parse IMDB datasets (title.basics, title.ratings)
- [ ] Filter data by criteria (rating >= 7.0, votes >= 3000, target languages)
- [ ] Set up TMDB API integration for posters and streaming links
- [ ] Implement data import/update pipeline
- [ ] Create scheduled job for weekly data refresh

### Phase 3: Telegram Bot (Week 3)
- [ ] Create Telegram bot via BotFather
- [ ] Set up python-telegram-bot library
- [ ] Implement command handlers (/recommend, /watched, /list)
- [ ] Message parser for watched list commands
- [ ] Rich notification formatting with inline buttons and links

### Phase 4: Research & Link Consolidation (Week 4)
- [ ] JustWatch integration for streaming availability
- [ ] Review aggregation from multiple sources
- [ ] Trailer link fetching (YouTube API)
- [ ] Hidden gem scoring algorithm
- [ ] Recommendation deduplication (don't re-recommend)

### Phase 5: Background Service & Deployment (Week 5)
- [ ] APScheduler or cron-based recommendation scheduling
- [ ] Daily/weekly recommendation digest options
- [ ] Docker containerization
- [ ] Deployment to cloud (Railway/Fly.io/VPS)
- [ ] Logging and monitoring

---

## Data Sources

### Primary: IMDB Datasets
- **URL**: https://datasets.imdbws.com/
- **Files needed**:
  - `title.basics.tsv.gz` - Title information
  - `title.ratings.tsv.gz` - Ratings and vote counts
- **Update frequency**: Daily

### Secondary: TMDB API
- **Purpose**: Poster images, additional metadata, streaming availability
- **Rate limit**: 40 requests/10 seconds
- **API Key**: Required (free tier available)

### Tertiary: OMDb API
- **Purpose**: Rotten Tomatoes scores, Metacritic, detailed reviews
- **Rate limit**: 1,000 requests/day (free tier)
- **API Key**: Required

---

## File Structure

```
entertainment-recommender/
├── src/
│   ├── __init__.py
│   ├── main.py                 # Entry point, scheduler
│   ├── config.py               # Configuration and API keys
│   ├── models.py               # SQLAlchemy models
│   ├── database.py             # Database connection
│   ├── services/
│   │   ├── __init__.py
│   │   ├── imdb_service.py     # IMDB data fetching/parsing
│   │   ├── tmdb_service.py     # TMDB API integration
│   │   ├── research_service.py # Link consolidation, reviews
│   │   └── recommender.py      # Hidden gem algorithm
│   └── telegram/
│       ├── __init__.py
│       ├── bot.py              # Telegram bot setup and handlers
│       ├── commands.py         # /recommend, /watched, /list handlers
│       └── formatters.py       # Rich message formatting
├── data/
│   └── imdb/                   # Downloaded IMDB datasets
├── scripts/
│   ├── import_data.py          # Initial data import
│   └── run_recommendations.py  # Manual recommendation trigger
├── requirements.txt
├── Dockerfile
├── .env.example
├── PLAN.md
└── README.md
```

---

## Success Metrics

1. **Recommendation Quality**: Discover content you wouldn't have found otherwise
2. **Hidden Gem Hit Rate**: >50% of recommendations are titles with <100k IMDB votes
3. **Watched List Ease**: Simple message reply to mark as watched
4. **Link Quality**: All recommendations include working streaming/IMDB/trailer links
5. **No Repeats**: Never recommend something already watched or recently sent

---

## Telegram Bot Features

- **/recommend** - Get top recommendations now
- **/recommend movies** - Movies only
- **/recommend series** - TV series only  
- **/watched [title]** - Mark as watched
- **/list** - See your watched list
- **/settings** - Configure notification frequency

Configurable options:
- **Frequency**: Daily digest, weekly digest, or on-demand only
- **Count**: Number of recommendations per notification (default: 3-5)
- **Quiet Hours**: Don't notify between certain hours

---

## Next Steps

1. Initialize Python project structure
2. Set up SQLite database and models
3. Build IMDB data import pipeline
4. Create Telegram bot via @BotFather and connect
