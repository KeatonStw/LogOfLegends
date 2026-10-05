# Log Of Legends
A mobile companion app built with React Native Expo and Supabase that logs League of Legends match histories, tracks solo queue climbs, and introduces social leaderboards and LP races with friends.
## 1. Problem Definition & Mobile Scope

### The Problem
While web-based League of Legends statistics sites (like OP.GG or U.GG) are abundant, they are optimized for desktop browsers and transactional, single-session lookups. They lack persistent, context-aware social spaces where friend groups can track collective climbs, compare historical trends over time, or run friendly competitive challenges (like LP races) directly from their mobile devices.

### Semester Scope
To ensure a manageable and successful build within the semester timeline, the project scope focuses strictly on core utility and foundational relational tracking:
* **In-Scope:** User authentication (via Supabase Auth), North American (for now) Riot account linking, secure data fetching (via Supabase Edge Functions), historical LP logging, and private friend-group creation with custom leaderboards.
* **Out-of-Scope (Deferred):** More regions, Real-time in-game websocket notifications, automated background cron jobs for live match tracking, and complex fantasy drafting mini-games.
* **Target Platform:** Cross-platform mobile application deployed via **React Native Expo** targeting iOS and Android.

---

## 2. Initial Database Design (HIGHLY VOLATILE UNTIL ACTUAL DEVELOPMENT HAS STARTED)

The database is built on **PostgreSQL (via Supabase)** to maintain strict referential integrity across users, accounts, and groups.

### Entity-Relationship Outline
* **`profiles`** $\rightarrow$ Extends Supabase auth; parent to `lol_accounts` and `groups`.
* **`lol_accounts`** $\rightarrow$ Child of `profiles`; parent to `lp_history`.
* **`groups`** $\rightarrow$ Created by a profile; links users via the `group_members` join table.
* **`group_members`** $\rightarrow$ Many-to-many junction table connecting `profiles` and `groups`.
* **`lp_history`** $\rightarrow$ Time-series child table of `lol_accounts`.

### SQL Schema DDL

```sql
-- 1. PROFILES TABLE
CREATE TABLE public.profiles (
  id UUID REFERENCES auth.users ON DELETE CASCADE PRIMARY KEY,
  username TEXT UNIQUE NOT NULL,
  avatar_url TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT TIMEZONE('utc'::text, NOW()) NOT NULL
);

-- 2. LOL_ACCOUNTS TABLE (Primary Key: id, Foreign Key: user_id)
CREATE TABLE public.lol_accounts (
  id UUID DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
  user_id UUID REFERENCES public.profiles(id) ON DELETE CASCADE NOT NULL,
  game_name TEXT NOT NULL,       
  tag_line TEXT NOT NULL,        
  puuid TEXT UNIQUE NOT NULL,    
  region TEXT NOT NULL,          
  tier TEXT,                     
  rank TEXT,                     
  league_points INT,             
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT TIMEZONE('utc'::text, NOW()) NOT NULL
);

-- 3. GROUPS TABLE (Primary Key: id, Foreign Key: created_by)
CREATE TABLE public.groups (
  id UUID DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
  name TEXT NOT NULL,
  invite_code TEXT UNIQUE NOT NULL,
  created_by UUID REFERENCES public.profiles(id) ON DELETE CASCADE NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT TIMEZONE('utc'::text, NOW()) NOT NULL
);

-- 4. GROUP_MEMBERS TABLE (Primary Key: id, Foreign Keys: group_id, user_id)
CREATE TABLE public.group_members (
  id UUID DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
  group_id UUID REFERENCES public.groups(id) ON DELETE CASCADE NOT NULL,
  user_id UUID REFERENCES public.profiles(id) ON DELETE CASCADE NOT NULL,
  joined_at TIMESTAMP WITH TIME ZONE DEFAULT TIMEZONE('utc'::text, NOW()) NOT NULL,
  UNIQUE(group_id, user_id)
);

-- 5. LP_HISTORY TABLE (Primary Key: id, Foreign Key: lol_account_id)
CREATE TABLE public.lp_history (
  id UUID DEFAULT GEN_RANDOM_UUID() PRIMARY KEY,
  lol_account_id UUID REFERENCES public.lol_accounts(id) ON DELETE CASCADE NOT NULL,
  tier TEXT NOT NULL,
  rank TEXT NOT NULL,
  league_points INT NOT NULL,
  recorded_at TIMESTAMP WITH TIME ZONE DEFAULT TIMEZONE('utc'::text, NOW()) NOT NULL
);
```

## 3. Relational Algebra and queries
Example queries which will need to be made for app functionality:

- Find all linked League accounts for a specific user:

  **π_game_name, tag_line, tier, league_points ( σ_user_id = 'user_uuid_123' (lol_accounts) )**

- Retrieve all members belonging to a specific friend group:

  **π_username, avatar_url ( profiles ⨝_profiles.id = group_members.user_id ( σ_group_id = 'group_uuid_456' (group_members) ) )**

- Get historical LP records for a specific League account:

  **π_tier, rank, league_points, recorded_at ( σ_lol_account_id = 'account_uuid_789' (lp_history) )**

- Find all groups created by a specific user profile:

  **π_name, invite_code, created_at ( σ_created_by = 'user_uuid_123' (groups) )**

- Cross-reference group members with their corresponding active League accounts (Join of 3 relations):

  **π_username, game_name, tier, league_points ( (group_members ⨝_group_members.user_id = profiles.id profiles) ⨝_profiles.id = lol_accounts.user_id lol_accounts )**


## 4. AI Utilization Plan

Conversational Tutors (Gemini / ChatGPT): Used exclusively for conceptual explanations, database architecture feedback, understanding external API documentation (Riot Games API), and troubleshooting SQL syntax errors.

Coding Assistant (GitHub Copilot - Student Account): Used in the Visual Studio Code to accelerate boilerplate code generation, standard React Native component styling, and repetitive UI layout implementation once core logic has been manually designed.

AI Log - There will be an AI Log included in the project which will include every prompt and response/added code to ensure that all AI usage follows the plan outlined here.

Example Prompts:
- "I am designing a Supabase Edge Function to fetch a summoner's PUUID from the Riot Games API using their game name and tagline. Can you explain the best practices for handling Riot's rate limits and caching responses in PostgreSQL?"

- "I am encountering a foreign key constraint violation when inserting into my group_members table. Here is my table definition schema and my insert payload: [Snippet]. Can you explain why PostgreSQL is rejecting this reference and what conceptual rule I am breaking?"

- "I have already set up my Supabase client and created a custom hook useGroupLeaderboard(groupId) that returns an array of objects with { username, game_name, tier, league_points }. Using React Native (Expo) and TypeScript, can you help me write a clean, performant LeaderboardScreen component using a FlatList that renders each user as a styled card using standard StyleSheet?"
