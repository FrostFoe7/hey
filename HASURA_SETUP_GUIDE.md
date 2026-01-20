# Hasura + PostgreSQL Setup Guide for Hey

## 🎯 Complete Setup Plan

---

# PART 1: INFRASTRUCTURE SETUP

## Option A: Quick Start (Development)
```bash
# 1. Install Docker
# Download from https://docker.com

# 2. Run Hasura + PostgreSQL together
docker-compose up -d
```

**docker-compose.yml:**
```yaml
version: '3.6'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: postgres_password
      POSTGRES_DB: hey_db
    volumes:
      - db_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  graphql-engine:
    image: hasura/graphql-engine:latest
    ports:
      - "8080:8080"
    environment:
      HASURA_GRAPHQL_DATABASE_URL: postgresql://postgres:postgres_password@postgres:5432/hey_db
      HASURA_GRAPHQL_ENABLE_CONSOLE: "true"
      HASURA_GRAPHQL_ADMIN_SECRET: admin_secret_123
      HASURA_GRAPHQL_JWT_SECRET: '{"type": "HS256", "key": "your-secret-key-here"}'
    depends_on:
      - postgres

volumes:
  db_data:
```

**Run it:**
```bash
docker-compose up -d
# Access Hasura Console: http://localhost:8080
# PgAdmin: http://localhost:5050
```

---

## Option B: Production (Cloud)
Pick ONE:

### **Neon (Recommended - PostgreSQL hosting)**
```bash
# 1. Sign up: https://neon.tech
# 2. Create project
# 3. Get connection string:
postgresql://user:password@ep-xxx.neon.tech/hey_db

# 4. Deploy Hasura on Railway/Render
```

### **Railway (Hasura + PostgreSQL together)**
```bash
# 1. Sign up: https://railway.app
# 2. Deploy via Railway dashboard
# 3. Add PostgreSQL service
# 4. Deploy Hasura
# Cost: ~$10/month
```

### **Render (Hasura hosting)**
```bash
# 1. Sign up: https://render.com
# 2. Deploy Hasura
# 3. Connect to Neon PostgreSQL
# Cost: Free tier available
```

---

# PART 2: DATABASE SCHEMA

## Your Entities (From Lens → PostgreSQL)

