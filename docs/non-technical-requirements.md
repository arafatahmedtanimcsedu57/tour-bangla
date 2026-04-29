# Non-Technical Requirements — Tour Bangladesh

## 1. Product Vision

Tour Bangladesh is a community-driven travel experience platform where people share their journeys across Bangladesh. Anyone can browse experiences for inspiration. Registered users can post their own trips, follow fellow travelers, and engage through likes and comments.

---

## 2. Target Audience

| Segment | Description |
|---------|-------------|
| **Travelers** | People who have visited places in Bangladesh and want to share their experience |
| **Explorers** | People planning a trip who want to discover destinations through real stories |
| **Locals** | Bangladeshi residents who want to explore their own country |
| **General visitors** | Anyone browsing travel content without intent to post |

**Primary device**: Mobile (smartphone), primarily on Android

---

## 3. User Roles

| Role | Access Level |
|------|-------------|
| **Guest** | Browse all published experiences, view profiles, use search and filters |
| **Registered User** | All guest access + post experiences, like, comment, follow others |
| **Author** | Edit and delete their own experiences and comments |
| **Admin** *(future)* | Moderate content, manage reported posts, manage users |

---

## 4. Core Features

### 4.1 Browse & Discovery
- View all published travel experiences on the homepage
- Search experiences by keyword (title, location, description)
- Filter by:
  - Division (e.g., Dhaka, Chittagong, Sylhet)
  - District
  - Category (e.g., nature, heritage, adventure, food, beach)
  - Star rating
  - Estimated cost range (BDT)
  - Travel date range
- Explore experiences visually on an interactive map
- Browse experiences by tag

### 4.2 Experience Posts
Each experience post contains:
- Title
- Description (rich text)
- Cover photo + up to 10 photos
- Location (division + district + optional map pin)
- Travel route (optional series of map points)
- Star rating (1–5)
- Estimated cost in BDT (with optional breakdown)
- Travel dates (start and end)
- Travel tips
- Tags (e.g., #sundarban, #beach, #heritage)
- Category

### 4.3 User Accounts
- Register with email and password or Google account
- Edit profile: display name, username, avatar, bio
- View own posted experiences
- View follower and following counts

### 4.4 Social Interactions
- **Like** an experience (toggle)
- **Comment** on an experience
- **Follow** another user
- **Activity feed**: see latest experiences from people you follow

### 4.5 Map Features
- View location pin on a map for each experience
- View travel route as a line on the map
- Explore page with all experiences as pins on a map
- Click a map pin to preview the experience

---

## 5. Content Rules

- All experience content must relate to Bangladesh
- Photos must be real travel photos (no stock images encouraged)
- No hate speech, adult content, or spam
- Users may report inappropriate content *(moderation in future phase)*
- Authors can edit or delete their own posts at any time

---

## 6. User Experience Principles

- **Mobile-first**: The app must be fully usable on a 375px wide screen with one hand
- **Fast loading**: Pages should load within 3 seconds on a typical mobile network
- **Intuitive navigation**: Bottom navigation bar on mobile for key sections
- **Offline-friendly**: Browsed content should remain readable if connection drops *(Phase 3)*
- **Accessible**: Text contrast, tap target sizes, and screen reader support

---

## 7. Language & Localisation

- Primary language: **English**
- Location names displayed in both **English and Bangla** (বাংলা)
- Currency: **BDT (৳)** for all cost fields
- Date format: **DD MMM YYYY** (e.g., 15 Mar 2024)

---

## 8. Performance Expectations

| Metric | Target |
|--------|--------|
| Homepage load time | < 3 seconds on 3G |
| API response time | < 300ms for standard queries |
| Image display | Lazy-loaded, progressive |
| Search results | Appear within 500ms of query |

---

## 9. Privacy & Data

- User email addresses are never shown publicly
- Users can delete their account and all associated content
- No tracking or analytics beyond basic usage metrics
- Passwords are never stored directly (handled by Keycloak)

---

## 10. Future Considerations *(out of scope for MVP)*

- Video uploads
- Experience collections / itineraries
- Bookmark / save for later
- Push notifications
- Admin moderation dashboard
- Author analytics (views, likes over time)
- In-app messaging between users
