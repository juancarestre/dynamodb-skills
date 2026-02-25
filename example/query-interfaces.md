# DynamoDB Query Interfaces

> Generated on: 2026-02-25
> Application: Music Ranking
> Language: TypeScript
> Input: .test/table-design.md
> Status: DRAFT

## Entity Models

### User

```typescript
// Entity: User
// Description: A registered user who creates and ranks songs
// PK: USER#<userId>
// SK: USER#<userId>
interface User {
  pk: string;                   // USER#<userId>
  sk: string;                   // USER#<userId>
  GSI1PK: string;               // USERNAME#<username>
  GSI1SK: string;               // USERNAME#<username>
  userId: string;               // ULID
  username: string;
  email: string;
  avatarUrl?: string;
  createdAt: string;            // ISO 8601
  entityType: "User";
}
```

### SongRating

```typescript
// Entity: SongRating (merged Song + Rating)
// Description: A song with metadata and the user's score (1-100)
// PK: USER#<userId>
// SK: SONG#<invertedScore>#<songId>
interface SongRating {
  pk: string;                   // USER#<userId>
  sk: string;                   // SONG#<invertedScore>#<songId>
  GSI1PK: string;               // USER#<userId>#GENRE#<genre>
  GSI1SK: string;               // SONG#<invertedScore>#<songId>
  GSI2PK: string;               // USER#<userId>#ARTIST#<artist>
  GSI2SK: string;               // SONG#<invertedScore>#<songId>
  songId: string;               // ULID
  userId: string;
  title: string;
  artist: string;
  album?: string;
  genre: string;
  subgenre?: string;
  year?: number;
  duration?: number;            // seconds
  coverArtUrl?: string;
  score: number;                // 1-100
  songKey: string;              // lowercase(artist)#lowercase(title)
  createdAt: string;            // ISO 8601
  updatedAt: string;            // ISO 8601
  entityType: "SongRating";
  titleLower: string;           // lowercase title for search
}
```

### SongRatingSub

```typescript
// Entity: SongRatingSub
// Description: Duplicate entry for subgenre filtering via GSI1
// PK: USER#<userId>
// SK: SONGSUB#<invertedScore>#<songId>
interface SongRatingSub {
  pk: string;                   // USER#<userId>
  sk: string;                   // SONGSUB#<invertedScore>#<songId>
  GSI1PK: string;               // USER#<userId>#SUBGENRE#<genre>#<subgenre>
  GSI1SK: string;               // SONG#<invertedScore>#<songId>
  songId: string;
  userId: string;
  title: string;
  artist: string;
  genre: string;
  subgenre: string;
  score: number;
  entityType: "SongRatingSub";
}
```

### SongRatingTitle

```typescript
// Entity: SongRatingTitle
// Description: Index entry for title prefix search via GSI2
// PK: USER#<userId>
// SK: SONGTITLE#<titleLower>#<songId>
interface SongRatingTitle {
  pk: string;                   // USER#<userId>
  sk: string;                   // SONGTITLE#<titleLower>#<songId>
  GSI2PK: string;               // USER#<userId>#TITLE
  GSI2SK: string;               // SONG#<titleLower>#<songId>
  songId: string;
  title: string;
  artist: string;
  score: number;
  entityType: "SongRatingTitle";
}
```

### GlobalRating

```typescript
// Entity: GlobalRating
// Description: Aggregated average score for a song across all users
// PK: GLOBAL
// SK: SONG#<invertedAvgScore>#<songKey>
interface GlobalRating {
  pk: string;                   // GLOBAL
  sk: string;                   // SONG#<invertedAvgScore>#<songKey>
  GSI1PK: string;               // GLOBAL#GENRE#<genre>
  GSI1SK: string;               // SONG#<invertedAvgScore>#<songKey>
  GSI2PK: string;               // GLOBAL#ARTIST#<artist>
  GSI2SK: string;               // SONG#<invertedAvgScore>#<songKey>
  songKey: string;              // lowercase(artist)#lowercase(title)
  title: string;
  artist: string;
  genre: string;
  subgenre?: string;
  averageScore: number;
  ratingCount: number;
  updatedAt: string;            // ISO 8601
  entityType: "GlobalRating";
}
```