```sql
-- =====================================================
-- 1. USERS / ACCOUNTS
-- =====================================================

CREATE TABLE accounts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  address VARCHAR(42) UNIQUE NOT NULL,
  username VARCHAR(100) UNIQUE,
  
  -- Profile info
  name VARCHAR(255),
  bio TEXT,
  avatar_url TEXT,
  cover_url TEXT,
  
  -- Verification & Status
  is_verified BOOLEAN DEFAULT FALSE,
  is_blocked BOOLEAN DEFAULT FALSE,
  is_pro BOOLEAN DEFAULT FALSE,
  
  -- Stats
  follower_count INT DEFAULT 0,
  following_count INT DEFAULT 0,
  post_count INT DEFAULT 0,
  
  -- Metadata (JSON for flexibility)
  metadata JSONB DEFAULT '{}',
  
  -- Timestamps
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  CHECK (address ~ '^0x[a-fA-F0-9]{40}$')
);

CREATE INDEX idx_accounts_address ON accounts(address);
CREATE INDEX idx_accounts_username ON accounts(username);


-- =====================================================
-- 2. AUTHENTICATION & SESSIONS
-- =====================================================

CREATE TABLE auth_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  
  -- Session tokens
  access_token TEXT NOT NULL,
  refresh_token TEXT NOT NULL,
  
  -- Session info
  device_name VARCHAR(255),
  ip_address INET,
  user_agent TEXT,
  
  -- Expiry
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  
  -- Revoked
  is_active BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_sessions_account ON auth_sessions(account_id);
CREATE INDEX idx_sessions_active ON auth_sessions(is_active);


-- =====================================================
-- 3. SOCIAL GRAPH (Follows/Blocks)
-- =====================================================

CREATE TABLE follows (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  follower_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  following_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  
  created_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(follower_id, following_id),
  CHECK (follower_id != following_id)
);

CREATE INDEX idx_follows_follower ON follows(follower_id);
CREATE INDEX idx_follows_following ON follows(following_id);


CREATE TABLE blocks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  blocker_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  blocked_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  
  created_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(blocker_id, blocked_id),
  CHECK (blocker_id != blocked_id)
);

CREATE INDEX idx_blocks_blocker ON blocks(blocker_id);


CREATE TABLE mutes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  muter_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  muted_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  
  created_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(muter_id, muted_id)
);


-- =====================================================
-- 4. POSTS / CONTENT
-- =====================================================

CREATE TABLE posts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  author_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  
  -- Content
  content TEXT NOT NULL,
  metadata JSONB DEFAULT '{}',
  
  -- Post type
  post_type VARCHAR(50) DEFAULT 'post', -- 'post', 'quote', 'comment', 'repost'
  
  -- References (for quotes/comments)
  parent_post_id UUID REFERENCES posts(id) ON DELETE CASCADE,
  quoted_post_id UUID REFERENCES posts(id) ON DELETE CASCADE,
  
  -- Stats
  likes_count INT DEFAULT 0,
  comments_count INT DEFAULT 0,
  reposts_count INT DEFAULT 0,
  quotes_count INT DEFAULT 0,
  
  -- Visibility
  is_hidden BOOLEAN DEFAULT FALSE,
  is_deleted BOOLEAN DEFAULT FALSE,
  
  -- Timestamps
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  deleted_at TIMESTAMP
);

CREATE INDEX idx_posts_author ON posts(author_id);
CREATE INDEX idx_posts_parent ON posts(parent_post_id);
CREATE INDEX idx_posts_created ON posts(created_at DESC);
CREATE INDEX idx_posts_type ON posts(post_type);


-- =====================================================
-- 5. ATTACHMENTS / MEDIA
-- =====================================================

CREATE TABLE attachments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  
  -- File info
  file_name VARCHAR(255) NOT NULL,
  file_url TEXT NOT NULL,
  file_size INT,
  file_type VARCHAR(50), -- 'image', 'video', 'audio'
  mime_type VARCHAR(100),
  
  -- Media dimensions
  width INT,
  height INT,
  
  -- Storage
  storage_path TEXT,
  storage_provider VARCHAR(50) DEFAULT 'supabase', -- 'supabase', 's3', 'gcs'
  
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_attachments_post ON attachments(post_id);


-- =====================================================
-- 6. INTERACTIONS (Likes, Reposts, etc)
-- =====================================================

CREATE TABLE post_reactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  
  reaction_type VARCHAR(50) DEFAULT 'like', -- 'like', 'love', 'haha', etc
  
  created_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(post_id, account_id, reaction_type)
);

CREATE INDEX idx_reactions_post ON post_reactions(post_id);
CREATE INDEX idx_reactions_account ON post_reactions(account_id);


CREATE TABLE reposts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  
  repost_type VARCHAR(50) DEFAULT 'repost', -- 'repost', 'quote'
  
  created_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(post_id, account_id)
);

CREATE INDEX idx_reposts_post ON reposts(post_id);
CREATE INDEX idx_reposts_account ON reposts(account_id);


-- =====================================================
-- 7. GROUPS / COMMUNITIES
-- =====================================================

CREATE TABLE groups (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  address VARCHAR(42) UNIQUE NOT NULL,
  creator_id UUID NOT NULL REFERENCES accounts(id),
  
  -- Info
  name VARCHAR(255) NOT NULL,
  description TEXT,
  avatar_url TEXT,
  cover_url TEXT,
  
  -- Stats
  member_count INT DEFAULT 1,
  post_count INT DEFAULT 0,
  
  -- Settings
  rules TEXT,
  metadata JSONB DEFAULT '{}',
  
  -- Status
  is_private BOOLEAN DEFAULT FALSE,
  is_deleted BOOLEAN DEFAULT FALSE,
  
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_groups_creator ON groups(creator_id);
CREATE INDEX idx_groups_address ON groups(address);


CREATE TABLE group_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  group_id UUID NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
  account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  
  -- Role
  role VARCHAR(50) DEFAULT 'member', -- 'owner', 'admin', 'moderator', 'member'
  
  joined_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(group_id, account_id)
);

CREATE INDEX idx_group_members_group ON group_members(group_id);
CREATE INDEX idx_group_members_account ON group_members(account_id);


-- =====================================================
-- 8. NOTIFICATIONS
-- =====================================================

CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  recipient_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  
  -- Action
  action_type VARCHAR(50) NOT NULL, -- 'like', 'follow', 'comment', etc
  actor_id UUID REFERENCES accounts(id) ON DELETE SET NULL,
  
  -- Related entity
  post_id UUID REFERENCES posts(id) ON DELETE CASCADE,
  group_id UUID REFERENCES groups(id) ON DELETE CASCADE,
  
  -- Content
  message TEXT,
  metadata JSONB DEFAULT '{}',
  
  -- Status
  is_read BOOLEAN DEFAULT FALSE,
  
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_notifications_recipient ON notifications(recipient_id);
CREATE INDEX idx_notifications_unread ON notifications(recipient_id, is_read);
CREATE INDEX idx_notifications_created ON notifications(created_at DESC);


-- =====================================================
-- 9. MESSAGES / DMs
-- =====================================================

CREATE TABLE conversations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  participant_1_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  participant_2_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  
  last_message_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(participant_1_id, participant_2_id),
  CHECK (participant_1_id < participant_2_id)
);

CREATE INDEX idx_conversations_p1 ON conversations(participant_1_id);
CREATE INDEX idx_conversations_p2 ON conversations(participant_2_id);


CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
  sender_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  
  content TEXT NOT NULL,
  
  is_read BOOLEAN DEFAULT FALSE,
  
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_messages_conversation ON messages(conversation_id);
CREATE INDEX idx_messages_sender ON messages(sender_id);


-- =====================================================
-- 10. BOOKMARKS / SAVED POSTS
-- =====================================================

CREATE TABLE bookmarks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  
  created_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(account_id, post_id)
);

CREATE INDEX idx_bookmarks_account ON bookmarks(account_id);


-- =====================================================
-- 11. SUBSCRIPTIONS / MONETIZATION
-- =====================================================

CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subscriber_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  creator_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  
  subscription_tier VARCHAR(50) DEFAULT 'pro', -- 'pro', 'vip', etc
  price_per_month DECIMAL(10, 2),
  
  is_active BOOLEAN DEFAULT TRUE,
  
  subscribed_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP,
  
  UNIQUE(subscriber_id, creator_id)
);

CREATE INDEX idx_subscriptions_subscriber ON subscriptions(subscriber_id);
CREATE INDEX idx_subscriptions_creator ON subscriptions(creator_id);


-- =====================================================
-- 12. SUPER FOLLOWS (Additional monetization)
-- =====================================================

CREATE TABLE super_follows (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  follower_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  creator_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  
  price DECIMAL(10, 2),
  
  started_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP,
  
  UNIQUE(follower_id, creator_id)
);

CREATE INDEX idx_super_follows_follower ON super_follows(follower_id);
CREATE INDEX idx_super_follows_creator ON super_follows(creator_id);


-- =====================================================
-- 13. COLLECTIONS / FEEDS
-- =====================================================

CREATE TABLE feeds (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  
  name VARCHAR(255) NOT NULL,
  description TEXT,
  
  feed_type VARCHAR(50) DEFAULT 'custom', -- 'timeline', 'curated', 'custom'
  
  is_private BOOLEAN DEFAULT FALSE,
  
  created_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(account_id, name)
);

CREATE TABLE feed_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  feed_id UUID NOT NULL REFERENCES feeds(id) ON DELETE CASCADE,
  post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  
  added_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(feed_id, post_id)
);


-- =====================================================
-- 14. SEARCH INDEX (for full-text search)
-- =====================================================

CREATE TABLE search_index (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  entity_type VARCHAR(50) NOT NULL, -- 'post', 'account', 'group'
  entity_id UUID NOT NULL,
  
  search_text TEXT NOT NULL,
  
  created_at TIMESTAMP DEFAULT NOW()
);

-- Full-text search index
CREATE INDEX idx_search_text ON search_index USING GIN(
  to_tsvector('english', search_text)
);

-- =====================================================
-- 15. ANALYTICS / ACTIVITY LOG
-- =====================================================

CREATE TABLE activity_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id UUID REFERENCES accounts(id) ON DELETE SET NULL,
  
  action VARCHAR(100) NOT NULL,
  resource_type VARCHAR(50),
  resource_id UUID,
  
  ip_address INET,
  user_agent TEXT,
  
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_activity_account ON activity_log(account_id);
CREATE INDEX idx_activity_created ON activity_log(created_at DESC);
```

