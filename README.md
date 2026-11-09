# Aircraft Seat Allocation Optimizer (Core Java)

A console application that registers passengers, allocates seats by preference,
seats groups together, supports reallocation, prints reports, and saves/restores
data from a text file. Core Java only (Java 8+): no enums, lambdas, streams,
databases or third-party libraries. It works fully offline.

## 1. Architecture

```
Main  --->  InputHandler            (keyboard input, validation loops)
  |
  +----->  PassengerService         (who is registered)          HashMap
  +----->  SeatAllocationService    (the ONLY class that changes seat/passenger state)
  |            +-- AllocationStrategy (interface) <- PreferenceBasedStrategy
  |            +-- Aircraft  -> Seat  (18 seats, layout, adjacent-block search)
  +----->  ReportService            (seating map, tables, report text)
  +----->  FileStorage              (save/load data/passengers.txt)
```

Design decisions:

- **Single owner of state.** Only `SeatAllocationService` links a seat to a passenger
  (private `assign()` method). `Seat.occupy()` refuses an occupied seat, and
  `Passenger.assignSeat/clearSeat` are package-private. This keeps the rules
  "one seat = one passenger" and "one passenger = one seat" consistent.
- **Seat status is derived** from the occupant (`occupant == null` means AVAILABLE),
  so status and occupant can never disagree.
- **Validate first, change second.** Reallocation and group allocation check everything
  before modifying anything, which gives "failed reallocation keeps the old seat" and
  "group allocation is all-or-nothing".
- **"Adjacent" means consecutive seats on the same side of the aisle.** Seats C and D
  are across the aisle, so they are not adjacent. Blocks are A-C or D-F, so a group
  can have at most 3 members to be seated together.
- **Seat types:** A, F = WINDOW; B, E = MIDDLE; C, D = AISLE.

## 2. Folder structure

```
AircraftSeatOptimizer/
├── README.md
├── data/
│   └── passengers.txt
└── src/
    └── aircraftseatoptimizer/
        ├── Main.java
        ├── Aircraft.java
        ├── Seat.java
        ├── Passenger.java
        ├── SeatAssignment.java
        ├── AllocationStrategy.java
        ├── PreferenceBasedStrategy.java
        ├── PassengerService.java
        ├── SeatAllocationService.java
        ├── ReportService.java
        ├── FileStorage.java
        ├── InputHandler.java
        └── exceptions/
            ├── PassengerNotFoundException.java
            ├── SeatUnavailableException.java
            └── AllocationException.java
```

## 3. Compile and run

Run these from the `AircraftSeatOptimizer` folder (the program looks for `data/` in the
current directory; it is created automatically if missing).

Linux / macOS:
```
mkdir -p out
javac -d out src/aircraftseatoptimizer/*.java src/aircraftseatoptimizer/exceptions/*.java
java -cp out aircraftseatoptimizer.Main
```

Windows (Command Prompt):
```
mkdir out
javac -d out src\aircraftseatoptimizer\*.java src\aircraftseatoptimizer\exceptions\*.java
java -cp out aircraftseatoptimizer.Main
```

## 4. How to use

| Menu | What it does |
|------|--------------|
| 1 | Register passenger: ID, name, age, preference, optional group ID |
| 2 | View all passengers (sorted by ID) |
| 3 | Search passenger by ID |
| 4 | Allocate a seat to ONE passenger (preference first, then any free seat) |
| 5 | Allocate everyone: groups first (all-or-nothing), then individuals |
| 6 | Move a passenger to a requested seat (e.g. 2C) |
| 7 | Remove passenger (their seat is released first, after confirmation) |
| 8 | Seating map and available seats |
| 9 | Allocation report (counts, assignments, unassigned) |
| 10 | Save to `data/passengers.txt` |
| 11 | Save and exit |

Notes:
- IDs and seat numbers are not case-sensitive (`p1` = `P1`, `2c` = `2C`).
- A group that cannot be seated together stays **unassigned** in option 5. Seat its members
  one by one with option 4 (or free up seats and run option 5 again).
- Group IDs only matter for groups of 2 or more. A group of one is treated as an individual.
- Names and IDs may not contain `|` (it is the file separator).

### Data file format

One passenger per line: `id|name|age|preference|groupId|seatId`
(an empty `groupId` / `seatId` means none). Lines starting with `#` are comments.

