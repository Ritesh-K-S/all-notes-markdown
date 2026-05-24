# 02 - Elevator System

## 📋 Problem Statement

Design an elevator system for a building that can:
- Handle multiple elevators across multiple floors
- Optimize elevator assignment based on direction and distance
- Support internal (inside elevator) and external (floor) requests

---

## 📌 Requirements

### Functional Requirements
1. Building has **N floors** and **M elevators**
2. Each floor has **Up/Down buttons** (external request)
3. Inside the elevator, passengers press **destination floor** (internal request)
4. Elevator moves **Up, Down, or stays Idle**
5. System assigns the **optimal elevator** for an external request
6. Elevator opens door at requested floors in its path
7. Display shows current floor and direction

### Non-Functional Requirements
- Minimize **wait time** and **travel time**
- Handle **concurrent requests**
- System should be **scalable** (add more elevators)

---

## 🧩 Core Entities

```
ElevatorSystem, Elevator, Floor, Request, Door, 
Display, InternalButton, ExternalButton, Direction, ElevatorState
```

---

## 📐 Class Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     ElevatorSystem                          │
│  - elevators: List<Elevator>                                │
│  - floors: List<Floor>                                      │
│  - strategy: ElevatorSelectionStrategy                      │
│  + handleExternalRequest(floor, direction): void            │
│  + handleInternalRequest(elevatorId, floor): void           │
└───────────────┬─────────────────────────────────────────────┘
                │ 1..*
┌───────────────▼─────────────────────────────────────────────┐
│                       Elevator                              │
│  - id: int                                                  │
│  - currentFloor: int                                        │
│  - direction: Direction                                     │
│  - state: ElevatorState                                     │
│  - upQueue: PriorityQueue<Integer> (min-heap)               │
│  - downQueue: PriorityQueue<Integer> (max-heap)             │
│  - door: Door                                               │
│  - display: Display                                         │
│  + addRequest(floor): void                                  │
│  + move(): void                                             │
│  + openDoor(): void                                         │
│  + closeDoor(): void                                        │
└─────────────────────────────────────────────────────────────┘

┌──────────────────┐    ┌──────────────────┐
│   Direction       │    │  ElevatorState   │
│  ─ ─ ─ ─ ─ ─     │    │  ─ ─ ─ ─ ─ ─    │
│  UP               │    │  MOVING_UP       │
│  DOWN             │    │  MOVING_DOWN     │
│  IDLE             │    │  IDLE            │
│                   │    │  MAINTENANCE     │
└──────────────────┘    └──────────────────┘

┌──────────────────────────────────────────┐
│              Floor                        │
│  - floorNumber: int                       │
│  - upButton: ExternalButton               │
│  - downButton: ExternalButton             │
│  + pressUp(): void                        │
│  + pressDown(): void                      │
└──────────────────────────────────────────┘
```

---

## 🔧 Enums

```java
public enum Direction {
    UP, DOWN, IDLE
}

public enum ElevatorState {
    MOVING_UP, MOVING_DOWN, IDLE, MAINTENANCE
}

public enum DoorState {
    OPEN, CLOSED
}
```

---

## 💻 Code Implementation

### Request Class

```java
public class Request {
    private int floor;
    private Direction direction;
    private LocalDateTime timestamp;

    public Request(int floor, Direction direction) {
        this.floor = floor;
        this.direction = direction;
        this.timestamp = LocalDateTime.now();
    }

    public int getFloor() { return floor; }
    public Direction getDirection() { return direction; }
}
```

### Door & Display

```java
public class Door {
    private DoorState state;

    public Door() {
        this.state = DoorState.CLOSED;
    }

    public void open() {
        if (state == DoorState.CLOSED) {
            state = DoorState.OPEN;
            System.out.println("Door opened");
        }
    }

    public void close() {
        if (state == DoorState.OPEN) {
            state = DoorState.CLOSED;
            System.out.println("Door closed");
        }
    }
}

public class Display {
    private int currentFloor;
    private Direction direction;

    public void update(int floor, Direction dir) {
        this.currentFloor = floor;
        this.direction = dir;
        System.out.println("Floor: " + floor + " | Direction: " + dir);
    }
}
```

### Elevator Class

```java
public class Elevator {
    private int id;
    private int currentFloor;
    private Direction direction;
    private ElevatorState state;
    private Door door;
    private Display display;

    // Min-heap for going UP (process lowest floor first)
    private PriorityQueue<Integer> upQueue;
    // Max-heap for going DOWN (process highest floor first)
    private PriorityQueue<Integer> downQueue;

    public Elevator(int id) {
        this.id = id;
        this.currentFloor = 0;
        this.direction = Direction.IDLE;
        this.state = ElevatorState.IDLE;
        this.door = new Door();
        this.display = new Display();
        this.upQueue = new PriorityQueue<>();
        this.downQueue = new PriorityQueue<>(Collections.reverseOrder());
    }

    public void addRequest(int floor) {
        if (floor > currentFloor) {
            upQueue.offer(floor);
        } else if (floor < currentFloor) {
            downQueue.offer(floor);
        }
        // If currently idle, start moving
        if (state == ElevatorState.IDLE) {
            processRequests();
        }
    }

    public void processRequests() {
        while (!upQueue.isEmpty() || !downQueue.isEmpty()) {
            processUpRequests();
            processDownRequests();
        }
        state = ElevatorState.IDLE;
        direction = Direction.IDLE;
        display.update(currentFloor, direction);
    }