---

# PART 3: ENABLE HASURA FEATURES

After creating the schema, run these in Hasura Console:

## 1. **Track Tables**
```
Hasura Console → Data → Untracked Tables
→ Click "Track All"
```

## 2. **Set Up Relationships**
```
Data → accounts → Relationships
→ Auto-detect relationships
→ Add manual relationships for complex ones
```

## 3. **Configure Permissions**
```
Data → accounts → Permissions
→ Select role
→ Define access control
```

Example (Users can only see their own data + public profiles):
```
Role: user
Filter: { id: { _eq: "X-Hasura-User-Id" } }
```

## 4. **Enable Real-time**
```
Data → posts → Properties
→ Toggle "Streaming subscriptions"
```

---

# PART 4: YOUR NEW ENVIRONMENT FILE

Update your `.env`:

```bash
# Hasura
VITE_HASURA_URL=http://localhost:8080/v1/graphql  # Or cloud URL
VITE_HASURA_ADMIN_SECRET=admin_secret_123

# Authentication
VITE_JWT_SECRET=your-secret-key-here
VITE_SESSION_EXPIRY=3600000  # 1 hour in ms

# File Storage (Supabase)
VITE_SUPABASE_URL=https://xxxx.supabase.co
VITE_SUPABASE_ANON_KEY=your-key

# Web3 (optional, keep if needed)
VITE_WALLETCONNECT_PROJECT_ID=cd542acc70c2b548030f9901a52e70c8
```

