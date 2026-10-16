# CPEN 221A - Milestone 1: Design Specification

## 1. Design Your Team

**Team Name:** EEL (Eli Eli Le)

**Team Roles & Responsibilities:**
* **Project Manager (PM):** Eli Hanania
* **Designer:** Emi Harker
* **Developer:** Houze (Le) Guo

**Mission Statement:**
Our mission is to advance academic equity for commuter students by transforming underutilized campus infrastructure into accessible, collaborative learning environments. We champion student wellbeing by eliminating the daily friction of finding a workspace, ensuring every student has a place to belong and focus on campus.

---

## 2. Design Your Solution

### Problem
Over 60% of UBC Vancouver’s 60,000+ students commute to campus ([AMS Academic Experience Survey](https://www.ams.ubc.ca/)), frequently navigating schedules with awkward 50-minute to 3-hour gaps between lectures. During peak hours (10:00 AM – 4:00 PM), dedicated study hubs such as the Irving K. Barber Learning Centre, Koerner Library, and the AMS Nest routinely reach full capacity, while bookable group rooms are reserved weeks in advance. Paradoxically, this space shortage is an information and coordination problem rather than a physical lack of rooms. Although [UBC’s Official Classroom Usage Guidelines](https://learningspaces.ubc.ca/useful-resources/guidelines/) explicitly permit students to use general teaching classrooms as study spaces whenever classes or events are not in session, dozens of unlocked classrooms and lecture halls sit completely empty at any given hour. This mismatch stems from a rigid separation between formal academic scheduling and informal student space usage, leaving thousands of square feet of functional study infrastructure hidden in plain sight.

The inability to secure a reliable workspace directly undermines student productivity, well-being, and equitable access to education. Commuter students who cannot practically return home are left wandering buildings in search of an open seat and a power outlet, or settling for noisy hallway floors and stairwells. This daily scavenger hunt wastes hours of instructional recovery time every week, increases physical fatigue, and disproportionately penalizes students who lack the privilege of private residence rooms or exclusive departmental lounge access. When students attempt to collaborate on team projects, the problem compounds; coordinating overlapping free windows alongside a mutually accessible, unoccupied room requires tedious back-and-forth guesswork.

Current student workarounds are highly inefficient and unreliable for day-to-day use. Some students walk down hallways pulling on doors at random—frequently interrupting active classes or settling into a room only to be displaced ten minutes later by an incoming lecture. Others manually cross-reference official university portals ([UBC Online Timetable](https://sws-van.as.it.ubc.ca/) and [UBC Learning Spaces](https://learningspaces.ubc.ca/find-a-space/)), which only allow querying one building and one room at a time. These static, room-by-room interfaces neither account for walking travel times between a student's classes nor reflect real-time conditions on the ground, such as locked doors, early building lock-up hours (e.g., MacLeod and Gateway buildings), or unscheduled club meetings. Because these improvised methods require high effort for uncertain results, students remain bottlenecked in overcrowded libraries while perfectly viable classrooms down the hall sit unused.

---

### Solution

We envision **EEL SpaceFinder**, a mobile-responsive client-server web application that matches student gap windows and physical building locations to unoccupied UBC classrooms and informal learning spaces. The application contains no sound and requires no keyboard typing; all inputs use tap-based dropdowns, toggle chips, and buttons, using the same responsive layout across mobile and desktop platforms.

The system is powered by a server-side dataset loaded from local JSON/CSV files containing UBC building coordinates, building open/lock-up hours, room capacities, room amenities (power outlets, whiteboards, accessibility), and weekly scheduled lecture intervals, combined with live in-memory occupancy and status reports submitted by connected clients.

---

#### Screen 1: Solo Gap Search & Ranked Results Screen (Default View)

When a user opens the application, they land on the **Solo Gap Search Screen**. At the top of the screen is a two-tab mode switcher: **`[ Solo Gap Finder ]`** (selected by default) and **`[ Group Meet-in-the-Middle ]`**.

<!-- TODO: Embed Screen 1 Sketch Here: ![Screen 1 Mockup](./mockups/screen1-solo-search.png) -->

**Interactive Controls (Input Panel):**
1. **Current Building Dropdown (`From`):** A tap-to-open dropdown menu listing UBC academic buildings alphabetically by code and name (e.g., `BUCH - Buchanan`, `ICCS - ICICS`, `KAIS - Fred Kaiser`, `MCLD - MacLeod`). Defaults to `KAIS - Fred Kaiser`.
2. **Next Class Building Dropdown (`To`):** A dropdown menu with the same building list. Defaults to `BUCH - Buchanan`.
3. **Day of Week Selector:** Five tap-able pill buttons (`Mon`, `Tue`, `Wed`, `Thu`, `Fri`). Exactly one day is active at a time.
4. **Gap Start Time & Gap End Time Dropdowns:** Two dropdown selectors in 30-minute increments from `08:00` to `22:30` (respecting UBC's 11:00 PM classroom vacate policy).
5. **Amenity Filter Chips (Optional Toggles):** Three multi-select toggle chips:
   * `⚡ Power Outlets`
   * `🧑‍🦽 Step-Free Accessible`
   * `🖍️ Whiteboard`
6. **`[ Find Available Rooms ]` Button:** Primary action button.

**Validation & Error States on Screen 1:**
* **Invalid Time Range Error:** If the user selects a `Gap End Time` that is earlier than or equal to `Gap Start Time`, or a total gap duration of less than **30 minutes**, the `[ Find Available Rooms ]` button is disabled (turns gray) and an inline red error banner displays: *"Gap window must be at least 30 minutes long (End Time must be at least 30 minutes after Start Time)."*
* **No Matching Rooms State:** If no rooms satisfy all selected constraints for the full time window, the results area displays an empty-state card: *"No guaranteed-free rooms found for this exact window. Try unchecking an amenity filter or shortening your gap window."*

**Algorithmic Behavior (Detour-Bounded Spatio-Temporal Search & Anti-Crowding Load Balancing):**
When the user taps `[ Find Available Rooms ]`, the client sends the query to the server, which executes the following algorithm:
1. **Travel-Time & Transition Buffer Pruning:** Using a campus walkway graph (`CampusGraph`), the server computes the walking time $t_{\text{walk}}(\text{From}, B)$ from the user's current building to candidate building $B$, and $t_{\text{walk}}(B, \text{To})$ from $B$ to the next class building. In accordance with UBC Classroom Usage Guidelines, a **10-minute pre-class transition buffer** is added before the next scheduled lecture. A room $R$ in building $B$ is eligible if and only if:
   * Building $B$ is unlocked during the entire window $[t_{\text{start}}, t_{\text{end}}]$,
   * Room $R$ is not flagged as `Locked / Unavailable` by live user reports,
   * Room $R$ matches all selected amenity chips, and
   * Room $R$'s schedule `IntervalTree` has **zero overlapping lectures** during $[t_{\text{start}} + t_{\text{walk}}(\text{From}, B),\; t_{\text{end}} - t_{\text{walk}}(B, \text{To}) + 10\text{ min}]$, leaving at least 20 minutes of usable study time.
2. **Congestion-Aware Ranking (Mitigating the "Waze Effect"):** Eligible rooms are scored and returned in descending order of utility:
   $$\text{Score}(R) = w_1 \cdot (\text{Extra Free Buffer After } t_{\text{end}}) - w_2 \cdot \text{DetourMinutes}(\text{From}, B, \text{To}) - w_3 \cdot \left(\frac{\text{LiveCheckedInUsers}(R)}{\text{Capacity}(R)}\right)^2$$
   This congestion penalty automatically load-balances concurrent users across nearby empty classrooms so the application never routes dozens of students into the same small seminar room.

**Results List Display:**
Below the input panel, up to **10 top-ranked room cards** are displayed. Each card shows:
* **Room Code & Building Name** (e.g., `MCLD 202 — MacLeod Building`)
* **Availability Badge:** Green badge showing exact continuous free window (e.g., `Free 11:00 – 14:00 (3h 00m)`) and building lock time (`Locks at 21:00`).
* **Walking Detour Summary:** `🚶 2 min from KAIS • 6 min to BUCH (+1 min detour)`.
* **Live Occupancy & Amenities:** `👥 4 / 40 seats checked in (Quiet)` and icons for `⚡`, `🧑‍🦽`, `🖍️`.
* Tapping any room card opens **Screen 2 (Room Detail & Live Status Modal)**.

---

#### Screen 2: Room Detail & Live Check-In Modal

When a user taps a room card from either Solo or Group results, a modal overlay opens showing the room's full day schedule and live crowd status.

<!-- TODO: Embed Screen 2 Sketch Here: ![Screen 2 Mockup](./mockups/screen2-room-modal.png) -->

**Visual Content & Interactive Controls:**
1. **Header:** Room title (`MCLD 202`), capacity (`Capacity: 40`), and amenity badges.
2. **Daily Timeline Bar:** A horizontal timeline from `08:00` to `22:00` showing scheduled lectures in gray blocks, the user's requested study window in blue, and remaining free buffer time in green.
3. **UBC Classroom Etiquette Reminder Box:** Static notice summarizing UBC policy: *"Yield immediately to scheduled bookings, return furniture to wall-diagram layout, erase whiteboards, and pack waste to hallway Sort It Out stations."*
4. **Live Status Actions (3 Mutually Exclusive Tap Buttons for the Current Session):**
   * **`[ ✅ Check In Here ]`:** Increments the room's live concurrent student count by 1 on the server and changes the button to **`[ 👋 Check Out ]`**. A user can only be checked into **one room at a time** (checking into a new room automatically checks them out of their previous room).
   * **`[ 🔒 Flag: Room Locked / Event in Progress ]`:** Marks the room as temporarily unavailable on the server for the current hour block, removing it from active search recommendations for other users and displaying a yellow warning badge (`Reported Occupied/Locked`).
   * **`[ ✖ Close ]`:** Closes the modal and returns to the results list.

**Error State on Screen 2:**
* **Room Full / Recently Flagged Error:** If another client flags the room as locked or checks into the last remaining seat while the user has the modal open, tapping `[ ✅ Check In Here ]` displays an alert banner: *"This room was just reported locked or full by another student. Please select another room."* and refreshes the room's state.

---

#### Screen 3: Group "Meet-in-the-Middle" Mode

When the user taps the **`[ Group Meet-in-the-Middle ]`** tab at the top of the app, the input panel switches to Group Coordination Mode for **2 to 4 teammates**.

<!-- TODO: Embed Screen 3 Sketch Here: ![Screen 3 Mockup](./mockups/screen3-group-mode.png) -->

**Interactive Controls:**
1. **Day of Week Selector** (`Mon`–`Fri`) and **Minimum Meeting Duration Dropdown** (`30 min`, `60 min`, `90 min`, `120 min`).
2. **Teammate Rows (2 rows shown by default; `[ + Add Teammate ]` button adds up to 4 rows total; `[ 🗑 Remove ]` button removes a row down to a minimum of 2):**
   * Each teammate row contains:
     * **Current Building Dropdown** (where that teammate is walking from).
     * **Busy Time Blocks Selector:** Tap-able 1-hour blocks (`08:00` through `20:00`) that the teammate taps to mark when they are in class/busy (selected busy blocks turn dark gray).
3. **`[ Find Fair Group Room & Time ]` Button.**

**Algorithmic Behavior (Interval Intersection + Minimax Walking Center):**
1. **Temporal Intersection:** The server computes the complement of the union of all teammates' busy blocks to identify all contiguous common free windows of length $\ge$ `Minimum Meeting Duration`.
2. **Minimax Spatial Fairness:** For each common free window, the server filters for rooms with Capacity $\ge$ group size that are unoccupied for the entire window. It ranks candidate rooms by minimizing the **maximum walking time experienced by any single teammate** ($\min_{R} \max_{i \in \text{Group}} t_{\text{walk}}(\text{Building}_i, R)$), breaking ties by lowest total group walking time and lowest room congestion.

**Validation & Error States on Screen 3:**
* **No Overlapping Free Time Error:** If the teammates' busy blocks leave no common free window of at least the requested duration, an inline error banner displays: *"No overlapping free time of at least [X] minutes found across all teammates' schedules."*
