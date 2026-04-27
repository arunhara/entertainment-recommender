# Entertainment Recommender - Development Plan

## Project Overview

A personalized movie and TV series recommendation application that prioritizes hidden gems based on user preferences, watched history, and online review aggregation.

## Requirements Summary

### Core Features
- **Recommendation Engine**: Suggest movies and TV series based on quality metrics and user preferences
- **Watched List Management**: Track completed movies/series with easy "mark as watched" functionality
- **Hidden Gems Focus**: Prioritize lesser-known quality content over mainstream popular titles
- **Review Aggregation**: Analyze online reviews to ensure recommendations have positive reception

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
| Frontend | React.js with TypeScript |
| Backend | Python with FastAPI |
| Database | SQLite (dev) / PostgreSQL (prod) |
| Data Source | IMDB Datasets, TMDB API, OMDb API |
| Review Analysis | Sentiment analysis on aggregated reviews |

### System Architecture

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   React Frontend │────▶│  FastAPI Backend │────▶│    Database     │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   External APIs     │
                    │ (TMDB, OMDb, IMDB)  │
                    └─────────────────────┘
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

## API Design

### Endpoints

#### Recommendations
```
GET /api/recommendations
  Query params:
    - type: 'movie' | 'series' | 'all'
    - language: string (optional, filter by language)
    - limit: integer (default: 20)
    - exclude_watched: boolean (default: true)
  Response: Array of recommended titles sorted by hidden_gem_score
```

#### Watched List
```
GET /api/watched
  Query params:
    - type: 'movie' | 'series' | 'all'
    - sort: 'date' | 'rating' | 'title'
  Response: Array of watched items

POST /api/watched
  Body: { title_id: UUID, rating?: number, notes?: string }
  Response: Created watched item

DELETE /api/watched/{id}
  Response: 204 No Content
```

#### Titles
```
GET /api/titles/{id}
  Response: Full title details with reviews

GET /api/titles/search
  Query params:
    - q: string (search query)
    - type: 'movie' | 'series' | 'all'
  Response: Array of matching titles
```

---

## Development Phases

### Phase 1: Foundation (Week 1-2)
- [ ] Set up project structure (monorepo with frontend/backend)
- [ ] Initialize FastAPI backend with basic routing
- [ ] Set up SQLite database with SQLAlchemy ORM
- [ ] Create database models and migrations
- [ ] Initialize React frontend with TypeScript

### Phase 2: Data Integration (Week 3-4)
- [ ] Download and parse IMDB datasets (title.basics, title.ratings)
- [ ] Filter data by criteria (rating >= 7.0, votes >= 3000, target languages)
- [ ] Set up TMDB API integration for additional metadata
- [ ] Implement data import pipeline
- [ ] Create scheduled job for data updates

### Phase 3: Core Features (Week 5-6)
- [ ] Implement recommendations API with hidden gem algorithm
- [ ] Build watched list CRUD operations
- [ ] Create "Mark as Watched" functionality
- [ ] Develop frontend components (title cards, lists, filters)
- [ ] Implement search functionality

### Phase 4: Review Aggregation (Week 7-8)
- [ ] Integrate OMDb API for review data
- [ ] Implement sentiment analysis on reviews
- [ ] Update hidden gem scoring with sentiment data
- [ ] Add review display in frontend
- [ ] Create review quality indicators

### Phase 5: Polish & Deployment (Week 9-10)
- [ ] Responsive design improvements
- [ ] Performance optimization (caching, pagination)
- [ ] User preferences and settings
- [ ] Deployment configuration (Docker, CI/CD)
- [ ] Documentation and testing

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
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── models/
│   │   ├── routers/
│   │   ├── services/
│   │   └── utils/
│   ├── requirements.txt
│   └── alembic/
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── hooks/
│   │   └── services/
│   ├── package.json
│   └── tsconfig.json
├── data/
│   └── imdb/
├── scripts/
│   └── import_data.py
├── PLAN.md
└── README.md
```

---

## Success Metrics

1. **Recommendation Quality**: Users discover content they wouldn't have found otherwise
2. **Hidden Gem Hit Rate**: >50% of recommendations are titles with <100k IMDB votes
3. **Watched List Usage**: Easy one-click "mark as watched" with <2 second response time
4. **Review Accuracy**: Sentiment analysis correlates with user satisfaction

---

## Next Steps

1. Initialize the project structure
2. Set up the backend with FastAPI
3. Create database models
4. Begin IMDB data import pipeline
