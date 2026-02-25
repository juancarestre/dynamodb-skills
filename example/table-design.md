# DynamoDB Table Design

> Generated on: 2026-02-25
> Application: Music Ranking
> Input: .test/access-patterns.md
> Status: DRAFT

## Table Configuration

| Property | Value |
|---|---|
| Table Name | `music-ranking` |
| Partition Key | `pk` (String) |
| Sort Key | `sk` (String) |
| Billing Mode | PAY_PER_REQUEST |
| TTL Attribute | N/A |

## Global Secondary Indexes

| GSI Name | Partition Key | Sort Key | Projection | Purpose |
|----------|--------------|----------|------------|---------|
| GSI1 | `GSI1PK` (S) | `GSI1SK` (S) | ALL | User lookup by username; user songs by genre; user songs by subgenre; global ranking by genre |
| GSI2 | `GSI2PK` (S) | `GSI2SK` (S) | ALL | User songs by artist; global ranking by artist; song title search |

## Entity Key Design

### User

- **Description**: A registered user who creates and ranks songs
- **PK**: `USER#<userId>`
- **SK**: `USER#<userId>`
- **GSI1PK**: `USERNAME#<username>`
- **GSI1SK**: `USERNAME#<username>`
- **Attributes**:
  - `userId` (string) - ULID
  - `username` (string) - unique display name
  - `email` (string) - unique email
  - `avatarUrl` (string, optional) - profile picture
  - `createdAt` (string) - ISO 8601
  - `entityType` (string) - `"User"`

Example item:
```json
{
  "pk": "USER#01HUSER001",
  "sk": "USER#01HUSER001",
  "GSI1PK": "USERNAME#peto",
  "GSI1SK": "USERNAME#peto",
  "userId": "01HUSER001",
  "username": "peto",
  "email": "peto@example.com",
  "avatarUrl": "https://cdn.example.com/peto.jpg",
  "createdAt": "2026-01-15T10:00:00Z",
  "entityType": "User"
}
```

### SongRating (merged Song + Rating)

- **Description**: A song with its metadata and the user's score. Song and Rating merged into one item since they are always 1:1.
- **PK**: `USER#<userId>`
- **SK**: `SONG#<invertedScore>#<songId>` where invertedScore = zero-padded 3 digits of (100 - score). This gives score-descending order with ScanIndexForward=true.
- **GSI1PK**: `USER#<userId>#GENRE#<genre>` (for genre filter) -- also: `USER#<userId>#SUBGENRE#<genre>#<subgenre>` for subgenre items are dual-written via a second approach (see Design Decisions)
- **GSI1SK**: `SONG#<invertedScore>#<songId>`
- **GSI2PK**: `USER#<userId>#ARTIST#<artist>`
- **GSI2SK**: `SONG#<invertedScore>#<songId>`
- **Attributes**:
  - `songId` (string) - ULID
  - `userId` (string) - owner
  - `title` (string) - song title
  - `artist` (string) - artist/band name
  - `album` (string, optional)
  - `genre` (string) - from fixed list
  - `subgenre` (string, optional) - from fixed list
  - `year` (number, optional) - release year
  - `duration` (number, optional) - seconds
  - `coverArtUrl` (string, optional)
  - `score` (number) - 1-100 rating
  - `songKey` (string) - normalized `lowercase(artist)#lowercase(title)` for global aggregation
  - `createdAt` (string) - ISO 8601
  - `updatedAt` (string) - ISO 8601, last score change
  - `entityType` (string) - `"SongRating"`
  - `titleLower` (string) - lowercase title for search

