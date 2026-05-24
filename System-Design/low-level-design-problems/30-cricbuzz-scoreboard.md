# 30 - Cricbuzz / Sports Scoreboard

## 📋 Problem Statement

Design a live cricket scoreboard system like Cricbuzz that:
- Tracks live match scores ball-by-ball
- Maintains player and team statistics
- Supports different match formats (T20, ODI, Test)

---

## 📌 Requirements

### Functional Requirements
1. **Create match** with two teams, toss, and format
2. **Ball-by-ball** score update (runs, wicket, extras)
3. **Live scorecard** — batting and bowling stats
4. **Innings management** — switch innings, follow-on
5. **Match result** — Win, Loss, Draw, Tie
6. **Player stats** — runs, balls, 4s, 6s, strike rate, bowling figures
7. **Commentary** for each ball

### Non-Functional Requirements
- Real-time updates
- Accurate statistics calculation

---

## 🧩 Core Entities

```
Match, Team, Player, Innings, Over, Ball, 
Scorecard, BattingStats, BowlingStats, MatchType
```

---

## 🔧 Enums

```java
public enum MatchType {
    T20(20), ODI(50), TEST(0); // TEST has no fixed over limit per innings

    private final int maxOvers;
    MatchType(int maxOvers) { this.maxOvers = maxOvers; }
    public int getMaxOvers() { return maxOvers; }
}

public enum MatchResult {
    IN_PROGRESS, TEAM1_WON, TEAM2_WON, DRAW, TIE
}

public enum BallType {
    NORMAL, WIDE, NO_BALL, BYE, LEG_BYE
}

public enum WicketType {
    NONE, BOWLED, CAUGHT, LBW, RUN_OUT, STUMPED,
    HIT_WICKET, CAUGHT_BEHIND, RETIRED_HURT
}
```

---

## 💻 Code Implementation

### Player and Team

```java
public class Player {
    private String playerId;
    private String name;
    private boolean isKeeper;
    private boolean isCaptain;

    public Player(String id, String name) {
        this.playerId = id;
        this.name = name;
    }

    public String getName() { return name; }
    public String getPlayerId() { return playerId; }
    public void setKeeper(boolean keeper) { this.isKeeper = keeper; }
    public void setCaptain(boolean captain) { this.isCaptain = captain; }
}

public class Team {
    private String name;
    private List<Player> players;

    public Team(String name, List<Player> players) {
        this.name = name;
        this.players = new ArrayList<>(players);
    }

    public String getName() { return name; }
    public List<Player> getPlayers() { return players; }
}
```

### Ball (Single Delivery)

```java
public class Ball {
    private int overNumber;
    private int ballNumber;
    private Player bowler;
    private Player batsman;
    private int runsScored;
    private BallType type;
    private WicketType wicket;
    private Player fielder; // who took the catch / did the run-out
    private String commentary;

    public Ball(int overNumber, int ballNumber, Player bowler, Player batsman) {
        this.overNumber = overNumber;
        this.ballNumber = ballNumber;
        this.bowler = bowler;
        this.batsman = batsman;
        this.type = BallType.NORMAL;
        this.wicket = WicketType.NONE;
    }

    public void setResult(int runs, BallType type) {
        this.runsScored = runs;
        this.type = type;
    }

    public void setWicket(WicketType wicket, Player fielder) {
        this.wicket = wicket;
        this.fielder = fielder;
    }

    public void setCommentary(String commentary) {
        this.commentary = commentary;
    }

    public boolean isLegalDelivery() {
        return type != BallType.WIDE && type != BallType.NO_BALL;
    }

    public boolean isWicket() { return wicket != WicketType.NONE; }

    // Getters
    public int getRunsScored() { return runsScored; }
    public BallType getType() { return type; }
    public WicketType getWicket() { return wicket; }
    public Player getBowler() { return bowler; }
    public Player getBatsman() { return batsman; }
}
```

