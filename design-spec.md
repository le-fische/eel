# CPEN 221A - Milestone 1: Design Specification

## 1. Design Your Team

**Team Roles & Responsibilities:**
*   **Project Manager (PM):** Eli Hanania
*   **Designer:** Emi Harker
*   **Developer:** Houze Guo

**Mission Statement:**
Our mission is to advance academic equity for commuter students by transforming underutilized campus infrastructure into accessible, collaborative learning environments. We champion student wellbeing by eliminating the daily friction of finding a workspace, ensuring every student has a place to belong and focus on campus.

---

## 2. Design Your Solution

### Problem
Over 60% of UBC Vancouver’s 60,000+ students commute to campus, frequently navigating schedules with awkward 50-minute to 3-hour gaps between lectures. During peak hours (10:00 AM – 4:00 PM), dedicated study hubs such as the Irving K. Barber Learning Centre, Koerner Library, and the AMS Nest routinely reach full capacity, while bookable group rooms are reserved weeks in advance. Paradoxically, this space shortage is an information problem rather than a physical lack of rooms. Across campus, dozens of unlocked departmental classrooms and lecture halls sit completely empty at any given hour. This mismatch stems from a rigid separation between formal academic scheduling and informal student space usage, leaving thousands of square feet of functional study infrastructure hidden in plain sight.

The inability to secure a reliable workspace directly undermines student productivity, well-being, and equitable access to education. Commuter students who cannot practically return home are left wandering buildings in search of an open seat and a power outlet, or settling for noisy hallway floors and stairwells. This daily scavenger hunt wastes hours of instructional recovery time every week, increases physical fatigue, and disproportionately penalizes students who lack the privilege of private residence rooms or exclusive departmental lounge access. When students attempt to collaborate on team projects, the problem compounds; coordinating overlapping free windows alongside a mutually accessible, unoccupied room requires tedious back-and-forth guesswork.

Current student workarounds are highly inefficient and unreliable for day-to-day use. Some students walk down hallways pulling on doors at random—frequently interrupting active classes or settling into a room only to be displaced ten minutes later by an incoming lecture. Others manually cross-reference official university timetable portals, which are organized by course code rather than spatial location. These static, room-by-room interfaces neither account for walking travel times nor reflect real-time conditions on the ground, such as locked doors or unscheduled club meetings. Because these improvised methods require high effort for uncertain results, students remain bottlenecked in overcrowded libraries while perfectly viable classrooms down the hall sit unused.

### Solution
We envision a mobile-responsive web application that maps student schedules and physical locations to real-time classroom availability across the UBC campus. The application contains no sound and no keyboard support; its interface layout is responsive to screen size but uses the same layout for all platforms, optimized for one-handed tapping while walking. 

The primary interface is the solo routing search screen, designed for rapid data entry:

> `[PLACEHOLDER: EMI, INSERT SKETCH OF MAIN SEARCH SCREEN HERE]`

Players tap the "Current Location" and "Next Class Location" fields to select buildings from an autocomplete dropdown, and use time pickers to establish their gap window. When a player submits their parameters by tapping the "Find Space" button, the system validates the input. If any fields are empty, the button shakes slightly, the missing input borders turn red, and a tooltip prompts the user to fill them in. 

If the inputs are valid, the application queries the server. Behind the scenes, the engine uses an Interval Tree to parse the UBC classroom schedule dataset and runs Dijkstra/A* on a CampusGraph of UBC walkways. It finds rooms where the available time window covers the user's gap *plus* the walking time to and from the buildings. The system then transitions to the Results view:

> `[PLACEHOLDER: EMI, INSERT SKETCH OF RESULTS & MAP SCREEN HERE]`

The Results screen is a split view featuring a simplified campus map on top and a ranked list of available rooms below. Each list item displays the Room Number, the walking detour time (e.g., "+4 mins"), and guaranteed free time. To prevent the "Waze Effect"—where one empty room is flooded by 50 nearby searchers—the backend ranking applies a congestion-weighted score. As live check-ins approach a room's physical capacity, the algorithm penalizes that room's score, dynamically load-balancing students across other nearby empty rooms. If no rooms perfectly match the criteria, the system displays a friendly error ("No perfectly matching rooms found") and provides a "Next Best Options" list that relaxes the walking distance constraint by 10 minutes.

Users can tap a specific list item to expand it, revealing the exact walking path on the map and transitioning to the Room Details state:

> `[PLACEHOLDER: EMI, INSERT SKETCH OF ROOM DETAILS / CHECK-IN SCREEN HERE]`

This screen displays large typography with a countdown timer ("Lecture starts in 1h 15m") and two large action buttons. When a user taps "Check In Here" upon arriving, the server increments an atomic occupancy counter using thread-safe read-write locks, and the user's screen flashes green to confirm. 

If a user arrives to find the room locked or taken by an unscheduled club, they tap "Report Locked/Occupied". A toast notification ("Routing you to the next closest room...") appears for 2 seconds. The server instantly removes that room from the active query pool for the next 2 hours, recalculates Dijkstra's algorithm, and returns the user to the updated Results screen. 

Finally, the application supports a "Group Meet-in-the-Middle" mode, accessed via a toggle switch on the main screen:

> `[PLACEHOLDER: EMI, INSERT SKETCH OF GROUP SETUP SCREEN HERE]`

In Group Mode, the interface provides a dynamic list where users can tap "+ Add Member" to add up to 3 additional rows. Each row requires a "Current Building" and "Busy Blocks" (time ranges). Tapping "Calculate Optimal Room" triggers a Multi-Schedule Interval Intersection algorithm to find common free time slots, and solves the Minimax Graph Center problem to output the classroom that requires the fairest walking distance for everyone. If members input conflicting schedules with strictly zero overlapping free time, an error modal alerts them to adjust their time windows. 

If a user loses connection at any point, a persistent top banner appears reading "Offline Mode: Waiting for connection," and all interactive buttons become disabled (turn gray) until connectivity is restored.