Example item (score = 95, invertedScore = 005):
```json
{
  "pk": "USER#01HUSER001",
  "sk": "SONG#005#01HSONG001",
  "GSI1PK": "USER#01HUSER001#GENRE#Rock",
  "GSI1SK": "SONG#005#01HSONG001",
  "GSI2PK": "USER#01HUSER001#ARTIST#Radiohead",
  "GSI2SK": "SONG#005#01HSONG001",
  "songId": "01HSONG001",
  "userId": "01HUSER001",
  "title": "Paranoid Android",
  "artist": "Radiohead",
  "album": "OK Computer",
  "genre": "Rock",
  "subgenre": "Alternative Rock",
  "year": 1997,
  "duration": 383,
  "coverArtUrl": "https://cdn.example.com/ok-computer.jpg",
  "score": 95,
  "songKey": "radiohead#paranoid android",
  "createdAt": "2026-02-01T12:00:00Z",
  "updatedAt": "2026-02-20T15:30:00Z",
  "entityType": "SongRating",
  "titleLower": "paranoid android"
}
```

### SongRating Subgenre Index Item

- **Description**: Duplicate SongRating entry for subgenre filtering via GSI1. Written alongside the main SongRating item.
- **PK**: `USER#<userId>`
- **SK**: `SONGSUB#<invertedScore>#<songId>`
- **GSI1PK**: `USER#<userId>#SUBGENRE#<genre>#<subgenre>`
- **GSI1SK**: `SONG#<invertedScore>#<songId>`
- **Attributes**: Same as SongRating

Example item:
```json
{
  "pk": "USER#01HUSER001",
  "sk": "SONGSUB#005#01HSONG001",
  "GSI1PK": "USER#01HUSER001#SUBGENRE#Rock#Alternative Rock",
  "GSI1SK": "SONG#005#01HSONG001",
  "songId": "01HSONG001",
  "title": "Paranoid Android",
  "artist": "Radiohead",
  "genre": "Rock",
  "subgenre": "Alternative Rock",
  "score": 95,
  "entityType": "SongRatingSub"
}
```

### SongRating Title Search Item

- **Description**: Index entry for searching user's songs by title prefix via GSI2.
- **PK**: `USER#<userId>`
- **SK**: `SONGTITLE#<titleLower>#<songId>`
- **GSI2PK**: `USER#<userId>#TITLE`
- **GSI2SK**: `SONG#<titleLower>#<songId>`
- **Attributes**: Minimal -- songId, title, artist, score, entityType

Example item:
```json
{
  "pk": "USER#01HUSER001",
  "sk": "SONGTITLE#paranoid android#01HSONG001",
  "GSI2PK": "USER#01HUSER001#TITLE",
  "GSI2SK": "SONG#paranoid android#01HSONG001",
  "songId": "01HSONG001",
  "title": "Paranoid Android",
  "artist": "Radiohead",
  "score": 95,
  "entityType": "SongRatingTitle"
}
```

### GlobalRating

- **Description**: Aggregated average score for a song (by normalized artist+title) across all users
- **PK**: `GLOBAL`
- **SK**: `SONG#<invertedAvgScore>#<songKey>` where invertedAvgScore = zero-padded 3 digits of (100 - floor(averageScore))
- **GSI1PK**: `GLOBAL#GENRE#<genre>`
- **GSI1SK**: `SONG#<invertedAvgScore>#<songKey>`
- **GSI2PK**: `GLOBAL#ARTIST#<artist>`
- **GSI2SK**: `SONG#<invertedAvgScore>#<songKey>`
- **Attributes**:
  - `songKey` (string) - `lowercase(artist)#lowercase(title)`
  - `title` (string) - canonical title
  - `artist` (string) - canonical artist
  - `genre` (string)
  - `subgenre` (string, optional)
  - `averageScore` (number) - average across all users
  - `ratingCount` (number) - number of users who rated
  - `updatedAt` (string) - ISO 8601
  - `entityType` (string) - `"GlobalRating"`