## Key Builders

### Score Utilities

```typescript
// Inverts a score for sort key encoding.
// Score 1-100 -> inverted 099-000 (3-digit zero-padded).
// Ascending lexicographic order on inverted = descending by actual score.
function invertScore(score: number): string {
  return String(100 - score).padStart(3, "0");
}

// Recovers the original score from an inverted score string.
function recoverScore(inverted: string): number {
  return 100 - parseInt(inverted, 10);
}
```

### User Keys

```typescript
// Builds keys for User
// AP references: AP-001, AP-020, AP-023, AP-031
function buildUserKey(params: { userId: string }): { pk: string; sk: string } {
  return {
    pk: `USER#${params.userId}`,
    sk: `USER#${params.userId}`,
  };
}

// Builds GSI1 keys for User (username lookup)
// AP references: AP-002
function buildUserUsernameGSI(params: { username: string }): { GSI1PK: string; GSI1SK: string } {
  return {
    GSI1PK: `USERNAME#${params.username}`,
    GSI1SK: `USERNAME#${params.username}`,
  };
}
```

### SongRating Keys

```typescript
// Builds keys for SongRating (main item)
// AP references: AP-021, AP-022, AP-030
function buildSongRatingKey(params: {
  userId: string;
  score: number;
  songId: string;
}): { pk: string; sk: string } {
  return {
    pk: `USER#${params.userId}`,
    sk: `SONG#${invertScore(params.score)}#${params.songId}`,
  };
}

// Builds prefix for querying all songs of a user (top 100)
// AP references: AP-003
function buildSongRatingCollectionPrefix(params: { userId: string }): {
  pk: string;
  skPrefix: string;
} {
  return {
    pk: `USER#${params.userId}`,
    skPrefix: "SONG#",
  };
}

// Builds GSI1 keys for genre filtering
// AP references: AP-004
function buildSongRatingGenreGSI(params: {
  userId: string;
  genre: string;
  score: number;
  songId: string;
}): { GSI1PK: string; GSI1SK: string } {
  return {
    GSI1PK: `USER#${params.userId}#GENRE#${params.genre}`,
    GSI1SK: `SONG#${invertScore(params.score)}#${params.songId}`,
  };
}

// Builds GSI2 keys for artist filtering
// AP references: AP-006
function buildSongRatingArtistGSI(params: {
  userId: string;
  artist: string;
  score: number;
  songId: string;
}): { GSI2PK: string; GSI2SK: string } {
  return {
    GSI2PK: `USER#${params.userId}#ARTIST#${params.artist}`,
    GSI2SK: `SONG#${invertScore(params.score)}#${params.songId}`,
  };
}
```

### SongRatingSub Keys

```typescript
// Builds keys for SongRatingSub (subgenre index item)
// AP references: AP-021, AP-022, AP-030
function buildSongRatingSubKey(params: {
  userId: string;
  score: number;
  songId: string;
}): { pk: string; sk: string } {
  return {
    pk: `USER#${params.userId}`,
    sk: `SONGSUB#${invertScore(params.score)}#${params.songId}`,
  };
}

// Builds GSI1 keys for subgenre filtering
// AP references: AP-005
function buildSongRatingSubgenreGSI(params: {
  userId: string;
  genre: string;
  subgenre: string;
  score: number;
  songId: string;
}): { GSI1PK: string; GSI1SK: string } {
  return {
    GSI1PK: `USER#${params.userId}#SUBGENRE#${params.genre}#${params.subgenre}`,
    GSI1SK: `SONG#${invertScore(params.score)}#${params.songId}`,
  };
}
```

### SongRatingTitle Keys

```typescript
// Builds keys for SongRatingTitle (title search item)
// AP references: AP-021, AP-022, AP-030
function buildSongRatingTitleKey(params: {
  userId: string;
  titleLower: string;
  songId: string;
}): { pk: string; sk: string } {
  return {
    pk: `USER#${params.userId}`,
    sk: `SONGTITLE#${params.titleLower}#${params.songId}`,
  };
}

