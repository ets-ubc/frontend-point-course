## PointQuest (WL Student Guide)

An interactive campus map where students can explore curated points and submit their own. Each course can have its own map with custom style and viewport. Submissions can require admin approval and appear in real time once approved.

### Quick Start

1) Install Node 18+ and MongoDB.
2) Install dependencies:
```
npm install
```
3) Backend env: create `server/.env` with at least:
```
MONGODB_URI=mongodb://localhost:27017/pointquest
ADMIN_SECRET=choose-a-strong-password
CORS_ORIGIN=http://localhost:3000
```
4) Start everything (frontend + backend):
```
npm run dev
```
Or run separately:
```
npm start      # frontend on http://localhost:3000
npm run server # backend on http://localhost:5501
```

### What you’ll see

- Home (`/`): enter a course code or continue as guest.
- Guest map (`/guest`): default dataset + approved public markers.
- Course map (`/course/:courseCode`): course-specific markers and styling.
- Admin login (`/admin/login`) and dashboard (`/admin`): manage courses and approve submissions.

---

## How the app is organized

Frontend: React (Create React App)
- `src/index.js`: boots React, loads `App` and global CSS.
- `src/App.js`: defines routes using `react-router-dom`:
  - `/` → `LoginPage`
  - `/guest` → `MapComponent` (no course, default data)
  - `/course/:courseCode` → `CourseMapPage`
  - `/admin/login` → `AdminLoginPage`
  - `/admin` → `AdminPage`

Backend: Node.js + Express + Socket.IO + MongoDB
- `server/index.js`: API endpoints, admin auth via header `x-admin-secret`, image uploads, Mongo connection, and realtime events.
- `server/models/` contains Mongoose schemas for courses and markers.

Configuration
- `src/config.js`: `API_BASE_URL` → set via `REACT_APP_API_URL` or default `http://localhost:5501`.
- Mapbox token: `AdminPage` reads `REACT_APP_MAPBOX_TOKEN` (falls back to a default). `MapComponent` currently uses an in-file token to keep the demo working.

Assets
- Static images and 3D model live under `public/Media/` and `public/models/`.

---

## How pages and components work together

High level flow
- `LoginPage` lets you join as a guest or open a course by code.
- `CourseMapPage` fetches the course config from `GET /api/courses/by-code/:courseCode`, then renders `MapComponent` with `courseCode` and styling.
- `MapComponent` displays markers on a Mapbox map, allows filtering, and supports adding a new point (opens a modal for details and image upload).
- If a course requires approval, newly submitted markers appear in the admin queue; once approved, they appear instantly for everyone via Socket.IO.

Key components (`src/components/`)
- `MapComponent.js`
  - Renders the map with `react-map-gl` and Mapbox style.
  - Loads markers:
    - Guest mode: pulls approved markers from a public API, plus some built-in demo markers.
    - Course mode: calls your backend `GET /api/markers?courseCode=...&onlyApproved=true`.
  - Real-time updates: joins `Socket.IO` room `course:<courseCode>` and listens for `markerApproved` / `markerRejected` events from the server.
  - Adding points: opens `AddPointModalComponent`; on submit, it posts form data (including `lat`, `lng`, `name`, `description`, optional `image`, `category[]`, and `courseCode` when present) to either your backend (`/api/markers`) or the public service in guest mode.
  - Reverse geocoding: uses Google Maps Geocoding to display human-readable addresses on markers.
- `MarkerComponent.js`: places a Mapbox marker and opens `ModalComponent` with details when selected.
- `ModalComponent.js`: shows images, text, and links for a selected marker.
- `DropdownFilter.js`: filters markers by category.
- `AddPointButtonComponent.js` + `AddPointModalComponent.js`: toggle and collect data for new marker submissions.
- `InfoButtonComponent.js`, `FeedbackButtonComponent.js`, `NotificationModalComponent.js`, `OpeningModalComponent.js`, `LogoComponent.js`: UI/UX elements.

Pages (`src/pages/`)
- `LoginPage.js`: entry experience to pick guest vs course.
- `CourseMapPage.js`: fetches course settings then renders `MapComponent` with those settings.
- `AdminLoginPage.js`: prompts for the admin password; on success stores it with `setAdminSecret`.
- `AdminPage.js`: manage courses and markers.
  - Courses: create with map style, center, zoom, pitch, bearing, and approval requirement.
  - Pending markers: list all unapproved submissions, approve or reject.
  - Live markers: browse and delete approved markers for a specific course.

---

## Backend API and realtime

Auth
- Admin endpoints require header `x-admin-secret: <ADMIN_SECRET>`.