### Over Class

```java
public class Over {
    private int overNumber;
    private Player bowler;
    private List<Ball> balls;

    public Over(int overNumber, Player bowler) {
        this.overNumber = overNumber;
        this.bowler = bowler;
        this.balls = new ArrayList<>();
    }

    public void addBall(Ball ball) {
        balls.add(ball);
    }

    public boolean isComplete() {
        long legalBalls = balls.stream().filter(Ball::isLegalDelivery).count();
        return legalBalls >= 6;
    }

    public int getLegalBallCount() {
        return (int) balls.stream().filter(Ball::isLegalDelivery).count();
    }

    public int getTotalRuns() {
        return balls.stream().mapToInt(Ball::getRunsScored).sum();
    }

    public int getWickets() {
        return (int) balls.stream().filter(Ball::isWicket).count();
    }

    public Player getBowler() { return bowler; }
    public List<Ball> getBalls() { return balls; }
}
```

### BattingStats and BowlingStats

```java
public class BattingStats {
    private Player player;
    private int runs;
    private int ballsFaced;
    private int fours;
    private int sixes;
    private WicketType howOut;
    private Player dismissedBy;
    private boolean isNotOut;

    public BattingStats(Player player) {
        this.player = player;
        this.isNotOut = true;
    }

    public void addRuns(int runs, boolean isBoundary) {
        this.runs += runs;
        this.ballsFaced++;
        if (isBoundary) {
            if (runs == 4) fours++;
            else if (runs == 6) sixes++;
        }
    }

    public void dotBall() {
        this.ballsFaced++;
    }

    public void markOut(WicketType type, Player dismissedBy) {
        this.howOut = type;
        this.dismissedBy = dismissedBy;
        this.isNotOut = false;
    }

    public double getStrikeRate() {
        return ballsFaced == 0 ? 0 : (runs * 100.0) / ballsFaced;
    }

    @Override
    public String toString() {
        String status = isNotOut ? "not out" : howOut.name().toLowerCase();
        return String.format("%-20s %s  %d (%d)  4s:%d  6s:%d  SR:%.1f",
            player.getName(), status, runs, ballsFaced, fours, sixes, getStrikeRate());
    }

    public Player getPlayer() { return player; }
    public int getRuns() { return runs; }
}

public class BowlingStats {
    private Player player;
    private int oversBowled;
    private int ballsBowled; // legal deliveries in current over
    private int runsConceded;
    private int wickets;
    private int maidens;
    private int wides;
    private int noBalls;

    public BowlingStats(Player player) {
        this.player = player;
    }

    public void addBall(Ball ball) {
        if (ball.isLegalDelivery()) {
            ballsBowled++;
            if (ballsBowled == 6) {
                oversBowled++;
                ballsBowled = 0;
            }
        } else {
            if (ball.getType() == BallType.WIDE) wides++;
            else if (ball.getType() == BallType.NO_BALL) noBalls++;
        }
        runsConceded += ball.getRunsScored();
        if (ball.isWicket()) wickets++;
    }

    public double getEconomy() {
        double totalOvers = oversBowled + (ballsBowled / 6.0);
        return totalOvers == 0 ? 0 : runsConceded / totalOvers;
    }

    public String getOversString() {
        return oversBowled + "." + ballsBowled;
    }

    @Override
    public String toString() {
        return String.format("%-20s %s  %d/%d  Econ:%.1f",
            player.getName(), getOversString(), wickets, runsConceded, getEconomy());
    }

    public Player getPlayer() { return player; }
}
```

### Innings Class

