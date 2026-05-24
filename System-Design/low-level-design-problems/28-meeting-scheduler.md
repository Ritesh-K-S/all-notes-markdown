# 28 - Meeting / Calendar Scheduler

## 📋 Problem Statement

Design a meeting scheduler / calendar system that:
- Allows users to schedule, update, and cancel meetings
- Checks participant availability (no conflicts)
- Supports recurring meetings and room booking

---

## 📌 Requirements

### Functional Requirements
1. **Schedule a meeting** — title, time, duration, participants, room
2. **Check availability** of participants for a time slot
3. **Find available slots** for a set of participants
4. **Book meeting rooms** based on capacity
5. **Recurring meetings** — daily, weekly, monthly
6. **Cancel / Update** meetings
7. **Send invites** and handle RSVP (Accept/Decline/Tentative)
8. **Calendar view** — day, week, month

### Non-Functional Requirements
- No double-booking of rooms or conflicting meetings
- Efficient overlap detection
- Notification on invite/update/cancel

---

## 🧩 Core Entities

```
User, Meeting, Calendar, MeetingRoom, TimeSlot,
Recurrence, Invite, InviteStatus, MeetingScheduler
```

---

## 🔧 Enums

```java
public enum InviteStatus {
    PENDING, ACCEPTED, DECLINED, TENTATIVE
}

public enum RecurrenceType {
    NONE, DAILY, WEEKLY, BIWEEKLY, MONTHLY
}
```

---

## 💻 Code Implementation

### TimeSlot Class

```java
public class TimeSlot {
    private LocalDateTime start;
    private LocalDateTime end;

    public TimeSlot(LocalDateTime start, LocalDateTime end) {
        if (!end.isAfter(start)) throw new IllegalArgumentException("End must be after start");
        this.start = start;
        this.end = end;
    }

    public boolean overlapsWith(TimeSlot other) {
        return this.start.isBefore(other.end) && other.start.isBefore(this.end);
    }

    public long getDurationMinutes() {
        return Duration.between(start, end).toMinutes();
    }

    public LocalDateTime getStart() { return start; }
    public LocalDateTime getEnd() { return end; }
}
```

### Meeting Class

```java
public class Meeting {
    private String meetingId;
    private String title;
    private String description;
    private TimeSlot timeSlot;
    private User organizer;
    private Map<String, InviteStatus> participants; // userId → status
    private MeetingRoom room;
    private RecurrenceType recurrence;
    private LocalDateTime createdAt;

    public Meeting(String title, TimeSlot timeSlot, User organizer) {
        this.meetingId = UUID.randomUUID().toString();
        this.title = title;
        this.timeSlot = timeSlot;
        this.organizer = organizer;
        this.participants = new LinkedHashMap<>();
        this.recurrence = RecurrenceType.NONE;
        this.createdAt = LocalDateTime.now();

        // Organizer auto-accepts
        participants.put(organizer.getUserId(), InviteStatus.ACCEPTED);
    }

    public void addParticipant(User user) {
        participants.put(user.getUserId(), InviteStatus.PENDING);
    }

    public void respondToInvite(String userId, InviteStatus status) {
        if (!participants.containsKey(userId)) {
            throw new NotInvitedException("User not invited to this meeting");
        }
        participants.put(userId, status);
    }

    public void setRoom(MeetingRoom room) { this.room = room; }
    public void setRecurrence(RecurrenceType recurrence) { this.recurrence = recurrence; }

    public boolean conflictsWith(TimeSlot other) {
        return timeSlot.overlapsWith(other);
    }

    // Getters
    public String getMeetingId() { return meetingId; }
    public TimeSlot getTimeSlot() { return timeSlot; }
    public User getOrganizer() { return organizer; }
    public Map<String, InviteStatus> getParticipants() { return participants; }
    public MeetingRoom getRoom() { return room; }
    public String getTitle() { return title; }
}
```

### Calendar (Per User)

```java
public class Calendar {
    private String userId;
    private List<Meeting> meetings;

    public Calendar(String userId) {
        this.userId = userId;
        this.meetings = new ArrayList<>();
    }

    public void addMeeting(Meeting meeting) {
        meetings.add(meeting);
    }

    public void removeMeeting(String meetingId) {
        meetings.removeIf(m -> m.getMeetingId().equals(meetingId));
    }

    public boolean isAvailable(TimeSlot timeSlot) {
        return meetings.stream()
            .noneMatch(m -> m.conflictsWith(timeSlot));
    }

    public List<Meeting> getMeetingsForDay(LocalDate date) {
        return meetings.stream()
            .filter(m -> m.getTimeSlot().getStart().toLocalDate().equals(date))
            .sorted(Comparator.comparing(m -> m.getTimeSlot().getStart()))
            .collect(Collectors.toList());
    }

    public List<Meeting> getMeetingsInRange(LocalDateTime start, LocalDateTime end) {
        TimeSlot range = new TimeSlot(start, end);
        return meetings.stream()
            .filter(m -> m.conflictsWith(range))
            .sorted(Comparator.comparing(m -> m.getTimeSlot().getStart()))
            .collect(Collectors.toList());
    }

    public List<Meeting> getAllMeetings() { return meetings; }
}
```