Example item (avgScore = 92.5, invertedAvgScore = 007):
```json
{
  "pk": "GLOBAL",
  "sk": "SONG#007#radiohead#paranoid android",
  "GSI1PK": "GLOBAL#GENRE#Rock",
  "GSI1SK": "SONG#007#radiohead#paranoid android",
  "GSI2PK": "GLOBAL#ARTIST#Radiohead",
  "GSI2SK": "SONG#007#radiohead#paranoid android",
  "songKey": "radiohead#paranoid android",
  "title": "Paranoid Android",
  "artist": "Radiohead",
  "genre": "Rock",
  "subgenre": "Alternative Rock",
  "averageScore": 92.5,
  "ratingCount": 47,
  "updatedAt": "2026-02-25T08:00:00Z",
  "entityType": "GlobalRating"
}
```

## Item Collections

| Collection PK | Entity Types | Purpose |
|---|---|---|
| `USER#<userId>` | User, SongRating, SongRatingSub, SongRatingTitle | User profile + full music collection with scores |
| `GLOBAL` | GlobalRating | Global ranking aggregated across all users |

## Access Pattern Mapping

### Reads

| AP ID | Description | Operation | Table/GSI | PK | SK Condition | SK Value | Params |
|-------|-------------|-----------|-----------|----|--------------|---------|---------| 
| AP-001 | Get user by userId | Query | Table | `USER#<userId>` | = | `USER#<userId>` | Limit=1 |
| AP-002 | Get user by username | Query | GSI1 | `USERNAME#<username>` | = | `USERNAME#<username>` | Limit=1 |
| AP-003 | User's top 100 (all songs with scores) | Query | Table | `USER#<userId>` | begins_with | `SONG#` | ScanIndexForward=true (inverted score = ascending SK = descending score) |
| AP-004 | User's top songs by genre | Query | GSI1 | `USER#<userId>#GENRE#<genre>` | begins_with | `SONG#` | ScanIndexForward=true |
| AP-005 | User's top songs by subgenre | Query | GSI1 | `USER#<userId>#SUBGENRE#<genre>#<subgenre>` | begins_with | `SONG#` | ScanIndexForward=true |
| AP-006 | User's top songs by artist | Query | GSI2 | `USER#<userId>#ARTIST#<artist>` | begins_with | `SONG#` | ScanIndexForward=true |
| AP-007 | Search user's songs by title | Query | GSI2 | `USER#<userId>#TITLE` | begins_with | `SONG#<titlePrefix>` | - |
| AP-008 | Global top 100 | Query | Table | `GLOBAL` | begins_with | `SONG#` | ScanIndexForward=true, Limit=100 |
| AP-009 | Global top by genre | Query | GSI1 | `GLOBAL#GENRE#<genre>` | begins_with | `SONG#` | ScanIndexForward=true, Limit=100 |
| AP-010 | Global top by artist | Query | GSI2 | `GLOBAL#ARTIST#<artist>` | begins_with | `SONG#` | ScanIndexForward=true, Limit=100 |

### Writes

| AP ID | Description | Operation | Items | Conditions |
|-------|-------------|-----------|-------|------------|
| AP-020 | Create user | PutItem | 1. Put User item | attribute_not_exists(pk) -- also need uniqueness on username/email via GSI1 conditional check |
| AP-021 | Add song with rating | TransactWriteItems | 1. Put SongRating (main) 2. Put SongRatingSub (subgenre index) 3. Put SongRatingTitle (title search) 4. Update GlobalRating (recalculate avg) | attribute_not_exists(pk) on items 1-3; app-level check for max 100 songs |
| AP-022 | Update rating score | TransactWriteItems | 1. Delete old SongRating + Put new SongRating (SK changes because score changed) 2. Delete old SongRatingSub + Put new SongRatingSub 3. Delete old SongRatingTitle + Put new SongRatingTitle 4. Update GlobalRating (recalculate avg) | Old item must exist |
| AP-023 | Update user profile | UpdateItem | 1. Update User item (+ update GSI1PK/GSI1SK if username changes) | attribute_exists(pk) |

### Deletes

