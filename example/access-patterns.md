# DynamoDB Access Patterns

> Generated on: 2026-02-25
> Application: Music Ranking
> Status: DRAFT

## Domain Overview

Multi-user application where each user builds a personal top 100 of their favorite music. Users add songs with metadata (title, artist, album, genre, subgenre, year, duration, cover art) and rate them on a 1-100 scale. Rankings can be viewed globally, filtered by genre, subgenre, or artist. A global ranking aggregates average scores across all users.

## Entities

### User

- **Description**: A registered user who creates and ranks songs
- **Attributes**:
  - `userId` (string) - unique identifier (ULID)
  - `username` (string) - display name, unique
  - `email` (string) - unique email address
  - `avatarUrl` (string, optional) - profile picture URL
  - `createdAt` (datetime) - account creation timestamp
- **Relationships**: Owns many Songs and Ratings
- **Cardinality**: ~1K-10K users
- **Lifecycle**: Created on signup. Updated on profile changes. Deleted on account removal.

### Song

- **Description**: A music track added by a user to their collection
- **Attributes**:
  - `songId` (string) - unique identifier (ULID)
  - `userId` (string) - owner who added this song
  - `title` (string) - song title
  - `artist` (string) - artist/band name
  - `album` (string, optional) - album name
  - `genre` (string) - genre from a fixed list (Rock, Pop, Jazz, Electronic, Hip-Hop, Classical, etc.)
  - `subgenre` (string, optional) - subgenre from a fixed list per genre
  - `year` (number, optional) - release year
  - `duration` (number, optional) - duration in seconds
  - `coverArtUrl` (string, optional) - album cover image URL
  - `createdAt` (datetime) - when the user added this song
- **Relationships**: Belongs to a User. Has one Rating (from its owner).
- **Cardinality**: ~100 songs per user, ~1M total
- **Lifecycle**: Created when user adds a song. Deleted when user removes it from their collection. Each user creates their own Song entry even if the same real-world song exists for another user.

### Rating

- **Description**: The score (1-100) a user assigns to one of their songs
- **Attributes**:
  - `userId` (string) - the user who rated
  - `songId` (string) - the song being rated
  - `score` (number) - integer 1-100
  - `updatedAt` (datetime) - last time the score was changed
- **Relationships**: Belongs to a User and a Song. One Rating per User-Song pair.
- **Cardinality**: 1:1 with Song (~100 per user, ~1M total)
- **Lifecycle**: Created when user rates a song. Updated when user changes the score. Deleted when the song is removed.

### GlobalRating

- **Description**: Aggregated average score for a song title+artist combination across all users
- **Attributes**:
  - `songKey` (string) - normalized key: lowercase(artist + "#" + title)
  - `title` (string) - canonical song title
  - `artist` (string) - canonical artist name
  - `genre` (string) - genre
  - `subgenre` (string, optional) - subgenre
  - `averageScore` (number) - average of all user ratings for this song
  - `ratingCount` (number) - how many users have rated this song
  - `updatedAt` (datetime) - last recalculation
- **Relationships**: Aggregates Ratings across users for the same song title+artist
- **Cardinality**: ~100K unique songs globally
- **Lifecycle**: Created or updated when any user rates a matching song. Recalculated on rating changes.

## Access Patterns

### Reads