```java
public class Innings {
    private Team battingTeam;
    private Team bowlingTeam;
    private int inningsNumber;
    private List<Over> overs;
    private Over currentOver;
    private int totalRuns;
    private int totalWickets;
    private int extras;
    private Map<String, BattingStats> battingCard;
    private Map<String, BowlingStats> bowlingCard;
    private Player striker;
    private Player nonStriker;
    private boolean isComplete;

    public Innings(Team batting, Team bowling, int inningsNumber) {
        this.battingTeam = batting;
        this.bowlingTeam = bowling;
        this.inningsNumber = inningsNumber;
        this.overs = new ArrayList<>();
        this.battingCard = new LinkedHashMap<>();
        this.bowlingCard = new LinkedHashMap<>();
        this.isComplete = false;
    }

    public void startInnings(Player striker, Player nonStriker) {
        this.striker = striker;
        this.nonStriker = nonStriker;
        battingCard.put(striker.getPlayerId(), new BattingStats(striker));
        battingCard.put(nonStriker.getPlayerId(), new BattingStats(nonStriker));
    }

    public void startOver(Player bowler) {
        int overNum = overs.size();
        currentOver = new Over(overNum, bowler);
        overs.add(currentOver);
        bowlingCard.putIfAbsent(bowler.getPlayerId(), new BowlingStats(bowler));
    }

    public void recordBall(int runs, BallType type, WicketType wicket,
                            Player fielder, String commentary) {
        Ball ball = new Ball(overs.size() - 1, currentOver.getLegalBallCount() + 1,
            currentOver.getBowler(), striker);
        ball.setResult(runs, type);
        ball.setCommentary(commentary);

        if (wicket != WicketType.NONE) {
            ball.setWicket(wicket, fielder);
        }

        currentOver.addBall(ball);

        // Update stats
        BattingStats batStats = battingCard.get(striker.getPlayerId());
        BowlingStats bowlStats = bowlingCard.get(currentOver.getBowler().getPlayerId());

        if (type == BallType.NORMAL) {
            batStats.addRuns(runs, runs == 4 || runs == 6);
            totalRuns += runs;
        } else {
            extras += (type == BallType.WIDE || type == BallType.NO_BALL) ? runs : 0;
            totalRuns += runs;
        }

        bowlStats.addBall(ball);

        // Handle wicket
        if (wicket != WicketType.NONE) {
            batStats.markOut(wicket, fielder);
            totalWickets++;
        }

        // Rotate strike on odd runs
        if (runs % 2 != 0 && type == BallType.NORMAL) {
            rotateStrike();
        }

        // End of over → rotate strike
        if (currentOver.isComplete()) {
            rotateStrike();
        }
    }

    private void rotateStrike() {
        Player temp = striker;
        striker = nonStriker;
        nonStriker = temp;
    }

    public void setNewBatsman(Player batsman) {
        this.striker = batsman;
        battingCard.put(batsman.getPlayerId(), new BattingStats(batsman));
    }

    public void printScorecard() {
        System.out.println("\n═══════════════════════════════════════════");
        System.out.println("  " + battingTeam.getName() + " INNINGS");
        System.out.println("═══════════════════════════════════════════");
        System.out.println("  BATTING:");
        battingCard.values().forEach(s -> System.out.println("  " + s));
        System.out.println("\n  BOWLING:");
        bowlingCard.values().forEach(s -> System.out.println("  " + s));
        System.out.printf("\n  TOTAL: %d/%d (%d.%d overs)%n",
            totalRuns, totalWickets, overs.size() - 1,
            currentOver != null ? currentOver.getLegalBallCount() : 0);
        System.out.println("═══════════════════════════════════════════");
    }

    public int getTotalRuns() { return totalRuns; }
    public int getTotalWickets() { return totalWickets; }
    public boolean isComplete() { return isComplete; }
    public void setComplete(boolean complete) { isComplete = complete; }
}
```

### Match Class

