# Offline-First Apps — Complete Deep Dive
### Local DBs · Sync Strategies · Conflict Resolution · Architecture

---

## Table of Contents

1. [What is Offline-First?](#1-what-is-offline-first)
2. [Why Offline-First is Different from Every Other App](#2-why-offline-first-is-different)
3. [The Three Tiers of Data Storage](#3-the-three-tiers-of-data-storage)
4. [Local Database Options (Complete Comparison)](#4-local-database-options)
5. [SQLite — The Foundation](#5-sqlite--the-foundation)
6. [WatermelonDB — Built for React Native](#6-watermelondb--built-for-react-native)
7. [Realm — Object Database](#7-realm--object-database)
8. [MMKV — Ultra-Fast Key-Value Store](#8-mmkv--ultra-fast-key-value-store)
9. [AsyncStorage — Simple Persistence](#9-asyncstorage--simple-persistence)
10. [Server-Side Database Considerations](#10-server-side-database-considerations)
11. [Sync Architecture — The Full Picture](#11-sync-architecture--the-full-picture)
12. [Sync Strategies (4 Types)](#12-sync-strategies-4-types)
13. [Conflict Resolution](#13-conflict-resolution)
14. [CRDTs — Conflict-Free Replicated Data Types](#14-crdts)
15. [Delta Sync vs Full Sync](#15-delta-sync-vs-full-sync)
16. [The Sync Engine (Building Your Own)](#16-the-sync-engine)
17. [Optimistic Updates](#17-optimistic-updates)
18. [Tombstoning (Soft Deletes)](#18-tombstoning-soft-deletes)
19. [Network State & Queue Management](#19-network-state--queue-management)
20. [Schema Migrations](#20-schema-migrations)
21. [Data Encryption at Rest](#21-data-encryption-at-rest)
22. [Real-World Offline-First Architectures](#22-real-world-architectures)

---

## 1. What is Offline-First?

Offline-first is an **architectural philosophy** where the app is designed to work **fully without a network connection** and syncs with a server when connectivity is available.

### Online-First (Traditional) vs Offline-First

```
ONLINE-FIRST APP:
  User opens screen
        ↓
  App calls API
        ↓
  Wait for response (spinner)
        ↓
  Show data
        ↓
  No internet? → show error screen
  Slow internet? → user stares at spinner

OFFLINE-FIRST APP:
  User opens screen
        ↓
  App reads from LOCAL database instantly (0ms)
        ↓
  Show data immediately (no spinner)
        ↓
  In background → try to sync with server
        ↓
  No internet? → app still works perfectly
  Slow internet? → user already sees data, sync happens silently
```

### The Core Mental Model Shift

> In an online-first app, the **server is the source of truth** and the client is a viewer.
> In an offline-first app, the **local database is the source of truth** and the server is the sync target.

This single shift changes **everything** about how you design the app.

---

## 2. Why Offline-First is Different

### Problem 1: You now have TWO databases

Every offline-first app has:
- A **local database** on the device (the one the UI reads from)
- A **server database** in the cloud (the one other devices sync to)

These two databases can diverge. Keeping them consistent is the central challenge.

### Problem 2: The same data can be modified in two places at once

```
Device A (no internet): User edits a task title → "Buy groceries"
Device B (with internet): Same user edits same task → "Buy vegetables"

Both sync to server.
Which version wins? CONFLICT.
```

### Problem 3: Deletes are dangerous

```
User deletes a record on Device A (offline)
Server still has the record
Device B syncs → downloads the "deleted" record again
        ↓
The delete is lost. Record comes back from the dead.
```

This is why offline-first apps use **tombstoning** (soft deletes), not hard deletes.

### Problem 4: Order of operations matters

```
Server state: count = 0

Device A (offline): count += 1  →  count = 1
Device B (offline): count += 1  →  count = 1

Both sync.
Expected result: count = 2
Naive "last write wins": count = 1  (WRONG)
```

You need **operational transforms** or **CRDTs** to handle this.

### Problem 5: Sync can happen at any time

- While user is actively using the app
- When app comes to foreground
- When network switches from WiFi to cellular
- From a background fetch
- When a push notification arrives (silent push)

Your UI must **react to sync changes** without disrupting what the user is doing.

---

## 3. The Three Tiers of Data Storage

```
┌──────────────────────────────────────────────────────┐
│  TIER 1 — IN-MEMORY (React State / Zustand / Redux)  │
│  Speed: ~0ms  |  Persists: No  |  Size: RAM          │
│  Use for: UI state, current screen data, temp data   │
└──────────────────────┬───────────────────────────────┘
                       │ read/write
┌──────────────────────▼───────────────────────────────┐
│  TIER 2 — LOCAL DATABASE (SQLite / Realm / WaterDB)  │
│  Speed: <5ms  |  Persists: Yes  |  Size: GBs         │
│  Use for: app data, user content, sync'd records     │
└──────────────────────┬───────────────────────────────┘
                       │ sync (background)
┌──────────────────────▼───────────────────────────────┐
│  TIER 3 — SERVER DATABASE (PostgreSQL / MongoDB etc) │
│  Speed: 100-500ms  |  Persists: Yes  |  Size: Unlim  │
│  Use for: source of truth, cross-device sync         │
└──────────────────────────────────────────────────────┘
```

### What goes in each tier

| Data Type | Tier | Why |
|---|---|---|
| Current selected tab | In-memory | Doesn't need to survive app kill |
| User's messages | Local DB | Needs offline access |
| User's profile photo URL | Local DB | Cached for offline |
| Auth token | Keychain (special) | Encrypted secure storage |
| App preferences (dark mode) | MMKV / AsyncStorage | Simple KV, survives restart |
| Cart items | Local DB + sync | Needs offline, cross-device |
| All historical orders | Server only + lazy load | Too much data to store locally |
| Frequently accessed records | Local DB (with expiry) | Cache, sync on stale |

---

## 4. Local Database Options

### Full Comparison Table

| Database | Type | Query Language | Performance | Offline Sync Built-in | Best For |
|---|---|---|---|---|---|
| **AsyncStorage** | Key-Value | JS object | Slow (JSON parse) | No | Simple preferences |
| **MMKV** | Key-Value | JS object | Ultra-fast (C++) | No | Settings, tokens, flags |
| **SQLite** | Relational | SQL | Fast | No | Complex queries, joins |
| **WatermelonDB** | Relational (SQLite) | JS Observable | Very Fast (lazy) | Yes (custom) | Large datasets, React Native |
| **Realm** | Object Database | JS / RealmQL | Fast | Yes (Atlas Sync) | Real-time sync apps |
| **RxDB** | Document | MongoDB-like | Fast | Yes (CouchDB/custom) | PWA + mobile hybrid |
| **SQLCipher** | Relational (encrypted) | SQL | Slightly slower | No | Sensitive data + SQL |

---

## 5. SQLite — The Foundation

SQLite is a **C-language relational database** that lives as a single `.db` file on the device. It's the most battle-tested local database, used by every major app (WhatsApp, Firefox, Spotify).

### Setup
```bash
npm install react-native-sqlite-storage
# OR the newer:
npm install react-native-quick-sqlite
```

### Basic Usage with react-native-quick-sqlite
```jsx
import { open } from 'react-native-quick-sqlite';

// Open (creates if not exists)
const db = open({ name: 'myapp.db' });

// Create table
db.execute(`
  CREATE TABLE IF NOT EXISTS tasks (
    id         TEXT PRIMARY KEY,
    title      TEXT NOT NULL,
    completed  INTEGER DEFAULT 0,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL,
    deleted_at INTEGER,          -- soft delete (tombstone)
    synced_at  INTEGER,          -- last time synced with server
    server_id  TEXT              -- server's UUID for this record
  )
`);

// Insert
db.execute(
  'INSERT INTO tasks (id, title, created_at, updated_at) VALUES (?, ?, ?, ?)',
  [uuid(), 'Buy groceries', Date.now(), Date.now()]
);

// Query
const { rows } = db.execute(
  'SELECT * FROM tasks WHERE deleted_at IS NULL ORDER BY created_at DESC'
);
const tasks = rows._array;

// Update
db.execute(
  'UPDATE tasks SET title = ?, updated_at = ? WHERE id = ?',
  ['Buy vegetables', Date.now(), taskId]
);

// Soft delete (tombstone)
db.execute(
  'UPDATE tasks SET deleted_at = ? WHERE id = ?',
  [Date.now(), taskId]
);
```

### Transactions (Atomic operations)
```jsx
// Either ALL succeed or ALL fail — critical for data consistency
db.transaction((tx) => {
  tx.executeSql('INSERT INTO orders (id, total) VALUES (?, ?)', [orderId, total]);
  tx.executeSql('UPDATE inventory SET stock = stock - ? WHERE id = ?', [qty, productId]);
  tx.executeSql('INSERT INTO order_items (order_id, product_id) VALUES (?, ?)', [orderId, productId]);
});
```

**Why transactions matter in offline-first:** If your app crashes mid-operation, a transaction ensures the database isn't left in a half-written, corrupt state.

### Indexes (Performance)
```sql
-- Without index: full table scan = O(n)
-- With index: B-tree lookup = O(log n)

CREATE INDEX idx_tasks_updated_at ON tasks(updated_at);
CREATE INDEX idx_tasks_user_id ON tasks(user_id);
CREATE INDEX idx_messages_room_id ON messages(room_id, created_at);
```

Always index columns you **filter or sort by**. For offline-first, always index `updated_at` — sync queries heavily use it (`WHERE updated_at > lastSyncTime`).

---

## 6. WatermelonDB — Built for React Native

WatermelonDB is built on top of SQLite but adds **lazy loading**, **observable queries**, and **sync primitives** designed specifically for React Native.

### Why WatermelonDB over raw SQLite

```
Raw SQLite:
  - You write SQL manually
  - You manage loading states
  - No reactivity — UI doesn't auto-update when DB changes
  - No built-in sync protocol

WatermelonDB:
  - Model classes (like ORM)
  - Reactive queries — UI automatically re-renders when data changes
  - Lazy loading — only loads data you actually use (critical for 100k+ records)
  - Built-in sync protocol with conflict resolution primitives
```

### Setup & Schema
```bash
npm install @nozbe/watermelondb
npm install @nozbe/with-observables
```

```jsx
// schema.js
import { appSchema, tableSchema } from '@nozbe/watermelondb';

export const mySchema = appSchema({
  version: 3,
  tables: [
    tableSchema({
      name: 'tasks',
      columns: [
        { name: 'title', type: 'string' },
        { name: 'completed', type: 'boolean' },
        { name: 'user_id', type: 'string', isIndexed: true },
        { name: 'created_at', type: 'number' },
        { name: 'updated_at', type: 'number' },
      ],
    }),
    tableSchema({
      name: 'comments',
      columns: [
        { name: 'body', type: 'string' },
        { name: 'task_id', type: 'string', isIndexed: true },
        { name: 'author_id', type: 'string' },
        { name: 'created_at', type: 'number' },
        { name: 'updated_at', type: 'number' },
      ],
    }),
  ],
});
```

### Model Classes
```jsx
// models/Task.js
import { Model, field, children } from '@nozbe/watermelondb';

export class Task extends Model {
  static table = 'tasks';
  static associations = {
    comments: { type: 'has_many', foreignKey: 'task_id' },
  };

  @field('title')        title;
  @field('completed')    completed;
  @field('user_id')      userId;
  @readonly @date('created_at') createdAt;
  @readonly @date('updated_at') updatedAt;

  @children('comments')  comments;

  @action async setCompleted(completed) {
    await this.update(task => {
      task.completed = completed;
    });
  }

  @action async addComment(body, authorId) {
    await this.collections.get('comments').create(comment => {
      comment.body = body;
      comment.taskId = this.id;
      comment.authorId = authorId;
    });
  }
}
```

### Reactive Queries (Auto-updating UI)
```jsx
import { withObservables } from '@nozbe/with-observables';
import { Q } from '@nozbe/watermelondb';

// HOC that re-renders when DB changes
const enhance = withObservables(['task'], ({ task }) => ({
  task,
  comments: task.comments.observe(),
}));

const TaskDetailsView = ({ task, comments }) => (
  <View>
    <Text>{task.title}</Text>
    {comments.map(c => <Text key={c.id}>{c.body}</Text>)}
  </View>
);

export default enhance(TaskDetailsView);

// List with filtering & sorting
const TaskList = withObservables([], ({ database }) => ({
  tasks: database
    .get('tasks')
    .query(
      Q.where('completed', false),
      Q.sortBy('created_at', Q.desc),
      Q.take(50)
    )
    .observe(),
}))(TaskListView);
```

### WatermelonDB Sync Protocol

WatermelonDB has a built-in sync protocol. You implement two API endpoints on your backend:

```jsx
import { synchronize } from '@nozbe/watermelondb/sync';

const sync = async () => {
  await synchronize({
    database,

    // Step 1: Pull — get changes from server since last sync
    pullChanges: async ({ lastPulledAt, schemaVersion, migration }) => {
      const { data } = await api.get('/sync/pull', {
        params: { lastPulledAt, schemaVersion },
      });
      // data = {
      //   changes: {
      //     tasks: { created: [...], updated: [...], deleted: ['id1', 'id2'] },
      //     comments: { created: [...], updated: [...], deleted: [] },
      //   },
      //   timestamp: 1234567890,
      // }
      return { changes: data.changes, timestamp: data.timestamp };
    },

    // Step 2: Push — send local changes to server
    pushChanges: async ({ changes, lastPulledAt }) => {
      // changes = same format as above, but YOUR local changes
      await api.post('/sync/push', { changes, lastPulledAt });
    },

    migrationsEnabledAtVersion: 1,
  });
};
```

**What happens under the hood:**
1. WatermelonDB tracks every local CREATE / UPDATE / DELETE automatically
2. On sync, it bundles all unsynced changes and sends them to server
3. Server responds with all server changes since `lastPulledAt`
4. WatermelonDB merges server changes into local DB
5. Marks local changes as synced
6. Updates `lastPulledAt` timestamp for next sync

---

## 7. Realm — Object Database

Realm is a **mobile-native object database**. Instead of tables and SQL, you work with JavaScript objects directly. MongoDB Atlas Device Sync provides real-time sync.

### Why Realm is different from SQLite

```
SQLite mindset:
  data lives in TABLES
  you SELECT rows with SQL
  UI reads data → you update UI manually

Realm mindset:
  data lives as OBJECTS (like JS classes)
  you query objects like JS arrays
  UI subscribes to objects → auto-updates on change (live objects)
```

### Setup & Schema
```bash
npm install realm @realm/react
```

```jsx
import Realm, { createRealmContext } from '@realm/react';

class Task extends Realm.Object {
  static schema = {
    name: 'Task',
    primaryKey: '_id',
    properties: {
      _id:       'objectId',
      title:     'string',
      completed: { type: 'bool', default: false },
      userId:    'string',
      createdAt: 'date',
      updatedAt: 'date',
    },
  };
}

const { RealmProvider, useRealm, useQuery, useObject } = createRealmContext({
  schema: [Task],
});
```

### CRUD with Realm
```jsx
const { useRealm, useQuery } = require('@realm/react');

const TaskList = () => {
  const realm = useRealm();

  // Live query — auto-rerenders when data changes
  const tasks = useQuery(Task, (collection) =>
    collection.filtered('completed == false').sorted('createdAt', true)
  );

  const addTask = (title) => {
    realm.write(() => {
      realm.create('Task', {
        _id: new Realm.BSON.ObjectId(),
        title,
        createdAt: new Date(),
        updatedAt: new Date(),
      });
    });
  };

  const toggleTask = (task) => {
    realm.write(() => {
      task.completed = !task.completed;
      task.updatedAt = new Date();
    });
  };

  const deleteTask = (task) => {
    realm.write(() => {
      realm.delete(task); // hard delete in Realm (soft delete = add deletedAt field)
    });
  };

  return (
    <FlatList
      data={tasks}
      renderItem={({ item }) => (
        <TouchableOpacity onPress={() => toggleTask(item)}>
          <Text>{item.title}</Text>
        </TouchableOpacity>
      )}
    />
  );
};
```

### Realm Atlas Device Sync (Real-Time Cloud Sync)
```jsx
import { AppProvider, UserProvider, useApp } from '@realm/react';
import Realm from 'realm';

// Sync config — Realm handles ALL sync automatically
const realmConfig = {
  schema: [Task],
  sync: {
    user: app.currentUser,
    flexible: true,        // Flexible Sync — you define what to sync
    initialSubscriptions: {
      update(subs, realm) {
        // Only sync THIS user's tasks
        subs.add(realm.objects('Task').filtered('userId == $0', userId));
      },
    },
    onError: (session, error) => console.error(error),
  },
};
```

**Realm Atlas Sync does for you:**
- Conflict resolution (last-write-wins by default)
- Delta sync (only changed fields, not entire objects)
- Real-time bi-directional sync
- Offline queue management
- Partition-based or flexible sync rules

**The tradeoff:** You're locked into MongoDB Atlas. WatermelonDB lets you use any backend.

---

## 8. MMKV — Ultra-Fast Key-Value Store

MMKV is a key-value store developed by WeChat (Tencent). Written in C++ using memory-mapped files. **10x faster than AsyncStorage**.

### Why MMKV exists

```
AsyncStorage:
  - Written in JavaScript/Java bridge
  - Uses JSON serialization
  - Async API only
  - Read benchmark: ~50ms for 1000 items

MMKV:
  - Written in C++ (native)
  - Binary encoding (protobuf)
  - Synchronous reads (no await needed)
  - Read benchmark: ~2ms for 1000 items
  - Memory-mapped: OS caches in RAM automatically
```

### Setup & Usage
```bash
npm install react-native-mmkv
```

```jsx
import { MMKV } from 'react-native-mmkv';

// Global storage
export const storage = new MMKV();

// Per-user storage (isolated)
export const userStorage = new MMKV({ id: `user-${userId}` });

// Encrypted storage
export const secureStorage = new MMKV({
  id: 'secure',
  encryptionKey: 'my-encryption-key', // store this in Keychain
});

// Write (synchronous!)
storage.set('theme', 'dark');
storage.set('userId', 12345);
storage.set('user', JSON.stringify({ name: 'Aashik', email: 'a@a.com' }));

// Read (synchronous!)
const theme = storage.getString('theme');         // 'dark'
const userId = storage.getNumber('userId');       // 12345
const user = JSON.parse(storage.getString('user'));

// Delete
storage.delete('theme');

// Check existence
const hasTheme = storage.contains('theme');
```

### Use MMKV for Zustand Persistence
```jsx
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { storage } from './mmkv';

// Bridge MMKV to Zustand's storage interface
const mmkvStorage = {
  setItem: (name, value) => storage.set(name, value),
  getItem: (name) => storage.getString(name) ?? null,
  removeItem: (name) => storage.delete(name),
};

const useSettingsStore = create(
  persist(
    (set) => ({
      theme: 'light',
      language: 'en',
      notificationsEnabled: true,
      setTheme: (theme) => set({ theme }),
    }),
    {
      name: 'settings',
      storage: createJSONStorage(() => mmkvStorage),
    }
  )
);
```

### What to store in MMKV vs SQLite

| MMKV | SQLite / WatermelonDB |
|---|---|
| Auth tokens | User messages |
| App settings / preferences | Products catalog |
| Feature flags | Orders history |
| Last sync timestamp | Tasks / notes |
| Cached user profile | Chat messages |
| A/B test assignments | Any relational data |

**Rule of thumb:** If it's a simple value you access often → MMKV. If it's structured data with queries → SQLite.

---

## 9. AsyncStorage — Simple Persistence

The original React Native key-value store. Asynchronous, unencrypted, stored as files on device.

```bash
npm install @react-native-async-storage/async-storage
```

```jsx
import AsyncStorage from '@react-native-async-storage/async-storage';

// Write
await AsyncStorage.setItem('onboardingDone', 'true');
await AsyncStorage.setItem('userPrefs', JSON.stringify({ darkMode: true }));

// Read
const done = await AsyncStorage.getItem('onboardingDone');
const prefs = JSON.parse(await AsyncStorage.getItem('userPrefs') ?? '{}');

// Delete
await AsyncStorage.removeItem('onboardingDone');

// Multi-read (faster than multiple getItem calls)
const [theme, lang] = await AsyncStorage.multiGet(['theme', 'language']);
```

### When to still use AsyncStorage
- Simple feature flags
- Onboarding completion status
- Migration from legacy code (MMKV is the modern replacement)

### When NOT to use AsyncStorage
- Auth tokens → use Keychain (encrypted)
- Large datasets → use SQLite
- Frequently read values → use MMKV (synchronous, faster)
- Sensitive data → use MMKV with encryption key or Keychain

---

## 10. Server-Side Database Considerations

The server database must be designed with sync in mind from day one. Retrofitting a standard REST API for offline sync is painful.

### Essential Server DB Schema Patterns

#### 1. Every record needs timestamps

```sql
-- Every synced table MUST have these columns
CREATE TABLE tasks (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES users(id),
  title       TEXT NOT NULL,
  completed   BOOLEAN DEFAULT false,

  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),  -- set via trigger
  deleted_at  TIMESTAMPTZ,                         -- NULL = alive, SET = deleted
  version     INTEGER NOT NULL DEFAULT 1           -- optimistic locking
);

-- Auto-update updated_at
CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON tasks
  FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
```

#### 2. Soft Deletes (Tombstones)

```sql
-- NEVER hard delete synced records
-- A hard delete cannot be propagated to clients
-- "I need to delete this" → set deleted_at, never DELETE FROM

-- Query living records
SELECT * FROM tasks WHERE deleted_at IS NULL;

-- The sync endpoint sends deleted record IDs to clients
SELECT id FROM tasks
WHERE user_id = $1
AND deleted_at IS NOT NULL
AND deleted_at > $lastPulledAt;
```

#### 3. The Pull Endpoint — What changed since last sync

```javascript
// Express.js — GET /sync/pull?lastPulledAt=1234567890
app.get('/sync/pull', authenticate, async (req, res) => {
  const { lastPulledAt } = req.query;
  const userId = req.user.id;
  const since = lastPulledAt ? new Date(Number(lastPulledAt)) : new Date(0);
  const now = Date.now();

  const [createdTasks, updatedTasks, deletedTaskIds] = await Promise.all([
    // New records since last sync
    db.query(`
      SELECT * FROM tasks
      WHERE user_id = $1
        AND created_at > $2
        AND deleted_at IS NULL
    `, [userId, since]),

    // Updated records since last sync
    db.query(`
      SELECT * FROM tasks
      WHERE user_id = $1
        AND updated_at > $2
        AND created_at <= $2
        AND deleted_at IS NULL
    `, [userId, since]),

    // Deleted records since last sync
    db.query(`
      SELECT id FROM tasks
      WHERE user_id = $1
        AND deleted_at > $2
    `, [userId, since]),
  ]);

  res.json({
    changes: {
      tasks: {
        created: createdTasks.rows,
        updated: updatedTasks.rows,
        deleted: deletedTaskIds.rows.map(r => r.id),
      },
    },
    timestamp: now,
  });
});
```

#### 4. The Push Endpoint — Accept client changes

```javascript
// POST /sync/push
app.post('/sync/push', authenticate, async (req, res) => {
  const { changes, lastPulledAt } = req.body;
  const userId = req.user.id;

  await db.transaction(async (trx) => {
    const { tasks } = changes;

    // Handle created records
    for (const task of tasks.created ?? []) {
      await trx.query(`
        INSERT INTO tasks (id, user_id, title, completed, created_at, updated_at)
        VALUES ($1, $2, $3, $4, $5, $6)
        ON CONFLICT (id) DO NOTHING  -- idempotent: safe to retry
      `, [task.id, userId, task.title, task.completed, task.created_at, task.updated_at]);
    }

    // Handle updated records
    for (const task of tasks.updated ?? []) {
      await trx.query(`
        UPDATE tasks
        SET title = $1, completed = $2, updated_at = $3
        WHERE id = $4
          AND user_id = $5
          AND (updated_at <= $3 OR $6::timestamptz IS NULL)
          -- Only update if server version isn't newer (basic conflict resolution)
      `, [task.title, task.completed, task.updated_at, task.id, userId, lastPulledAt]);
    }

    // Handle deleted records (soft delete)
    for (const id of tasks.deleted ?? []) {
      await trx.query(`
        UPDATE tasks SET deleted_at = NOW()
        WHERE id = $1 AND user_id = $2
      `, [id, userId]);
    }
  });

  res.json({ success: true });
});
```

---

## 11. Sync Architecture — The Full Picture

```
┌─────────────────────────────────────────────────────────────┐
│                        DEVICE                               │
│                                                             │
│  React UI (reads from local DB only — always instant)       │
│       ↕  observable queries                                 │
│  Local DB (WatermelonDB / SQLite / Realm)                   │
│       ↕  change tracking                                    │
│  Sync Engine                                                │
│       ↕  HTTP / WebSocket                                   │
└─────────────────┬───────────────────────────────────────────┘
                  │ internet (when available)
┌─────────────────▼───────────────────────────────────────────┐
│                        SERVER                               │
│                                                             │
│  Sync API (pull + push endpoints)                           │
│       ↕  SQL                                                │
│  Server DB (PostgreSQL / MySQL)                             │
│       ↕  change events                                      │
│  Change Data Capture (CDC) or Webhooks                      │
│       ↕  WebSocket push (optional real-time)                │
└─────────────────────────────────────────────────────────────┘
```

### The Sync Engine — What It Does

The sync engine is the brain of an offline-first app. It manages:

1. **Change tracking** — knows which local records are new/modified/deleted since last sync
2. **Conflict detection** — identifies when server and client both changed the same record
3. **Conflict resolution** — decides which version wins
4. **Queue management** — batches and retries failed sync attempts
5. **Timestamp management** — tracks `lastPulledAt` / `lastPushedAt`
6. **Error recovery** — handles partial sync failures without corruption

---

## 12. Sync Strategies (4 Types)

### Strategy 1: Pull-Only (Read-Only Sync)

The app only downloads data from the server. No local changes are pushed.

```
Server → Client: Always
Client → Server: Never

Use case: News app, stock prices, product catalog, read-only dashboards
```

```jsx
const pullOnlySync = async () => {
  const lastPulledAt = mmkv.getNumber('lastPulledAt') ?? 0;
  const { data } = await api.get('/sync/pull', { params: { lastPulledAt } });

  await db.transaction(() => {
    data.changes.products.created.forEach(p => db.execute(
      'INSERT OR REPLACE INTO products VALUES (?,?,?,?)',
      [p.id, p.name, p.price, p.updatedAt]
    ));
    data.changes.products.deleted.forEach(id => db.execute(
      'DELETE FROM products WHERE id = ?', [id]
    ));
  });

  mmkv.set('lastPulledAt', data.timestamp);
};
```

### Strategy 2: Push-Only (Write-Only Sync)

The app only sends data to the server. No server changes are pulled.

```
Server → Client: Never
Client → Server: Always

Use case: Analytics/telemetry, form submissions, IoT sensor data
```

```jsx
const pushOnlySync = async () => {
  const unsyncedEvents = await db.execute(
    'SELECT * FROM events WHERE synced_at IS NULL LIMIT 100'
  );

  if (!unsyncedEvents.rows.length) return;

  await api.post('/telemetry', { events: unsyncedEvents.rows._array });

  const ids = unsyncedEvents.rows._array.map(e => e.id);
  await db.execute(
    `UPDATE events SET synced_at = ? WHERE id IN (${ids.map(() => '?').join(',')})`,
    [Date.now(), ...ids]
  );
};
```

### Strategy 3: Two-Way Sync (Bidirectional)

Both client and server can create/update/delete. Changes flow both ways.

```
Server ↔ Client: Both directions

Use case: Notes app, task manager, chat, collaborative documents
```

This is the complex case — requires conflict resolution (see Section 13).

```jsx
const bidirectionalSync = async () => {
  const lastPulledAt = mmkv.getNumber('lastPulledAt') ?? 0;

  // PULL first, then PUSH
  // Order matters: pull first to detect conflicts before pushing

  // Step 1: Get server changes
  const { data: serverChanges } = await api.get('/sync/pull', {
    params: { lastPulledAt }
  });

  // Step 2: Get local unsynced changes
  const localChanges = await getLocalChanges(lastPulledAt);

  // Step 3: Detect conflicts (same record changed on both sides)
  const { resolved, toApply } = resolveConflicts(serverChanges, localChanges);

  // Step 4: Apply resolved server changes to local DB
  await applyServerChanges(toApply);

  // Step 5: Push local changes (minus conflicted ones already resolved)
  await api.post('/sync/push', { changes: resolved.localChanges, lastPulledAt });

  mmkv.set('lastPulledAt', serverChanges.timestamp);
};
```

### Strategy 4: Event Sourcing Sync

Instead of syncing the current state, you sync **events** (operations). The state is derived by replaying events.

```
Instead of: { taskTitle: "Buy groceries" }
You sync:   { event: "TITLE_CHANGED", from: "Buy milk", to: "Buy groceries", at: 123 }

Use case: Collaborative editing (Google Docs), financial ledgers, audit logs
```

```jsx
// Every mutation produces an event
const updateTaskTitle = async (taskId, newTitle) => {
  const event = {
    id: uuid(),
    type: 'TASK_TITLE_CHANGED',
    entityId: taskId,
    payload: { newTitle },
    userId: currentUser.id,
    timestamp: Date.now(),
    synced: false,
  };

  // Apply event locally
  await db.execute('INSERT INTO events VALUES (?,?,?,?,?,?)',
    [event.id, event.type, event.entityId, JSON.stringify(event.payload), event.timestamp, 0]
  );
  await db.execute('UPDATE tasks SET title = ? WHERE id = ?', [newTitle, taskId]);

  // Sync events to server
  await syncEvents();
};

// Server replays all events to rebuild state
const syncEvents = async () => {
  const unsyncedEvents = await getUnsyncedEvents();
  await api.post('/events', { events: unsyncedEvents });
  await markEventsSynced(unsyncedEvents.map(e => e.id));
};
```

---

## 13. Conflict Resolution

A conflict happens when the **same record** is modified on **two different clients** (or client + server) before they sync.

### The 4 Conflict Resolution Strategies

---

#### Strategy 1: Last Write Wins (LWW)

The record with the **most recent `updated_at` timestamp** overwrites the other.

```
Device A: updated task at 10:00:05  →  "Buy groceries"
Device B: updated task at 10:00:03  →  "Buy vegetables"

Result: "Buy groceries" wins (10:00:05 > 10:00:03)
```

```jsx
const resolveConflictLWW = (localRecord, serverRecord) => {
  return localRecord.updatedAt > serverRecord.updatedAt
    ? localRecord
    : serverRecord;
};
```

**Pros:** Simple, easy to implement, predictable
**Cons:** Data loss — one side's changes disappear silently. User on Device B loses their edit with no warning.

**Best for:** User preferences, profile settings, anything where losing an edit is acceptable.

---

#### Strategy 2: Server Wins

The server's version always wins. Client changes that conflict are discarded.

```jsx
const resolveConflictServerWins = (localRecord, serverRecord) => {
  return serverRecord; // always
};
```

**Pros:** Simple, no data loss on server
**Cons:** User's offline edits silently discarded — very bad UX

**Best for:** Financial transactions, inventory levels, anything where server authority is critical.

---

#### Strategy 3: Client Wins

Local changes always win. Server is just a sync target.

```jsx
const resolveConflictClientWins = (localRecord, serverRecord) => {
  return localRecord; // always
};
```

**Pros:** Offline changes never lost
**Cons:** Malicious or buggy clients can corrupt server data

**Best for:** User-owned private data (personal notes, drafts).

---

#### Strategy 4: Field-Level Merge (Best for UX)

Instead of resolving at the **record** level, merge at the **field** level. Each field is independently resolved.

```
Server version of task:
  { title: "Buy milk", completed: true,  priority: "high" }
  (user B completed it and changed priority)

Local version of task:
  { title: "Buy groceries", completed: false, priority: "medium" }
  (user A changed the title)

Field-level merge result:
  { title: "Buy groceries", completed: true, priority: "high" }
  (title from local, completed + priority from server)
```

```jsx
const fieldLevelMerge = (localRecord, serverRecord, baseRecord) => {
  // baseRecord = the common ancestor (what both started from before diverging)
  const merged = { ...baseRecord };

  for (const field of Object.keys(baseRecord)) {
    const serverChanged = serverRecord[field] !== baseRecord[field];
    const localChanged  = localRecord[field]  !== baseRecord[field];

    if (serverChanged && !localChanged) {
      merged[field] = serverRecord[field]; // server changed this field
    } else if (localChanged && !serverChanged) {
      merged[field] = localRecord[field];  // local changed this field
    } else if (serverChanged && localChanged) {
      // Both changed the same field — need a field-level strategy
      merged[field] = resolveFieldConflict(field, localRecord[field], serverRecord[field]);
    }
    // Both unchanged: keep base value
  }

  return merged;
};

const resolveFieldConflict = (fieldName, localValue, serverValue) => {
  // Numeric fields: could add (increments), max, or latest
  if (fieldName === 'likeCount') return Math.max(localValue, serverValue);
  // Default: server wins for individual conflicting fields
  return serverValue;
};
```

**Best for:** Collaborative apps, documents, anything with multiple editable fields.

---

#### Strategy 5: Show Conflict to User (Manual Resolution)

For critical data, show the user both versions and let them choose.

```jsx
const handleConflict = (localRecord, serverRecord) => {
  setConflictData({ local: localRecord, server: serverRecord });
  setShowConflictModal(true);
};

// ConflictModal.tsx
const ConflictModal = ({ local, server, onResolve }) => (
  <Modal>
    <Text>Conflict detected! Which version do you want to keep?</Text>
    <TouchableOpacity onPress={() => onResolve(local)}>
      <Text>My version (edited {formatDate(local.updatedAt)})</Text>
      <Text>{local.title}</Text>
    </TouchableOpacity>
    <TouchableOpacity onPress={() => onResolve(server)}>
      <Text>Server version (edited {formatDate(server.updatedAt)})</Text>
      <Text>{server.title}</Text>
    </TouchableOpacity>
  </Modal>
);
```

**Best for:** Medical records, legal documents, financial data.

---

## 14. CRDTs

### What is a CRDT?

**Conflict-free Replicated Data Type** — a data structure mathematically designed so that **concurrent updates always converge to the same result**, regardless of the order they're applied. No conflict resolution logic needed.

```
Without CRDT:
  Device A sets count = 5
  Device B sets count = 3
  Who wins? You need a rule.

With a CRDT Counter:
  Device A increments by +2 (from 0)
  Device B increments by +3 (from 0)
  Both sync — result is always 5, regardless of order.
  Math guarantees it. No conflict logic needed.
```

### CRDT Types and Use Cases

#### G-Counter (Grow-only counter — no decrements)
```jsx
// Each device has its own "slot" it can only increment
// Final value = sum of all slots

class GCounter {
  constructor(nodeId) {
    this.nodeId = nodeId;
    this.counts = {};
  }

  increment() {
    this.counts[this.nodeId] = (this.counts[this.nodeId] ?? 0) + 1;
  }

  value() {
    return Object.values(this.counts).reduce((a, b) => a + b, 0);
  }

  merge(other) {
    const merged = { ...this.counts };
    for (const [node, count] of Object.entries(other.counts)) {
      merged[node] = Math.max(merged[node] ?? 0, count);
    }
    return new GCounter(this.nodeId, merged);
  }
}

// Use case: Like counts, view counts, download counts
```

#### LWW-Register (Last-Write-Wins Register)
```jsx
// Attaches a timestamp to each value — highest timestamp wins automatically
class LWWRegister {
  constructor() {
    this.value = null;
    this.timestamp = 0;
  }

  set(value) {
    this.value = value;
    this.timestamp = Date.now();
  }

  merge(other) {
    if (other.timestamp > this.timestamp) {
      this.value = other.value;
      this.timestamp = other.timestamp;
    }
  }
}
// Use case: user's display name, avatar URL, any "latest value" field
```

#### OR-Set (Observed-Remove Set — add/remove with no conflicts)
```jsx
// Each element has a unique tag
// Add = add tag to "add set"
// Remove = move tag to "remove set"
// Element is in set if it's in add-set but NOT in remove-set

class ORSet {
  constructor() {
    this.addSet = new Map();   // element → Set of tags
    this.removeSet = new Set(); // Set of tags
  }

  add(element) {
    const tag = uuid();
    if (!this.addSet.has(element)) this.addSet.set(element, new Set());
    this.addSet.get(element).add(tag);
    return tag;
  }

  remove(element) {
    const tags = this.addSet.get(element) ?? new Set();
    tags.forEach(tag => this.removeSet.add(tag));
  }

  has(element) {
    const tags = this.addSet.get(element) ?? new Set();
    return [...tags].some(tag => !this.removeSet.has(tag));
  }

  merge(other) {
    // Merge add sets (union)
    for (const [el, tags] of other.addSet) {
      if (!this.addSet.has(el)) this.addSet.set(el, new Set());
      tags.forEach(tag => this.addSet.get(el).add(tag));
    }
    // Merge remove sets (union)
    other.removeSet.forEach(tag => this.removeSet.add(tag));
  }
}
// Use case: collaborative tag lists, shared todo lists, shopping carts
```

#### Text CRDT (Collaborative Editing)
Used by tools like Figma, Notion, Google Docs. Libraries: **Yjs**, **Automerge**.

```bash
npm install yjs y-websocket
```

```jsx
import * as Y from 'yjs';
import { WebsocketProvider } from 'y-websocket';

const doc = new Y.Doc();
const wsProvider = new WebsocketProvider('ws://server', 'room-id', doc);

// Shared text type — concurrent edits never conflict
const yText = doc.getText('document');

yText.insert(0, 'Hello');
// Device B simultaneously inserts 'World' at position 5
// Both sync — result is deterministic ("Hello World" or "WorldHello" based on timestamps)
// No conflict, no data loss, no manual resolution needed
```

---

## 15. Delta Sync vs Full Sync

### Full Sync

Download the **entire dataset** every time you sync.

```jsx
const fullSync = async () => {
  // Downloads everything every time
  const { data } = await api.get('/data/all');
  await db.execute('DELETE FROM tasks');
  await insertAll('tasks', data.tasks);
};
```

**When to use:** App first install, small datasets (<1000 records), simple read-only apps
**Never use for:** Large datasets, frequent syncs, mobile data connections

### Delta Sync (Incremental Sync)

Only download **what changed** since the last sync.

```jsx
const deltaSync = async () => {
  const lastPulledAt = mmkv.getNumber('lastPulledAt') ?? 0;

  // Only get records changed since lastPulledAt
  const { data } = await api.get('/sync/pull', {
    params: { since: lastPulledAt }
  });

  // Apply only the delta
  await applyChanges(data.changes);
  mmkv.set('lastPulledAt', data.timestamp);
};
```

**Comparison:**

| | Full Sync | Delta Sync |
|---|---|---|
| Bandwidth | High (entire dataset) | Low (only changes) |
| Server load | High | Low |
| Complexity | Low | Medium |
| First sync | Same | Same (must do full) |
| Subsequent syncs | Slow | Fast |
| Handles deletes | Yes (replace all) | Needs tombstones |

### Hybrid: Full Sync on First Install, Delta After

```jsx
const smartSync = async () => {
  const lastPulledAt = mmkv.getNumber('lastPulledAt');

  if (!lastPulledAt) {
    // First time — full sync
    await fullSync();
  } else {
    // Subsequent — delta sync
    await deltaSync(lastPulledAt);
  }
};
```

---

## 16. The Sync Engine

### Building a Complete Sync Engine

```jsx
class SyncEngine {
  constructor({ db, api, storage }) {
    this.db = db;
    this.api = api;
    this.storage = storage;
    this.isSyncing = false;
    this.syncQueue = [];
  }

  // Entry point — call this whenever you want to sync
  async sync() {
    if (this.isSyncing) {
      // Queue for after current sync completes
      return new Promise((resolve) => this.syncQueue.push(resolve));
    }

    this.isSyncing = true;
    const startTime = Date.now();

    try {
      const lastPulledAt = this.storage.getNumber('lastPulledAt') ?? 0;

      // Phase 1: Get server changes
      const serverChanges = await this.pull(lastPulledAt);

      // Phase 2: Get local changes
      const localChanges = await this.getLocalChanges(lastPulledAt);

      // Phase 3: Detect and resolve conflicts
      const { resolvedServerChanges, resolvedLocalChanges } =
        this.resolveConflicts(serverChanges.changes, localChanges);

      // Phase 4: Apply server changes to local DB
      await this.applyServerChanges(resolvedServerChanges);

      // Phase 5: Push local changes
      if (Object.keys(resolvedLocalChanges).some(k => resolvedLocalChanges[k].length > 0)) {
        await this.push(resolvedLocalChanges, lastPulledAt);
      }

      // Phase 6: Update sync timestamp
      this.storage.set('lastPulledAt', serverChanges.timestamp);
      this.storage.set('lastSyncedAt', Date.now());

      console.log(`Sync completed in ${Date.now() - startTime}ms`);

    } catch (error) {
      // Don't update lastPulledAt on failure — retry from same point
      this.storage.set('lastSyncError', error.message);
      throw error;
    } finally {
      this.isSyncing = false;
      // Process queued syncs
      const queued = this.syncQueue.shift();
      if (queued) { queued(); this.sync(); }
    }
  }

  async pull(since) {
    const response = await this.api.get('/sync/pull', { params: { since } });
    return response.data;
  }

  async push(changes, lastPulledAt) {
    await this.api.post('/sync/push', { changes, lastPulledAt });
    await this.markLocalChangesSynced(changes);
  }

  async getLocalChanges(since) {
    // Get records created/updated/deleted after last sync
    const changes = {};
    for (const table of SYNCED_TABLES) {
      const created = await this.db.execute(
        `SELECT * FROM ${table} WHERE created_at > ? AND deleted_at IS NULL`, [since]
      );
      const updated = await this.db.execute(
        `SELECT * FROM ${table} WHERE updated_at > ? AND created_at <= ? AND deleted_at IS NULL`, [since, since]
      );
      const deleted = await this.db.execute(
        `SELECT id FROM ${table} WHERE deleted_at > ?`, [since]
      );
      changes[table] = {
        created: created.rows._array,
        updated: updated.rows._array,
        deleted: deleted.rows._array.map(r => r.id),
      };
    }
    return changes;
  }

  resolveConflicts(serverChanges, localChanges) {
    // For each table, check if same record was changed on both sides
    const resolvedServer = {};
    const resolvedLocal = {};

    for (const table of SYNCED_TABLES) {
      const serverUpdated = serverChanges[table]?.updated ?? [];
      const localUpdated  = localChanges[table]?.updated ?? [];

      const localIds = new Set(localUpdated.map(r => r.id));
      const conflicts = serverUpdated.filter(r => localIds.has(r.id));

      const nonConflictingServer = serverUpdated.filter(r => !localIds.has(r.id));
      const nonConflictingLocal  = localUpdated.filter(r => !conflicts.find(c => c.id === r.id));

      // Resolve each conflict
      const resolvedConflicts = conflicts.map(serverRecord => {
        const localRecord = localUpdated.find(r => r.id === serverRecord.id);
        return fieldLevelMerge(localRecord, serverRecord);
      });

      resolvedServer[table] = {
        ...serverChanges[table],
        updated: [...nonConflictingServer, ...resolvedConflicts],
      };
      resolvedLocal[table] = {
        ...localChanges[table],
        updated: nonConflictingLocal,
      };
    }

    return { resolvedServerChanges: resolvedServer, resolvedLocalChanges: resolvedLocal };
  }
}

// Singleton instance
export const syncEngine = new SyncEngine({ db, api, storage: mmkv });
```

### When to Trigger Sync

```jsx
// 1. App comes to foreground
useEffect(() => {
  const sub = AppState.addEventListener('change', (state) => {
    if (state === 'active') syncEngine.sync().catch(console.error);
  });
  return () => sub.remove();
}, []);

// 2. Network reconnects
useEffect(() => {
  const unsubscribe = NetInfo.addEventListener((state) => {
    if (state.isConnected && !state.isInternetReachable === false) {
      syncEngine.sync().catch(console.error);
    }
  });
  return unsubscribe;
}, []);

// 3. User performs a write action (push immediately)
const createTask = async (title) => {
  await db.execute('INSERT INTO tasks ...', [...]);
  syncEngine.sync().catch(console.error); // fire and forget
};

// 4. Silent push notification from server ("hey, new data for you")
messaging().setBackgroundMessageHandler(async (msg) => {
  if (msg.data?.type === 'sync') await syncEngine.sync();
});

// 5. Background fetch (periodic)
BackgroundFetch.configure({ minimumFetchInterval: 15 }, async (taskId) => {
  await syncEngine.sync();
  BackgroundFetch.finish(taskId);
});
```

---

## 17. Optimistic Updates

Don't wait for the server to confirm before updating the UI. Apply changes locally first, sync in background.

```
WITHOUT optimistic updates:
  User taps "Complete task"
  → Spinner shown
  → Wait 300ms for API response
  → UI updates
  (Feels slow, requires internet)

WITH optimistic updates:
  User taps "Complete task"
  → UI updates INSTANTLY (local write)
  → Sync happens in background
  → If sync fails → rollback UI with error toast
  (Feels instant, works offline)
```

```jsx
const toggleTask = async (task) => {
  // 1. Optimistic: update local DB immediately
  await db.execute(
    'UPDATE tasks SET completed = ?, updated_at = ? WHERE id = ?',
    [!task.completed, Date.now(), task.id]
  );

  // 2. Sync in background (UI already updated)
  try {
    await syncEngine.sync();
  } catch (err) {
    // 3. Rollback if sync fails and we're online (indicates a real error, not just offline)
    const networkState = await NetInfo.fetch();
    if (networkState.isConnected) {
      await db.execute(
        'UPDATE tasks SET completed = ?, updated_at = ? WHERE id = ?',
        [task.completed, task.updatedAt, task.id] // restore original
      );
      showToast('Failed to save changes. Please try again.');
    }
    // If offline: keep optimistic update, queue for later sync (offline queue)
  }
};
```

---

## 18. Tombstoning (Soft Deletes)

### Why Hard Deletes Break Offline Sync

```
Without tombstones:
  Device A deletes Task #42 (offline)
  Device B has Task #42 in its DB
  
  When sync happens:
    Device A pushes DELETE → server deletes #42
    Device B pulls changes → no record of #42 being deleted
    Device B keeps Task #42 in its DB forever ← ZOMBIE RECORD
```

### With Tombstones

```jsx
// "Delete" = mark as deleted, not actual deletion
const deleteTask = async (taskId) => {
  await db.execute(
    'UPDATE tasks SET deleted_at = ? WHERE id = ?',
    [Date.now(), taskId]
  );
  // Sync sends: { tasks: { deleted: ['taskId'] } }
  // Server marks deleted_at on its copy
  // All devices eventually receive the tombstone and hide/delete the record
};

// Always filter out tombstoned records in queries
const getLiveTasks = () =>
  db.execute('SELECT * FROM tasks WHERE deleted_at IS NULL');

// Purge old tombstones after a safe period (e.g., 30 days)
// Once all devices have synced past that timestamp, the tombstone can be hard-deleted
const purgeStaleTombstones = async () => {
  const thirtyDaysAgo = Date.now() - (30 * 24 * 60 * 60 * 1000);
  await db.execute(
    'DELETE FROM tasks WHERE deleted_at IS NOT NULL AND deleted_at < ?',
    [thirtyDaysAgo]
  );
};
```

---

## 19. Network State & Queue Management

### Offline Queue Pattern

```jsx
import NetInfo from '@react-native-community/netinfo';

class OfflineQueue {
  constructor({ storage, onProcess }) {
    this.storageKey = 'offline_queue';
    this.storage = storage;
    this.onProcess = onProcess;
    this.isProcessing = false;

    // Start listening for connectivity
    NetInfo.addEventListener((state) => {
      if (state.isConnected) this.processQueue();
    });
  }

  async enqueue(action) {
    const queue = this.getQueue();
    queue.push({
      id: uuid(),
      action,
      enqueuedAt: Date.now(),
      attempts: 0,
    });
    this.storage.set(this.storageKey, JSON.stringify(queue));
  }

  getQueue() {
    const raw = this.storage.getString(this.storageKey);
    return raw ? JSON.parse(raw) : [];
  }

  async processQueue() {
    if (this.isProcessing) return;
    const queue = this.getQueue();
    if (!queue.length) return;

    this.isProcessing = true;
    const remaining = [];

    for (const item of queue) {
      try {
        await this.onProcess(item.action);
        // Success: don't re-queue
      } catch (err) {
        item.attempts += 1;
        // Exponential backoff: retry up to 5 times
        if (item.attempts < 5) {
          remaining.push(item);
        } else {
          // Dead letter — log to crash reporter
          console.error('Dropped action after 5 attempts:', item);
        }
      }
    }

    this.storage.set(this.storageKey, JSON.stringify(remaining));
    this.isProcessing = false;
  }
}

export const offlineQueue = new OfflineQueue({
  storage: mmkv,
  onProcess: (action) => api.post('/actions', action),
});

// Usage
offlineQueue.enqueue({ type: 'LIKE_POST', postId: '123' });
```

---

## 20. Schema Migrations

When you update your app and add new columns / tables, existing users have old schema in their local DB. Migrations upgrade their local DB without losing data.

### SQLite Migration

```jsx
const CURRENT_VERSION = 4;

const runMigrations = async () => {
  const { rows } = db.execute('PRAGMA user_version');
  const currentVersion = rows._array[0].user_version;

  if (currentVersion === CURRENT_VERSION) return;

  db.transaction((tx) => {
    if (currentVersion < 2) {
      tx.executeSql('ALTER TABLE tasks ADD COLUMN priority TEXT DEFAULT "medium"');
    }
    if (currentVersion < 3) {
      tx.executeSql('ALTER TABLE tasks ADD COLUMN due_date INTEGER');
      tx.executeSql('CREATE INDEX idx_tasks_due_date ON tasks(due_date)');
    }
    if (currentVersion < 4) {
      tx.executeSql(`
        CREATE TABLE IF NOT EXISTS tags (
          id TEXT PRIMARY KEY,
          name TEXT NOT NULL,
          color TEXT,
          created_at INTEGER
        )
      `);
      tx.executeSql('ALTER TABLE tasks ADD COLUMN tag_id TEXT REFERENCES tags(id)');
    }
    tx.executeSql(`PRAGMA user_version = ${CURRENT_VERSION}`);
  });
};

// Run on app start
await runMigrations();
```

### WatermelonDB Migration
```jsx
import { schemaMigrations, addColumns, createTable } from '@nozbe/watermelondb/Schema/migrations';

export const migrations = schemaMigrations({
  migrations: [
    {
      toVersion: 2,
      steps: [
        addColumns({ table: 'tasks', columns: [{ name: 'priority', type: 'string' }] }),
      ],
    },
    {
      toVersion: 3,
      steps: [
        addColumns({ table: 'tasks', columns: [{ name: 'due_date', type: 'number', isOptional: true }] }),
        createTable({ name: 'tags', columns: [
          { name: 'name', type: 'string' },
          { name: 'color', type: 'string' },
        ]}),
      ],
    },
  ],
});
```

**Golden rule for migrations:** NEVER rename or drop columns (data loss risk). Instead:
- Add new columns with defaults
- Copy data from old → new column in migration
- Leave old column as deprecated (remove in a later version after everyone has migrated)

---

## 21. Data Encryption at Rest

### Why encrypt local DB?

On a rooted Android device or jailbroken iPhone, the local SQLite file can be read by anyone. If it contains messages, medical records, or financial data — it must be encrypted.

### SQLCipher (Encrypted SQLite)

```bash
npm install react-native-sqlcipher-storage
```

```jsx
const db = SQLite.openDatabase({
  name: 'secure.db',
  key: encryptionKey,  // derive from user password or Keychain
  location: 'default',
});
```

### Getting the Encryption Key Safely

```jsx
import * as Keychain from 'react-native-keychain';
import { randomBytes } from 'react-native-randombytes';

const getOrCreateDbKey = async () => {
  // Try to get existing key from Keychain
  const stored = await Keychain.getGenericPassword({ service: 'db_key' });
  if (stored) return stored.password;

  // First time: generate a random 256-bit key
  const key = (await randomBytes(32)).toString('hex');

  // Store in Keychain (encrypted by OS Secure Enclave / StrongBox)
  await Keychain.setGenericPassword('db', key, {
    service: 'db_key',
    accessible: Keychain.ACCESSIBLE.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
  });

  return key;
};
```

### MMKV Encryption

```jsx
const key = await getOrCreateDbKey();

const secureStorage = new MMKV({
  id: 'secure-storage',
  encryptionKey: key,
});
```

---

## 22. Real-World Architectures

### Architecture 1: Notes App (Notion / Apple Notes style)

```
Local: WatermelonDB (offline-first, reactive queries)
Sync: Custom two-way delta sync (pull + push)
Conflict: Field-level merge (title, body, tags synced independently)
Server: PostgreSQL with updated_at + deleted_at columns
Push: Silent push when another device makes changes
```

### Architecture 2: Messaging App (WhatsApp style)

```
Local: SQLite (raw, for maximum control + custom encryption)
Sync: WebSocket for real-time delivery + REST for history
Conflict: No conflict — messages are append-only (never edited in-place)
Server: Cassandra / ScyllaDB (optimized for time-series message storage)
Offline: Messages queue locally, send when online
Encryption: End-to-end (messages encrypted before leaving device)
```

### Architecture 3: E-Commerce App (cart, orders)

```
Local: MMKV for cart + SQLite for order history
Sync: Pull-only for catalog, push-only for orders
Conflict: Server wins for prices/stock (client cannot override)
Server: PostgreSQL
Offline: Cart works offline (MMKV), checkout requires internet
```

### Architecture 4: Collaborative Task Manager (Linear / Asana style)

```
Local: Realm with Atlas Device Sync
Sync: Automatic real-time (Realm handles everything)
Conflict: LWW per field (Realm built-in)
Server: MongoDB Atlas
Push: WebSocket subscription for real-time updates
```

---

## The Offline-First Checklist

Before shipping an offline-first feature, verify:

- [ ] UI reads exclusively from local DB (never waits for API)
- [ ] All writes go to local DB first, then sync
- [ ] Every record has `created_at`, `updated_at`, `deleted_at`
- [ ] Soft deletes (tombstones) implemented — no hard deletes
- [ ] Conflict resolution strategy defined for every table
- [ ] Schema migrations written for new columns
- [ ] Sync is idempotent (safe to run multiple times)
- [ ] Local DB is encrypted if it contains sensitive data
- [ ] Sync errors don't corrupt `lastPulledAt` (only update on success)
- [ ] Offline queue drains when network reconnects
- [ ] Sync triggers on: app foreground, network reconnect, write actions, silent push
- [ ] Tombstone purge runs periodically (no unbounded DB growth)
- [ ] Tested with Airplane Mode on a real device

---

## One-Line Summary of Every DB

| DB | One line |
|---|---|
| **AsyncStorage** | Simple async key-value, the jQuery of local storage — works but slow |
| **MMKV** | C++ key-value, synchronous reads, 10x faster than AsyncStorage |
| **SQLite** | Full relational DB in a file — SQL queries, transactions, indexes |
| **WatermelonDB** | SQLite + reactive queries + sync protocol, built for React Native at scale |
| **Realm** | Object DB with live objects and optional Atlas real-time cloud sync |
| **RxDB** | Document DB (like MongoDB) with CouchDB/custom sync, good for offline PWAs |
| **SQLCipher** | Encrypted SQLite — same API, every byte on disk is AES-256 encrypted |