// Builds GSI2 keys for title search
// AP references: AP-007
function buildSongRatingTitleGSI(params: {
  userId: string;
  titleLower: string;
  songId: string;
}): { GSI2PK: string; GSI2SK: string } {
  return {
    GSI2PK: `USER#${params.userId}#TITLE`,
    GSI2SK: `SONG#${params.titleLower}#${params.songId}`,
  };
}

// Builds GSI2 prefix for title search with begins_with
// AP references: AP-007
function buildSongRatingTitleSearchPrefix(params: {
  userId: string;
  titlePrefix: string;
}): { GSI2PK: string; GSI2SKPrefix: string } {
  return {
    GSI2PK: `USER#${params.userId}#TITLE`,
    GSI2SKPrefix: `SONG#${params.titlePrefix.toLowerCase()}`,
  };
}
```

### GlobalRating Keys

```typescript
// Builds keys for GlobalRating
// AP references: AP-008, AP-021, AP-022, AP-030
function buildGlobalRatingKey(params: {
  averageScore: number;
  songKey: string;
}): { pk: string; sk: string } {
  return {
    pk: "GLOBAL",
    sk: `SONG#${invertScore(Math.floor(params.averageScore))}#${params.songKey}`,
  };
}

// Builds prefix for querying global top 100
// AP references: AP-008
function buildGlobalRatingCollectionPrefix(): { pk: string; skPrefix: string } {
  return {
    pk: "GLOBAL",
    skPrefix: "SONG#",
  };
}

// Builds GSI1 keys for global genre ranking
// AP references: AP-009
function buildGlobalRatingGenreGSI(params: {
  genre: string;
  averageScore: number;
  songKey: string;
}): { GSI1PK: string; GSI1SK: string } {
  return {
    GSI1PK: `GLOBAL#GENRE#${params.genre}`,
    GSI1SK: `SONG#${invertScore(Math.floor(params.averageScore))}#${params.songKey}`,
  };
}

// Builds GSI2 keys for global artist ranking
// AP references: AP-010
function buildGlobalRatingArtistGSI(params: {
  artist: string;
  averageScore: number;
  songKey: string;
}): { GSI2PK: string; GSI2SK: string } {
  return {
    GSI2PK: `GLOBAL#ARTIST#${params.artist}`,
    GSI2SK: `SONG#${invertScore(Math.floor(params.averageScore))}#${params.songKey}`,
  };
}
```

## Repository Interfaces

### UserRepository

```typescript
interface UserRepository {
  // AP-001: Get user by userId
  // Operation: Query
  // Table/GSI: Table
  // Key: USER#<userId> / = USER#<userId>
  // Params: Limit=1
  getUserById(params: { userId: string }): Promise<User | null>;

  // AP-002: Get user by username
  // Operation: Query
  // Table/GSI: GSI1
  // Key: USERNAME#<username> / = USERNAME#<username>
  // Params: Limit=1
  getUserByUsername(params: { username: string }): Promise<User | null>;

  // AP-020: Create user
  // Operation: PutItem
  // Key: buildUserKey({ userId })
  // Condition: attribute_not_exists(pk)
  // Note: Check username uniqueness via getUserByUsername before put
  createUser(params: {
    userId: string;
    username: string;
    email: string;
    avatarUrl?: string;
  }): Promise<User>;

  // AP-023: Update user profile
  // Operation: UpdateItem
  // Key: buildUserKey({ userId })
  // Condition: attribute_exists(pk)
  // Note: If username changes, GSI1PK/GSI1SK are updated automatically.
  //       Check new username uniqueness before updating.
  updateUser(params: {
    userId: string;
    username?: string;
    email?: string;
    avatarUrl?: string;
  }): Promise<void>;