```java
public class Match {
    private String matchId;
    private Team team1;
    private Team team2;
    private MatchType type;
    private LocalDateTime startTime;
    private List<Innings> inningsList;
    private Innings currentInnings;
    private MatchResult result;
    private Team tossWinner;
    private String venue;

    public Match(Team team1, Team team2, MatchType type, String venue) {
        this.matchId = UUID.randomUUID().toString();
        this.team1 = team1;
        this.team2 = team2;
        this.type = type;
        this.venue = venue;
        this.startTime = LocalDateTime.now();
        this.inningsList = new ArrayList<>();
        this.result = MatchResult.IN_PROGRESS;
    }

    public void toss(Team winner) {
        this.tossWinner = winner;
    }

    public Innings startInnings(Team batting, Team bowling) {
        int inningsNum = inningsList.size() + 1;
        Innings innings = new Innings(batting, bowling, inningsNum);
        inningsList.add(innings);
        currentInnings = innings;
        return innings;
    }

    public void endMatch(MatchResult result) {
        this.result = result;
        if (currentInnings != null) {
            currentInnings.setComplete(true);
        }
    }

    public String getMatchSummary() {
        StringBuilder sb = new StringBuilder();
        sb.append(team1.getName()).append(" vs ").append(team2.getName());
        sb.append(" (").append(type).append(")\n");
        sb.append("Venue: ").append(venue).append("\n");
        sb.append("Status: ").append(result).append("\n");
        for (Innings innings : inningsList) {
            sb.append(String.format("  Innings %d: %d/%d%n",
                inningsList.indexOf(innings) + 1,
                innings.getTotalRuns(), innings.getTotalWickets()));
        }
        return sb.toString();
    }

    public Innings getCurrentInnings() { return currentInnings; }
    public List<Innings> getInningsList() { return inningsList; }
}
```

---

## 🧪 Usage Example

```java
// Create teams
List<Player> indiaPlayers = List.of(
    new Player("P1", "Rohit Sharma"),
    new Player("P2", "Virat Kohli"),
    new Player("P3", "Jasprit Bumrah")
    // ... 11 players
);
Team india = new Team("India", indiaPlayers);
Team australia = new Team("Australia", ausPlayers);

// Create match
Match match = new Match(india, australia, MatchType.T20, "Melbourne");
match.toss(india);

// First innings
Innings innings1 = match.startInnings(india, australia);
innings1.startInnings(indiaPlayers.get(0), indiaPlayers.get(1)); // openers

innings1.startOver(ausPlayers.get(9)); // bowler
innings1.recordBall(4, BallType.NORMAL, WicketType.NONE, null, "Rohit drives through covers!");
innings1.recordBall(0, BallType.NORMAL, WicketType.NONE, null, "Dot ball");
innings1.recordBall(6, BallType.NORMAL, WicketType.NONE, null, "SIX! Over midwicket!");
innings1.recordBall(1, BallType.NORMAL, WicketType.NONE, null, "Single to mid-on");
innings1.recordBall(0, BallType.NORMAL, WicketType.BOWLED, ausPlayers.get(9), "BOWLED!");
// Wicket! New batsman
innings1.setNewBatsman(indiaPlayers.get(2));
innings1.recordBall(2, BallType.NORMAL, WicketType.NONE, null, "Two runs");

innings1.printScorecard();
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Composite** | Match → Innings → Over → Ball hierarchy |
| **Observer** | Live score updates, push notifications |
| **Strategy** | Different match rules for T20/ODI/Test |
| **Builder** | Ball construction with optional wicket, commentary |

---

## ⚠️ Edge Cases

- All out before max overs → end innings
- Wide/No-ball → extra run + ball doesn't count
- Run-out on non-striker end → correct batsman marked out
- Super Over / tiebreaker in T20/ODI
- Follow-on in Test matches
- Rain interruption — DLS method

---

## 🔑 Key Takeaways

1. **Ball → Over → Innings → Match** — clean hierarchical composition
2. **BattingStats & BowlingStats** — separate stat trackers per player per innings
3. **Strike rotation** — swap on odd runs and at end of over
4. **Legal delivery tracking** — wides/no-balls don't count towards over completion
5. **Scorecard** is a derived view built from ball-by-ball data