---

# PART 5: YOUR NEW APOLLO CLIENT

**src/lib/hasura/client.ts:**

```typescript
import { ApolloClient, InMemoryCache, HttpLink, from } from "@apollo/client";
import { onError } from "@apollo/client/link/error";
import { authLink } from "./authLink";

const errorLink = onError(({ graphQLErrors, networkError }) => {
  if (graphQLErrors) {
    graphQLErrors.forEach(({ message, locations, path }) =>
      console.log(`[GraphQL error]: ${message}`)
    );
  }
  if (networkError) {
    console.log(`[Network error]: ${networkError}`);
  }
});

const httpLink = new HttpLink({
  uri: import.meta.env.VITE_HASURA_URL,
  credentials: "include",
  headers: {
    "X-Hasura-Admin-Secret": import.meta.env.VITE_HASURA_ADMIN_SECRET,
  }
});

export const hasuraClient = new ApolloClient({
  link: from([authLink, errorLink, httpLink]),
  cache: new InMemoryCache()
});
```

**src/lib/hasura/authLink.ts:**

```typescript
import { ApolloLink, fromPromise } from "@apollo/client";
import { getSession } from "./auth";

export const authLink = new ApolloLink((operation, forward) => {
  const session = getSession();

  if (session?.access_token) {
    operation.setContext({
      headers: {
        Authorization: `Bearer ${session.access_token}`,
        "X-Hasura-User-Id": session.account_id,
      },
    });
  }

  return forward(operation);
});
```

---

# PART 6: MAPPING YOUR QUERIES

## Before (Lens GraphQL):
```graphql
query GetPosts($request: PostsRequest!) {
  mlPostsForYou(request: $request) {
    items {
      id
      author { address username }
      content
      likes_count
    }
  }
}
```

## After (Hasura GraphQL):
```graphql
query GetPostsForYou($limit: Int!, $offset: Int!) {
  posts(
    limit: $limit
    offset: $offset
    order_by: { created_at: desc }
    where: {
      _and: [
        { author: { followers: { follower_id: { _eq: $current_user_id } } } }
        { is_deleted: { _eq: false } }
      ]
    }
  ) {
    id
    content
    likes_count
    author {
      id
      address
      username
      avatar_url
    }
    post_reactions {
      reaction_type
    }
  }
}
```