  // AP-031: Delete user account
  // Operation: Query + BatchWriteItem (batched)
  // 1. Query all items with PK=USER#<userId>
  // 2. BatchWrite delete all items (batches of 25)
  // 3. For each song, recalculate its GlobalRating
  deleteUser(params: { userId: string }): Promise<void>;
}
```

### SongRatingRepository

```typescript
interface SongRatingRepository {
  // AP-003: Get user's top 100 (all songs with scores, highest first)
  // Operation: Query
  // Table/GSI: Table
  // Key: USER#<userId> / begins_with(SONG#)
  // Params: ScanIndexForward=true (inverted score = ascending SK = descending score)
  listUserTop100(params: { userId: string }): Promise<SongRating[]>;

  // AP-004: Get user's top songs filtered by genre
  // Operation: Query
  // Table/GSI: GSI1
  // Key: USER#<userId>#GENRE#<genre> / begins_with(SONG#)
  // Params: ScanIndexForward=true
  listUserSongsByGenre(params: {
    userId: string;
    genre: string;
  }): Promise<SongRating[]>;

  // AP-005: Get user's top songs filtered by subgenre
  // Operation: Query
  // Table/GSI: GSI1
  // Key: USER#<userId>#SUBGENRE#<genre>#<subgenre> / begins_with(SONG#)
  // Params: ScanIndexForward=true
  // Note: Reads from SongRatingSub items projected into GSI1
  listUserSongsBySubgenre(params: {
    userId: string;
    genre: string;
    subgenre: string;
  }): Promise<SongRating[]>;

  // AP-006: Get user's top songs filtered by artist
  // Operation: Query
  // Table/GSI: GSI2
  // Key: USER#<userId>#ARTIST#<artist> / begins_with(SONG#)
  // Params: ScanIndexForward=true
  listUserSongsByArtist(params: {
    userId: string;
    artist: string;
  }): Promise<SongRating[]>;

  // AP-007: Search user's songs by title prefix
  // Operation: Query
  // Table/GSI: GSI2
  // Key: USER#<userId>#TITLE / begins_with(SONG#<titlePrefix>)
  // Note: Reads from SongRatingTitle items projected into GSI2.
  //       titlePrefix is lowercased before query.
  searchUserSongsByTitle(params: {
    userId: string;
    titlePrefix: string;
  }): Promise<SongRatingTitle[]>;

  // AP-021: Add a song with rating
  // Operation: TransactWriteItems
  // Items:
  //   1. Put SongRating (condition: attribute_not_exists(pk))
  //   2. Put SongRatingSub (condition: attribute_not_exists(pk))
  //   3. Put SongRatingTitle (condition: attribute_not_exists(pk))
  //   4. Delete old GlobalRating + Put new GlobalRating (recalculate avg)
  // Note: App-level check that user has < 100 songs before calling
  addSongWithRating(params: {
    userId: string;
    songId: string;
    title: string;
    artist: string;
    album?: string;
    genre: string;
    subgenre?: string;
    year?: number;
    duration?: number;
    coverArtUrl?: string;
    score: number;
  }): Promise<SongRating>;

  // AP-022: Update rating score
  // Operation: TransactWriteItems
  // Items:
  //   1. Delete old SongRating + Put new SongRating (SK changes)
  //   2. Delete old SongRatingSub + Put new SongRatingSub (SK changes)
  //   3. Delete old SongRatingTitle + Put new SongRatingTitle (score attr changes)
  //   4. Delete old GlobalRating + Put new GlobalRating (recalculate avg)
  // Note: Caller must provide oldScore to build the old SK for deletion.
  updateRatingScore(params: {
    userId: string;
    songId: string;
    oldScore: number;
    newScore: number;
  }): Promise<void>;