    private void processUpRequests() {
        while (!upQueue.isEmpty()) {
            int nextFloor = upQueue.poll();
            moveToFloor(nextFloor, Direction.UP);
        }
    }

    private void processDownRequests() {
        while (!downQueue.isEmpty()) {
            int nextFloor = downQueue.poll();
            moveToFloor(nextFloor, Direction.DOWN);
        }
    }

    private void moveToFloor(int floor, Direction dir) {
        this.direction = dir;
        this.state = (dir == Direction.UP) ? ElevatorState.MOVING_UP : ElevatorState.MOVING_DOWN;
        display.update(currentFloor, direction);

        // Simulate movement
        currentFloor = floor;
        display.update(currentFloor, direction);

        door.open();
        // Wait for passengers
        door.close();
    }

    // Getters
    public int getCurrentFloor() { return currentFloor; }
    public Direction getDirection() { return direction; }
    public ElevatorState getState() { return state; }
    public int getId() { return id; }
}
```

### Elevator Selection Strategy

```java
public interface ElevatorSelectionStrategy {
    Elevator selectElevator(List<Elevator> elevators, int requestedFloor, Direction direction);
}

// LOOK Algorithm (Nearest in same direction)
public class LookStrategy implements ElevatorSelectionStrategy {
    @Override
    public Elevator selectElevator(List<Elevator> elevators, int requestedFloor, Direction direction) {
        Elevator bestElevator = null;
        int minDistance = Integer.MAX_VALUE;

        for (Elevator elevator : elevators) {
            if (elevator.getState() == ElevatorState.MAINTENANCE) continue;

            int distance = Math.abs(elevator.getCurrentFloor() - requestedFloor);

            // Priority 1: Idle elevator
            if (elevator.getState() == ElevatorState.IDLE && distance < minDistance) {
                bestElevator = elevator;
                minDistance = distance;
            }

            // Priority 2: Moving in same direction and hasn't passed the floor
            if (elevator.getDirection() == direction) {
                boolean canServe = (direction == Direction.UP && elevator.getCurrentFloor() <= requestedFloor)
                                || (direction == Direction.DOWN && elevator.getCurrentFloor() >= requestedFloor);
                if (canServe && distance < minDistance) {
                    bestElevator = elevator;
                    minDistance = distance;
                }
            }
        }

        // Fallback: pick any available elevator with min distance
        if (bestElevator == null) {
            for (Elevator elevator : elevators) {
                if (elevator.getState() == ElevatorState.MAINTENANCE) continue;
                int distance = Math.abs(elevator.getCurrentFloor() - requestedFloor);
                if (distance < minDistance) {
                    bestElevator = elevator;
                    minDistance = distance;
                }
            }
        }

        return bestElevator;
    }
}
```

### ElevatorSystem Controller

```java
public class ElevatorSystem {
    private List<Elevator> elevators;
    private List<Floor> floors;
    private ElevatorSelectionStrategy strategy;

    public ElevatorSystem(int numElevators, int numFloors) {
        elevators = new ArrayList<>();
        floors = new ArrayList<>();
        strategy = new LookStrategy();

        for (int i = 0; i < numElevators; i++) {
            elevators.add(new Elevator(i));
        }
        for (int i = 0; i < numFloors; i++) {
            floors.add(new Floor(i));
        }
    }

    // External request: from a floor button
    public void handleExternalRequest(int floor, Direction direction) {
        Elevator elevator = strategy.selectElevator(elevators, floor, direction);
        if (elevator != null) {
            elevator.addRequest(floor);
            System.out.println("Elevator " + elevator.getId() + " assigned to floor " + floor);
        }
    }

    // Internal request: from inside the elevator
    public void handleInternalRequest(int elevatorId, int floor) {
        elevators.get(elevatorId).addRequest(floor);
    }
}
```

---

## 🧪 Usage Example

```java
ElevatorSystem system = new ElevatorSystem(3, 10);

// Person on floor 5 wants to go UP
system.handleExternalRequest(5, Direction.UP);

// Inside elevator 0, person presses floor 8
system.handleInternalRequest(0, 8);

// Person on floor 3 wants to go DOWN
system.handleExternalRequest(3, Direction.DOWN);
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Strategy** | `ElevatorSelectionStrategy` — LOOK, FCFS, Shortest Seek |
| **State** | Elevator states: IDLE, MOVING_UP, MOVING_DOWN, MAINTENANCE |
| **Observer** | Display updates when elevator moves |
| **Singleton** | Can be used for ElevatorSystem if only one building |

---

## 📊 Elevator Scheduling Algorithms

| Algorithm | Description |
|-----------|-------------|
| **FCFS** | First Come First Serve — simple but inefficient |
| **SSTF** | Shortest Seek Time First — serve nearest request |
| **SCAN** | Go in one direction, serve all, reverse (like disk arm) |
| **LOOK** | Like SCAN but reverses when no more requests in direction |
| **C-SCAN** | Circular SCAN — always go up, jump to bottom, go up again |

---

## ⚠️ Edge Cases

- All elevators on maintenance → return error
- Request for current floor → just open the door
- Multiple requests at same time → queue them
- Elevator overweight → refuse new passengers
- Fire alarm → all elevators go to ground floor
- Power failure → move to nearest floor and open doors

---

## 🔑 Key Takeaways

1. **Two Priority Queues** (min-heap for UP, max-heap for DOWN) is the key data structure
2. **Strategy Pattern** makes the scheduling algorithm pluggable
3. **LOOK algorithm** is the most commonly expected answer
4. Separate **internal vs external** requests
5. Each elevator operates **independently** with its own thread