When loading, bad lines (wrong field count, bad age, ...) are skipped with a warning.
A duplicate passenger ID is skipped. If a saved seat does not exist or is already taken
by an earlier record, the passenger is kept but loaded **without a seat**.

## 5. Sample console interaction

```
=== Aircraft Seat Allocation Optimizer ===
Loaded 0 passenger record(s) from data/passengers.txt.

---------------- MENU ----------------
 1. Register passenger
 ...
11. Save and exit
--------------------------------------
Enter choice (1-11): 1
Passenger ID: P1
Name: Asha
Age (1-120): 20
Seat preference (WINDOW/MIDDLE/AISLE/ANY): window
Group ID (press Enter for none): G1
Registered: P1 - Asha (age 20, prefers WINDOW, group G1, seat none)
Enter choice (1-11): 1
Passenger ID: P2
Name: Ravi
Age (1-120): 25
Seat preference (WINDOW/MIDDLE/AISLE/ANY): any
Group ID (press Enter for none): G1
Registered: P2 - Ravi (age 25, prefers ANY, group G1, seat none)
Enter choice (1-11): 5
Group G1: 1A   WINDOW  Asha (P1)
Group G1: 1B   MIDDLE  Ravi (P2)
Enter choice (1-11): 6
Passenger ID: P1
Requested seat (e.g. 2C): 1B
Reallocation failed: Seat 1B is already occupied. Asha keeps seat 1A.
Enter choice (1-11): 6
Passenger ID: P1
Requested seat (e.g. 2C): 3F
Success: Asha is now in seat 3F [WINDOW].
Enter choice (1-11): 8

===== SEATING MAP =====
        A   B   C       D   E   F
Row 1  [ ] [X] [ ]  |  [ ] [ ] [ ]
Row 2  [ ] [ ] [ ]  |  [ ] [ ] [ ]
Row 3  [ ] [ ] [ ]  |  [ ] [ ] [X]

[ ] available   [X] occupied
Window: A, F   Middle: B, E   Aisle: C, D
Available seats (16): 1A 1C 1D 1E 1F 2A 2B 2C 2D 2E 2F 3A 3B 3C 3D 3E
Enter choice (1-11): 9

===== ALLOCATION REPORT =====
Total seats            : 18
Occupied seats         : 2
Available seats        : 16
Registered passengers  : 2
Assigned passengers    : 2
Unassigned passengers  : 0

--- Seat assignments ---
1B   MIDDLE  Ravi (P2)
3F   WINDOW  Asha (P1)

--- Unassigned passengers ---
(none)

Available seats: 1A 1C 1D 1E 1F 2A 2B 2C 2D 2E 2F 3A 3B 3C 3D 3E
Enter choice (1-11): 11
Data saved (2 passenger(s)).
Goodbye!
```

## 6. Where each Core Java concept is used

| # | Concept | Where |
|---|---------|-------|
| 1 | Classes and objects | Every class; `Aircraft` creates 18 `Seat` objects |
| 2 | Constructors | `Passenger(...)` validates input; `Seat(row, column, type)`; service constructors take their dependencies |
| 3 | Encapsulation | Private fields everywhere; `Passenger.assignSeat/clearSeat` are package-private; seat status derived from occupant |
| 4 | Inheritance | The three custom exceptions `extends Exception` (not forced anywhere else) |
| 5 | Abstraction | `AllocationStrategy` hides how a seat is chosen; `FileStorage` hides file details |
| 6 | Interfaces | `AllocationStrategy`, also `Comparator` |
| 7 | Polymorphism | `SeatAllocationService` calls `strategy.selectSeat(...)` on an `AllocationStrategy` variable holding a `PreferenceBasedStrategy`; `@Override` of `toString`/`compare` |
| 8 | ArrayList | Seat list in `Aircraft`, passenger lists, block of seats, log lines |
| 9 | HashMap | `Aircraft.seatLookup` (seat ID -> Seat); `PassengerService.passengers` (ID -> Passenger) |
| 10 | HashSet | `PassengerService.getTravellingGroupIds()` collects unique group IDs |
| 11 | Comparator | Anonymous `Comparator<Passenger>` (by ID) in `PassengerService`; anonymous `Comparator<SeatAssignment>` (by seat) in `ReportService` |
| 12 | Exception handling | try/catch in `Main` actions, `FileStorage` try-with-resources, safety-net `catch (RuntimeException)` in the main loop |
| 13 | Custom exceptions | `PassengerNotFoundException`, `SeatUnavailableException`, `AllocationException` |
| 14 | Generics | `List<Seat>`, `Map<String, Passenger>`, `Set<String>`, generic method `<T> List<T> sortedCopy(...)` |
| 15 | File handling | `FileStorage`: `File`, `BufferedReader/Writer`, `mkdirs`, `createNewFile`, loading with validation |
| 16 | static and final | `static final` constants (`Seat.WINDOW`, `Aircraft.ROWS`, `FileStorage.DATA_FILE_NAME`); `final` fields; static helpers `Passenger.normalizeId`, `Passenger.isValidPreference` |
| 17 | String handling | `trim`, `toUpperCase`, `split("\\|", -1)`, `String.format`, `StringBuilder`, `equals`, `startsWith`, `indexOf` |