  // AP-030: Remove song from collection
  // Operation: TransactWriteItems
  // Items:
  //   1. Delete SongRating
  //   2. Delete SongRatingSub
  //   3. Delete SongRatingTitle
  //   4. Delete old GlobalRating + Put updated GlobalRating (decrement count, recalculate avg)
  // Condition: Song must exist
  removeSong(params: {
    userId: string;
    songId: string;
    score: number;
    titleLower: string;
  }): Promise<void>;
}
```

### GlobalRatingRepository

```typescript
interface GlobalRatingRepository {
  // AP-008: Get global top 100 (all genres)
  // Operation: Query
  // Table/GSI: Table
  // Key: GLOBAL / begins_with(SONG#)
  // Params: ScanIndexForward=true, Limit=100
  listGlobalTop100(): Promise<GlobalRating[]>;

  // AP-009: Get global top songs by genre
  // Operation: Query
  // Table/GSI: GSI1
  // Key: GLOBAL#GENRE#<genre> / begins_with(SONG#)
  // Params: ScanIndexForward=true, Limit=100
  listGlobalTopByGenre(params: { genre: string }): Promise<GlobalRating[]>;

  // AP-010: Get global top songs by artist
  // Operation: Query
  // Table/GSI: GSI2
  // Key: GLOBAL#ARTIST#<artist> / begins_with(SONG#)
  // Params: ScanIndexForward=true, Limit=100
  listGlobalTopByArtist(params: { artist: string }): Promise<GlobalRating[]>;
}
```

## Transaction Definitions

### AddSongWithRatingTransaction

```typescript
// AP-021: Add song with rating
// Writes 3 items + updates GlobalRating (delete old + put new = 5 ops max)
// Total: 4-5 TransactWriteItems
interface AddSongWithRatingTransaction {
  items: [
    {
      operation: "Put";
      entity: "SongRating";
      key: ReturnType<typeof buildSongRatingKey>;
      gsi1: ReturnType<typeof buildSongRatingGenreGSI>;
      gsi2: ReturnType<typeof buildSongRatingArtistGSI>;
      condition: "attribute_not_exists(pk)";
    },
    {
      operation: "Put";
      entity: "SongRatingSub";
      key: ReturnType<typeof buildSongRatingSubKey>;
      gsi1: ReturnType<typeof buildSongRatingSubgenreGSI>;
      condition: "attribute_not_exists(pk)";
    },
    {
      operation: "Put";
      entity: "SongRatingTitle";
      key: ReturnType<typeof buildSongRatingTitleKey>;
      gsi2: ReturnType<typeof buildSongRatingTitleGSI>;
      condition: "attribute_not_exists(pk)";
    },
    {
      operation: "Delete";
      entity: "GlobalRating (old, if exists)";
      key: ReturnType<typeof buildGlobalRatingKey>;
      note: "Only if GlobalRating already exists for this songKey";
    },
    {
      operation: "Put";
      entity: "GlobalRating (new/updated)";
      key: ReturnType<typeof buildGlobalRatingKey>;
      note: "newAvg = ((oldAvg * oldCount) + score) / (oldCount + 1)";
    }
  ];
}
```

### UpdateRatingScoreTransaction

```typescript
// AP-022: Update rating score
// SK changes because inverted score is embedded in SK.
// Delete old items + Put new items + update GlobalRating = 7 ops
interface UpdateRatingScoreTransaction {
  items: [
    {
      operation: "Delete";
      entity: "SongRating (old SK)";
      key: "buildSongRatingKey({ userId, score: oldScore, songId })";
    },
    {
      operation: "Put";
      entity: "SongRating (new SK)";
      key: "buildSongRatingKey({ userId, score: newScore, songId })";
    },
    {
      operation: "Delete";
      entity: "SongRatingSub (old SK)";
      key: "buildSongRatingSubKey({ userId, score: oldScore, songId })";
    },
    {
      operation: "Put";
      entity: "SongRatingSub (new SK)";
      key: "buildSongRatingSubKey({ userId, score: newScore, songId })";
    },
    {
      operation: "Delete";
      entity: "SongRatingTitle (old, score attr changes)";
      key: "buildSongRatingTitleKey({ userId, titleLower, songId })";
    },
    {
      operation: "Put";
      entity: "SongRatingTitle (new score)";
      key: "buildSongRatingTitleKey({ userId, titleLower, songId })";
    },
    {
      operation: "Delete + Put";
      entity: "GlobalRating";
      note: "newAvg = ((oldAvg * oldCount) - oldScore + newScore) / oldCount";
    }
  ];
}
```

### RemoveSongTransaction

```typescript
// AP-030: Remove song from collection
// Delete 3 items + update GlobalRating = 4-5 ops
interface RemoveSongTransaction {
  items: [
    {
      operation: "Delete";
      entity: "SongRating";
      key: "buildSongRatingKey({ userId, score, songId })";
    },
    {
      operation: "Delete";
      entity: "SongRatingSub";
      key: "buildSongRatingSubKey({ userId, score, songId })";
    },
    {
      operation: "Delete";
      entity: "SongRatingTitle";
      key: "buildSongRatingTitleKey({ userId, titleLower, songId })";
    },
    {
      operation: "Delete + Put";
      entity: "GlobalRating";
      note: "newAvg = ((oldAvg * oldCount) - score) / (oldCount - 1). If oldCount == 1, delete GlobalRating entirely.";
    }
  ];
}
```

## Traceability Matrix

| AP ID | Description | Repository Method | Status |
|-------|-------------|-------------------|--------|
| AP-001 | Get user by userId | UserRepository.getUserById | Covered |
| AP-002 | Get user by username | UserRepository.getUserByUsername | Covered |
| AP-003 | User's top 100 | SongRatingRepository.listUserTop100 | Covered |
| AP-004 | User's top by genre | SongRatingRepository.listUserSongsByGenre | Covered |
| AP-005 | User's top by subgenre | SongRatingRepository.listUserSongsBySubgenre | Covered |
| AP-006 | User's top by artist | SongRatingRepository.listUserSongsByArtist | Covered |
| AP-007 | Search songs by title | SongRatingRepository.searchUserSongsByTitle | Covered |
| AP-008 | Global top 100 | GlobalRatingRepository.listGlobalTop100 | Covered |
| AP-009 | Global top by genre | GlobalRatingRepository.listGlobalTopByGenre | Covered |
| AP-010 | Global top by artist | GlobalRatingRepository.listGlobalTopByArtist | Covered |
| AP-020 | Create user | UserRepository.createUser | Covered |
| AP-021 | Add song with rating | SongRatingRepository.addSongWithRating | Covered |
| AP-022 | Update rating score | SongRatingRepository.updateRatingScore | Covered |
| AP-023 | Update user profile | UserRepository.updateUser | Covered |
| AP-030 | Remove song | SongRatingRepository.removeSong | Covered |
| AP-031 | Delete user account | UserRepository.deleteUser | Covered |

## Implementation Notes

- **SDK**: Use `@aws-sdk/client-dynamodb` with `@aws-sdk/lib-dynamodb` (v3 Document Client) for automatic marshalling/unmarshalling.
- **ULID generation**: Use a library like `ulid` or `ulidx` for generating sortable unique IDs.
- **Error handling**: Wrap DynamoDB calls in try/catch. `TransactWriteItems` throws `TransactionCanceledException` when conditions fail -- inspect `CancellationReasons` to determine which item failed.
- **Idempotency**: `addSongWithRating` uses `attribute_not_exists(pk)` conditions. If retried, the transaction fails safely without duplicating data.
- **Score update requires oldScore**: Since the score is embedded in the SK, updating a rating requires knowing the current score to build the old key for deletion. Read before write, or pass `oldScore` from the client.
- **GlobalRating atomicity**: The GlobalRating delete+put within a transaction ensures the average is recalculated atomically with the user's rating change. Brief staleness between the transaction commit and GSI propagation is acceptable.
- **Delete user (AP-031)**: This is a multi-step operation. Query all items for the user, then batch-delete in groups of 25. For each song, also update its GlobalRating. Consider running this as a background job or Step Function for large collections.
- **Testing**: Use DynamoDB Local (`amazon/dynamodb-local` Docker image) for integration tests. Create the table with both GSIs before running tests.