### Meeting Room

```java
public class MeetingRoom {
    private String roomId;
    private String name;
    private int capacity;
    private int floor;
    private List<Meeting> bookings;

    public MeetingRoom(String roomId, String name, int capacity, int floor) {
        this.roomId = roomId;
        this.name = name;
        this.capacity = capacity;
        this.floor = floor;
        this.bookings = new ArrayList<>();
    }

    public synchronized boolean book(Meeting meeting) {
        TimeSlot timeSlot = meeting.getTimeSlot();
        boolean conflict = bookings.stream().anyMatch(m -> m.conflictsWith(timeSlot));
        if (conflict) return false;

        bookings.add(meeting);
        return true;
    }

    public void cancelBooking(String meetingId) {
        bookings.removeIf(m -> m.getMeetingId().equals(meetingId));
    }

    public boolean isAvailable(TimeSlot timeSlot) {
        return bookings.stream().noneMatch(m -> m.conflictsWith(timeSlot));
    }

    // Getters
    public int getCapacity() { return capacity; }
    public String getName() { return name; }
    public String getRoomId() { return roomId; }
}
```

### Meeting Scheduler (Controller)

```java
public class MeetingScheduler {
    private Map<String, User> users;
    private Map<String, Calendar> calendars; // userId → Calendar
    private List<MeetingRoom> rooms;
    private Map<String, Meeting> meetings;
    private NotificationService notificationService;

    public MeetingScheduler() {
        this.users = new ConcurrentHashMap<>();
        this.calendars = new ConcurrentHashMap<>();
        this.rooms = new ArrayList<>();
        this.meetings = new ConcurrentHashMap<>();
        this.notificationService = new NotificationService();
    }

    public Meeting scheduleMeeting(String title, TimeSlot timeSlot,
                                    User organizer, List<User> participants,
                                    int requiredCapacity) {
        // Step 1: Check all participants' availability
        List<User> unavailable = new ArrayList<>();
        for (User participant : participants) {
            Calendar cal = calendars.get(participant.getUserId());
            if (cal != null && !cal.isAvailable(timeSlot)) {
                unavailable.add(participant);
            }
        }

        if (!unavailable.isEmpty()) {
            throw new ParticipantBusyException("Unavailable: " +
                unavailable.stream().map(User::getName).collect(Collectors.joining(", ")));
        }

        // Step 2: Find and book a room
        MeetingRoom room = findAvailableRoom(timeSlot, requiredCapacity);
        if (room == null) throw new NoRoomAvailableException("No rooms available");

        // Step 3: Create meeting
        Meeting meeting = new Meeting(title, timeSlot, organizer);
        for (User participant : participants) {
            meeting.addParticipant(participant);
        }
        meeting.setRoom(room);
        room.book(meeting);

        // Step 4: Add to all calendars
        calendars.get(organizer.getUserId()).addMeeting(meeting);
        for (User participant : participants) {
            calendars.get(participant.getUserId()).addMeeting(meeting);
        }

        meetings.put(meeting.getMeetingId(), meeting);

        // Step 5: Send invites
        for (User participant : participants) {
            notificationService.sendMeetingInvite(participant, meeting);
        }

        return meeting;
    }

    public void cancelMeeting(String meetingId, User requestor) {
        Meeting meeting = meetings.get(meetingId);
        if (meeting == null) throw new MeetingNotFoundException("Meeting not found");
        if (!meeting.getOrganizer().equals(requestor)) {
            throw new UnauthorizedException("Only organizer can cancel");
        }

        // Remove from all calendars
        for (String userId : meeting.getParticipants().keySet()) {
            calendars.get(userId).removeMeeting(meetingId);
        }

        // Release room
        if (meeting.getRoom() != null) {
            meeting.getRoom().cancelBooking(meetingId);
        }

        meetings.remove(meetingId);

        // Notify all participants
        for (String userId : meeting.getParticipants().keySet()) {
            notificationService.sendCancellationNotice(users.get(userId), meeting);
        }
    }

    public void respondToInvite(String meetingId, User user, InviteStatus response) {
        Meeting meeting = meetings.get(meetingId);
        if (meeting == null) throw new MeetingNotFoundException("Meeting not found");
        meeting.respondToInvite(user.getUserId(), response);

        if (response == InviteStatus.DECLINED) {
            calendars.get(user.getUserId()).removeMeeting(meetingId);
        }
    }

    /**
     * Find common free slots for a set of participants within a date range.
     */
    public List<TimeSlot> findAvailableSlots(List<User> participants,
                                              LocalDate date,
                                              Duration duration) {
        // Collect all busy slots for the day
        List<TimeSlot> busySlots = new ArrayList<>();
        for (User user : participants) {
            Calendar cal = calendars.get(user.getUserId());
            for (Meeting m : cal.getMeetingsForDay(date)) {
                busySlots.add(m.getTimeSlot());
            }
        }

        // Sort busy slots by start time
        busySlots.sort(Comparator.comparing(TimeSlot::getStart));

        // Find gaps that fit the required duration
        List<TimeSlot> freeSlots = new ArrayList<>();
        LocalDateTime dayStart = date.atTime(9, 0);  // office hours: 9 AM
        LocalDateTime dayEnd = date.atTime(18, 0);    // to 6 PM
        LocalDateTime current = dayStart;

        for (TimeSlot busy : busySlots) {
            if (Duration.between(current, busy.getStart()).compareTo(duration) >= 0) {
                freeSlots.add(new TimeSlot(current, busy.getStart()));
            }
            if (busy.getEnd().isAfter(current)) {
                current = busy.getEnd();
            }
        }

        // Check remaining time after last meeting
        if (Duration.between(current, dayEnd).compareTo(duration) >= 0) {
            freeSlots.add(new TimeSlot(current, dayEnd));
        }

        return freeSlots;
    }

    private MeetingRoom findAvailableRoom(TimeSlot timeSlot, int capacity) {
        return rooms.stream()
            .filter(r -> r.getCapacity() >= capacity)
            .filter(r -> r.isAvailable(timeSlot))
            .min(Comparator.comparingInt(MeetingRoom::getCapacity)) // smallest fitting room
            .orElse(null);
    }

    public void registerUser(User user) {
        users.put(user.getUserId(), user);
        calendars.put(user.getUserId(), new Calendar(user.getUserId()));
    }

    public void addRoom(MeetingRoom room) {
        rooms.add(room);
    }
}
```