| ID | Entity | Description | Input Parameters | Result | Sort | Paginated | Frequency | Consistency | Multi-Entity |
|----|--------|-------------|------------------|--------|------|-----------|-----------|-------------|--------------|
| AP-001 | User | Get user by userId | userId | Single | - | No | Hot | Eventual | No |
| AP-002 | User | Get user by username | username | Single | - | No | Warm | Eventual | No |
| AP-003 | Song, Rating | Get user's top 100 (all songs with scores) | userId | Collection | score DESC | No | Hot | Eventual | Yes |
| AP-004 | Song, Rating | Get user's top songs filtered by genre | userId, genre | Collection | score DESC | No | Warm | Eventual | Yes |
| AP-005 | Song, Rating | Get user's top songs filtered by subgenre | userId, genre, subgenre | Collection | score DESC | No | Warm | Eventual | Yes |
| AP-006 | Song, Rating | Get user's top songs filtered by artist | userId, artist | Collection | score DESC | No | Warm | Eventual | Yes |
| AP-007 | Song | Search user's songs by title | userId, titlePrefix | Collection | - | No | Warm | Eventual | No |
| AP-008 | GlobalRating | Get global top 100 (all genres) | - | Collection | averageScore DESC | No | Hot | Eventual | No |
| AP-009 | GlobalRating | Get global top songs by genre | genre | Collection | averageScore DESC | No | Warm | Eventual | No |
| AP-010 | GlobalRating | Get global top songs by artist | artist | Collection | averageScore DESC | No | Warm | Eventual | No |

### Writes

| ID | Entity | Description | Input Parameters | Items Written | Transaction | Conditions | Frequency |
|----|--------|-------------|------------------|---------------|-------------|------------|-----------|
| AP-020 | User | Create user | username, email | 1 (User) | No | username and email must be unique | Warm |
| AP-021 | Song, Rating | Add song with rating | userId, title, artist, album, genre, subgenre, year, duration, coverArtUrl, score | Song + Rating + GlobalRating update | Yes | User must exist, max 100 songs per user | Warm |
| AP-022 | Rating | Update rating score | userId, songId, newScore | Rating + GlobalRating update | Yes | Rating must exist | Warm |
| AP-023 | User | Update user profile | userId, username, email, avatarUrl | 1 (User) | No | User must exist | Cold |

### Deletes

| ID | Entity | Description | Input Parameters | Items Deleted | Transaction | Conditions | Frequency |
|----|--------|-------------|------------------|---------------|-------------|------------|-----------|
| AP-030 | Song, Rating | Remove song from collection | userId, songId | Song + Rating + GlobalRating update | Yes | Song must exist | Cold |
| AP-031 | User, Song, Rating | Delete user account | userId | User + all Songs + all Ratings + GlobalRating updates | Yes (batch) | User must exist | Cold |

## Item Collection Candidates

These access patterns benefit from grouping entities into the same item collection (same partition key) so they can be fetched in a single DynamoDB Query:

| Group | Entities | Access Patterns | Rationale |
|-------|----------|-----------------|-----------|
| User's Music Collection | Song, Rating | AP-003, AP-004, AP-005, AP-006, AP-007 | The top 100 view always needs both song metadata and score together. Song and Rating are always accessed in the context of a user. |
| Global Rankings | GlobalRating | AP-008, AP-009, AP-010 | Global rankings need to be sorted by average score across all users |

## Notes and Decisions

- **Song ownership**: Each user creates their own Song entry. Two users ranking "Bohemian Rhapsody" will have two separate Song items. GlobalRating aggregates them by normalized artist+title.
- **Rating is 1:1 with Song**: Since each user creates their own Song, there's exactly one Rating per Song. They could be a single item, but keeping them conceptually separate allows the Rating to carry only the mutable score while Song holds the static metadata.
- **Fixed genre/subgenre list**: Genres and subgenres come from a predefined list, not user-created. This keeps filtering consistent.
- **Max 100 songs per user**: The top 100 constraint means we never need pagination for a user's collection -- all 100 items fit in a single query response.
- **Global ranking recalculation**: When a user rates a song, the GlobalRating for that song (matched by normalized artist+title) is updated. This means writes to GlobalRating are eventual and may briefly show stale averages.
- **Search by title**: Only within a user's own collection (AP-007). Global search across all users' songs is not needed.
- **Analytics/reporting**: Not in scope. If needed later, could use DynamoDB Streams to export to a separate analytics system.
- **Audit trail**: Not needed for this version.