| AP ID | Description | Operation | Items | Conditions |
|-------|-------------|-----------|-------|------------|
| AP-030 | Remove song | TransactWriteItems | 1. Delete SongRating 2. Delete SongRatingSub 3. Delete SongRatingTitle 4. Update GlobalRating (recalculate avg, decrement count) | Song must exist |
| AP-031 | Delete user account | TransactWriteItems (batched) | 1. Query all items with PK=USER#<userId> 2. BatchWrite delete all items 3. For each song, update its GlobalRating | User must exist. May require multiple batches if >25 items. |

## Design Decisions

- **Song + Rating merged**: Since each user creates their own songs and there's exactly one rating per song, merging them into a single `SongRating` item avoids an extra item per song and simplifies reads. Every query that needs songs also needs scores.

- **Inverted score in SK**: Score is encoded as `(100 - score)` zero-padded to 3 digits in the sort key. This means `ScanIndexForward=true` (the default, lexicographic ascending) yields highest scores first. Score 95 becomes `005`, score 80 becomes `020`, score 1 becomes `099`. This avoids needing `ScanIndexForward=false` and makes the "top N" pattern a simple prefix query.

- **Subgenre as a separate item (SongRatingSub)**: GSI1PK is already used for `USER#<userId>#GENRE#<genre>` on the main SongRating. We can't also set GSI1PK to `USER#<userId>#SUBGENRE#<genre>#<subgenre>` on the same item. So we write a lightweight duplicate item with SK prefix `SONGSUB#` that carries the subgenre GSI key. Trade-off: 1 extra write per song, but enables subgenre filtering via the same GSI1.

- **Title search as a separate item (SongRatingTitle)**: Similarly, GSI2PK on the main SongRating is `USER#<userId>#ARTIST#<artist>`. For title search we need GSI2PK = `USER#<userId>#TITLE`. A lightweight index item with SK prefix `SONGTITLE#` enables `begins_with` search on lowercase titles. Trade-off: 1 extra write per song.

- **Write amplification on score update (AP-022)**: Updating a score changes the SK (because the inverted score is in the SK). DynamoDB doesn't allow updating key attributes, so we must delete + re-put. This affects 3 items (main + sub + title) plus the GlobalRating update = 7 operations in a transaction. Acceptable for Warm frequency and <100 items per user.

- **GlobalRating recalculation**: On each rating write/update/delete, the GlobalRating item is updated. To calculate the new average: `newAvg = ((oldAvg * oldCount) +/- scoreDelta) / newCount`. The GlobalRating SK also changes when the average changes (inverted score), requiring delete + re-put. This is eventual consistency -- brief staleness is acceptable.

- **GLOBAL partition as hot key**: With ~1K-10K users and Warm write frequency, the `GLOBAL` partition will handle perhaps 10-50 WCU. Well within DynamoDB's per-partition limits (1000 WCU). No sharding needed at this scale.

- **Username uniqueness**: Enforced via GSI1. On user creation, the app should first query GSI1 for `USERNAME#<username>` and fail if it exists. Not a DynamoDB-level constraint but safe at this scale.

- **No pagination needed**: User collections are max 100 songs. Global top is max 100. All fit in a single query response (<1MB).

## Validation

- [x] All 16 access patterns mapped (10 reads, 4 writes, 2 deletes)
- [x] 2 GSIs defined (GSI1, GSI2)
- [x] No hot partitions identified (GLOBAL PK is safe at this scale)
- [x] No Scan operations required
- [x] Every PK+SK combination is unique per item
- [x] Sort keys enable score-descending order via inverted encoding
- [x] Composite sort keys use consistent 3-digit zero-padding
- [x] No item exceeds 400KB (song items are ~1KB)
- [x] No item collection exceeds 10GB (100 songs * ~1KB = ~100KB per user)
- [x] Transactions involve <= 25 items per call (max 7 ops for score update)
- [x] `#` separator used consistently
- [x] Entity prefixes are UPPERCASE
- [x] Generic key names (`pk`, `sk`) used