Your React component needs **minimal changes**:

```typescript
// Before
import { usePostsForYouQuery } from "@/indexer/generated"
const { data } = usePostsForYouQuery(/* ... */)

// After
import { useGetPostsForYouQuery } from "@/lib/hasura/generated"
const { data } = useGetPostsForYouQuery(/* ... */)
```

**The component code stays THE SAME! ✅**

---

# PART 7: MIGRATION CHECKLIST

## Week 1: Infrastructure
- [ ] Set up PostgreSQL (local or cloud)
- [ ] Deploy Hasura
- [ ] Create admin account
- [ ] Verify Hasura Console access

## Week 2: Database Schema
- [ ] Run SQL schema creation
- [ ] Verify all tables created
- [ ] Track tables in Hasura
- [ ] Set up relationships
- [ ] Enable real-time subscriptions

## Week 3: Data Migration
- [ ] Export data from Lens Protocol
- [ ] Transform data to match schema
- [ ] Bulk insert into PostgreSQL
- [ ] Verify data integrity

## Week 4: Backend Integration
- [ ] Update Apollo Client config
- [ ] Generate new types with graphql-codegen
- [ ] Create auth helpers
- [ ] Test queries work

## Week 5: Frontend Updates
- [ ] Update imports in components
- [ ] Test data fetching
- [ ] Fix any type mismatches
- [ ] Deploy to production

---

# PART 8: YOUR FINAL FOLDER STRUCTURE

```
src/
├── lib/
│   └── hasura/              ← New!
│       ├── client.ts
│       ├── authLink.ts
│       ├── auth.ts
│       ├── generated.ts     ← Auto-generated from Hasura
│       ├── queries/
│       │   ├── posts.graphql
│       │   ├── accounts.graphql
│       │   └── groups.graphql
│       └── mutations/
│           ├── createPost.graphql
│           └── updateAccount.graphql
│
├── components/              ← 99% unchanged!
│   ├── Home/
│   ├── Post/
│   └── etc...
│
└── store/                   ← Minor auth store update
```

---

# PART 9: QUICK COMMANDS

```bash
# Local development
docker-compose up -d

# Access Hasura Console
http://localhost:8080

# Access PostgreSQL
psql postgresql://postgres:postgres_password@localhost:5432/hey_db

# Generate types from Hasura schema
npx graphql-codegen

# Deploy to Railway
railway link  # Connect to Railway project
railway up    # Deploy

# Check migration status
SELECT table_name FROM information_schema.tables 
WHERE table_schema = 'public';
```

---

# PART 10: COST BREAKDOWN

| Component | Option | Cost |
|-----------|--------|------|
| **PostgreSQL** | Neon | Free-$70/mo |
| **PostgreSQL** | Railway | $5-30/mo |
| **Hasura** | Self-hosted | Free (Docker) |
| **Hasura** | Cloud | $99/mo (Pro) |
| **Total (cheap)** | Neon + Self-hosted | ~Free ($5-10 for compute) |
| **Total (managed)** | Railway + Hasura Cloud | ~$100-150/mo |

**Estimate:** $10-30/month for production ✅

---

# SUMMARY

| Aspect | Details |
|--------|---------|
| **Setup Time** | 2-3 hours (database) + 1 week (integration) |
| **Database Tables** | 15 main tables |
| **Database Rows Limit** | PostgreSQL: Unlimited (millions possible) |
| **Component Changes** | ~5% of code |
| **Backend Changes** | ~60% (queries/mutations) |
| **Real-time Ready** | ✅ Yes (Hasura subscriptions) |
| **Type Safety** | ✅ Full (auto-generated types) |
| **Scalability** | ✅ Enterprise-ready |

---

## 🚀 Next Steps

1. Choose hosting: **Neon (DB) + Railway (Hasura)** = Simple & cheap
2. Run docker-compose locally to test
3. Create the schema
4. Track tables in Hasura
5. Generate types
6. Update Apollo config
7. Test queries
8. Deploy! 🎉

Want me to show you the **data migration queries** next? 📊