---

## 🧪 Usage Example

```java
MeetingScheduler scheduler = new MeetingScheduler();

// Register users
User alice = new User("U1", "Alice");
User bob = new User("U2", "Bob");
User charlie = new User("U3", "Charlie");
scheduler.registerUser(alice);
scheduler.registerUser(bob);
scheduler.registerUser(charlie);

// Add rooms
scheduler.addRoom(new MeetingRoom("R1", "Conf Room A", 5, 1));
scheduler.addRoom(new MeetingRoom("R2", "Board Room", 20, 2));

// Schedule meeting
TimeSlot slot = new TimeSlot(
    LocalDateTime.of(2026, 4, 20, 10, 0),
    LocalDateTime.of(2026, 4, 20, 11, 0)
);
Meeting meeting = scheduler.scheduleMeeting(
    "Sprint Planning", slot, alice, List.of(bob, charlie), 5
);

// Bob accepts
scheduler.respondToInvite(meeting.getMeetingId(), bob, InviteStatus.ACCEPTED);

// Charlie declines
scheduler.respondToInvite(meeting.getMeetingId(), charlie, InviteStatus.DECLINED);

// Find free slots
List<TimeSlot> freeSlots = scheduler.findAvailableSlots(
    List.of(alice, bob),
    LocalDate.of(2026, 4, 20),
    Duration.ofMinutes(30)
);
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Observer** | Notifications on invite/cancel/update |
| **Strategy** | Room selection strategy (smallest fit, preferred floor) |
| **Factory** | Creating recurring meeting instances |

---

## ⚠️ Edge Cases

- All participants busy → suggest alternative slots
- Room too small → find bigger room
- Organizer cancels → notify all, release room
- Recurring meeting conflict on one date → skip that instance
- Timezone differences → store in UTC, display in local
- Concurrent room booking → synchronized `book()` method

---

## 🔑 Key Takeaways

1. **TimeSlot.overlapsWith()** is the core logic — `start1 < end2 && start2 < end1`
2. **findAvailableSlots()** merges busy intervals and finds gaps — classic interval problem
3. **Room booking** picks smallest fitting room — avoids wasting large rooms
4. **Per-user Calendar** with conflict detection
5. **RSVP system** with per-participant status tracking