Courses
- `POST /api/courses` (admin): create a course with fields like `courseCode`, `title`, `requireApproval`, `mapStyle`, `centerLng`, `centerLat`, `startZoom`, `minZoom`, `maxZoom`, `pitch`, `bearing`.
- `GET /api/courses` (admin): list courses.
- `GET /api/courses/by-code/:courseCode`: fetch a single course for the public/course map.
- `DELETE /api/courses/:courseCode` (admin): delete a course and its markers.

Markers
- `GET /api/markers?courseCode=ABC&onlyApproved=true`: list approved markers for a course.
- `POST /api/markers` (multipart): create a marker. Fields include `lat`, `lng`, `name`, `description`, `category[]`, `markerId` (client UUID), `address`, optional `image`, `takeAction`, and required `courseCode` for course maps.
- `GET /api/markers/pending` (admin): list unapproved markers, optionally filtered by `courseCode`.
- `PATCH /api/markers/:id/approve` (admin): approve a marker and broadcast realtime event.
- `DELETE /api/markers/:id` (admin): delete a marker and broadcast rejection.

Realtime (Socket.IO)
- Clients emit `joinCourse` with the course code to join `course:<code>`.
- Server emits `markerApproved` with marker data to that room when a submission is approved; `MapComponent` appends it immediately.
- On deletion, server emits `markerRejected` with the marker id; `MapComponent` removes it.

Uploads
- Images are saved to `server/uploads/` and served from `/uploads/<filename>`.

---

## Common tasks

- Create a course (admin): open `/admin`, fill the form, click Create. Then share the course URL `/course/COURSECODE`.
- Submit a point (student): on a map, click "+" to add, click location on map, fill details, upload an image, submit.
- Approve points (instructor/TA): open `/admin`, go to Pending markers, Approve. Approved points appear live for students.

---

## Configuration notes

- API URL: to target a non-local server, set `REACT_APP_API_URL` before starting the frontend.
- Mapbox: you can set `REACT_APP_MAPBOX_TOKEN` to use your own token in `AdminPage`. `MapComponent` currently embeds a demo token; replace as needed.
- CORS: adjust `CORS_ORIGIN` in `server/.env` for your deployment domain(s).

---

## Scripts

- `npm start`: start React dev server on `http://localhost:3000`.
- `npm run server`: start backend on `http://localhost:5501`.
- `npm run dev`: run both concurrently.
- `npm test`: run tests (CRA).
- `npm run build`: production build to `build/`.

---

## Troubleshooting

- Backend 401 on admin calls: ensure `ADMIN_SECRET` is set and you’re sending header `x-admin-secret` (the UI sets this after admin login).
- No markers on a course map: check that the course exists and has approved markers; verify `REACT_APP_API_URL` points to the backend.
- Map not loading: verify internet access to Mapbox and token validity; check console for Mapbox errors.

---

## Step-by-step: Modify the built‑in points on the base map

These steps edit the default points shown in guest mode (the base map). Course maps do not use these built‑in points; they load markers from the backend.

1) Open the component that holds the default points
- File: `src/components/MapComponent.js`
- Look for the array named `initialMarkers` near the top of the file.

2) Understand the shape of a marker object
```js
{
  markerId: 1,                   // number or string id
  name: 'Location name',         // title shown in the UI
  image: 'Media/IMAGE.png',      // shown in lists and modal content
  description: 'Short text...',   // description in the modal
  lngLat: [-123.25, 49.26],      // [longitude, latitude]
  category: ['history','art'],    // used by the category filter
  content: [                      // what the modal displays (images/links/embeds/text)
    { type: 'image', src: 'Media/IMAGE.png' },
    { type: 'click-link', src: 'https://example.com' },
    { type: 'embed', src: 'https://...' },
    { type: 'text-content', text: '<p>HTML or text here</p>' }
  ],
  takeAction: 'Optional call-to-action shown in modal'
}
```

3) Add a new point
- Append a new object to the `initialMarkers` array.
- Use a unique `markerId` (a number or string).
- Set `lngLat` as `[longitude, latitude]` (note the order: lon, then lat).
- Put any images you reference into `public/Media/` and use paths like `'Media/your-image.png'`.

4) Edit an existing point
- Update fields like `name`, `description`, `category`, `content`, or `lngLat` directly in its object.
- Keep `content` simple: images and links are most common.

5) Remove a point
- Delete its object from the `initialMarkers` array.

6) Save and refresh
- Save the file and your dev server will hot‑reload. If the map doesn’t update, refresh the browser.

Notes and tips
- Required fields: `markerId`, `name`, `lngLat`.
- Coordinates must be valid numbers; invalid values will skip rendering.
- Categories drive the filter in the top‑left; use lower‑case words for consistency.
- Guest mode also fetches additional approved markers from a public API; your built‑in points will still appear alongside those.