## 7. Test scenarios (manual) and expected results

Start each scenario with an empty `data/passengers.txt` (just the `#` comment line),
unless it says otherwise. All were run against this code.

| # | Scenario | Steps | Expected result |
|---|----------|-------|-----------------|
| 1 | Window seat | Register P1 (pref WINDOW) -> menu 4, `P1` | `Allocated seat 1A [WINDOW]`; map shows 1A as `[X]` |
| 2 | Aisle seat | Register P1 (pref AISLE) -> menu 4, `P1` | `Allocated seat 1C [AISLE]` |
| 3 | Preference fallback | Register 7 passengers, all WINDOW -> menu 5 | First six get 1A, 1F, 2A, 2F, 3A, 3F; the 7th gets 1B `[MIDDLE]` (no window left) |
| 4 | Group with space | Register 3 passengers, all group `G1`, pref ANY -> menu 5 | `Group G1:` lines show 1A, 1B, 1C (same row, consecutive) |
| 5 | Group, no block | Register 6 passengers with pref MIDDLE -> menu 5 (they take 1B, 1E, 2B, 2E, 3B, 3E). Register 2 passengers in group `TRIP` -> menu 5 | `Group TRIP left unassigned: No block of 2 adjacent seats is available...`; both members stay unassigned (report shows them) |
| 6 | Same seat requested twice | Register P1, P2 (no seats yet) -> menu 6: P1 -> `2C`, then P2 -> `2C` | P1: `Success`; P2: `Reallocation failed: Seat 2C is already occupied.`; P2 still has no seat |
| 7 | Reallocate to occupied seat | Register P1, P2 -> menu 5 (1A, 1B) -> menu 6: P1 -> `1B` | `Reallocation failed: Seat 1B is already occupied. Asha keeps seat 1A.`; search P1 shows seat 1A |
| 8 | Reallocate to free seat | P1 allocated 1A -> menu 6: P1 -> `3F` | `Success`; search shows 3F; map shows 1A free and 3F occupied |
| 9 | Invalid passenger ID | Menus 3, 4, 6, 7 with `ZZ9` | `Error: No passenger found with ID 'ZZ9'.`; program keeps running |
| 10 | Invalid seat number | P1 has 1A -> menu 6: P1 -> `9Z` (also try `abc`) | `Reallocation failed: Seat '9Z' does not exist on this aircraft.`; P1 still in 1A |
| 11 | Restart and restore | Register P1, P2 (group G1) -> menu 5 -> menu 11. Start the program again -> menu 2, 8 | `Loaded 2 passenger record(s)`; same seats as before; map shows them occupied |
| 12 | Full aircraft | Register 19 passengers (pref ANY) -> menu 5 -> menu 9 | 18 seated; the 19th: `No seats remain ... is unallocated`; report: Occupied 18, Available 0, Unassigned 1, `Available seats: (none)` |

Extra check (data validation): put these lines in `data/passengers.txt` and start the program:

```
P1|Asha|20|WINDOW||1A
P2|Ravi|25|ANY||1A
P3|Meera|31|AISLE||9Z
P1|Dup|40|ANY||
P4|Bad|abc|ANY||
garbage line
```

Expected: warnings for the bad-age line, the garbage line, the seat conflict (P2),
the non-existent seat (P3) and the duplicate P1. P1 keeps 1A; P2 and P3 load without seats.
