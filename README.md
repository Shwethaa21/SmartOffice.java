# SmartOffice
A console-based application to manage a smart office facility. The system should handle  conference room bookings, occupancy detection, and automate the control of air conditioning  and lighting based on room occupancy. 
The code for this project given below:


import java.time.*;
import java.time.format.DateTimeFormatter;
import java.time.format.DateTimeParseException;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;
public class SmartOffice {
    public interface OccupancyObserver {
        void update(Room room, boolean occupied);
    }

    public static class Light implements OccupancyObserver {
        private boolean on = false;

        @Override
        public void update(Room room, boolean occupied) {
            boolean prev = on;
            on = occupied;
            if (on && !prev) {
                System.out.println("Lights turned ON in Room " + room.getId() + ".");
            } else if (!on && prev) {
                System.out.println("Lights turned OFF in Room " + room.getId() + ".");
            }
        }

        public boolean isOn() { return on; }
    }

    public static class AC implements OccupancyObserver {
        private boolean on = false;

        @Override
        public void update(Room room, boolean occupied) {
            boolean prev = on;
            on = occupied;
            if (on && !prev) {
                System.out.println("AC turned ON in Room " + room.getId() + ".");
            } else if (!on && prev) {
                System.out.println("AC turned OFF in Room " + room.getId() + ".");
            }
        }

        public boolean isOn() { return on; }
    }
    public static class Booking {
        private final int roomId;
        private final LocalDateTime start;
        private final Duration duration;
        private final String user;
        private volatile boolean released = false;

        public Booking(int roomId, LocalDateTime start, Duration duration, String user) {
            this.roomId = roomId;
            this.start = start;
            this.duration = duration;
            this.user = user;
        }

        public boolean isActiveAt(LocalDateTime dt) {
            if (released) return false;
            return !dt.isBefore(start) && dt.isBefore(start.plus(duration));
        }

        public boolean overlaps(LocalDateTime s, LocalDateTime e) {
            if (released) return false;
            LocalDateTime a1 = this.start;
            LocalDateTime a2 = this.start.plus(this.duration);
            return !e.isBefore(a1) && !s.isAfter(a2) && !(e.equals(a1) || s.equals(a2));
        }

        public int getRoomId() { return roomId; }
        public LocalDateTime getStart() { return start; }
        public LocalDateTime getEnd() { return start.plus(duration); }
        public String getUser() { return user; }
        public void release() { released = true; }
        public boolean isReleased() { return released; }

        @Override
        public String toString() {
            return String.format("Booking(Room %d by %s from %s for %d min)",
                    roomId, user, start.format(DateTimeFormatter.ofPattern("HH:mm")),
                    duration.toMinutes());
        }
    }
    public static class NotificationCenter {
        public static void notifyUser(String user, String message) {
            System.out.println("[Notification] To " + user + ": " + message);
        }
    }

    public static class Room {
        public static final int OCCUPANCY_THRESHOLD = 2;

        private final int id;
        private int maxCapacity = 10;
        private int currentPeople = 0;
        private boolean occupied = false;
        private final List<OccupancyObserver> observers = new ArrayList<>();
        private Booking booking = null;

        private long totalOccupiedMinutes = 0;
        private LocalDateTime occupiedSince = null;

        private ScheduledFuture<?> unoccupiedReleaseTask = null;
        private ScheduledFuture<?> bookingStartCheckTask = null;

        public Room(int id) { this.id = id; }

        public int getId() { return id; }
        public int getMaxCapacity() { return maxCapacity; }
        public void setMaxCapacity(int maxCapacity) {
            if (maxCapacity <= 0) throw new IllegalArgumentException("Invalid capacity.");
            this.maxCapacity = maxCapacity;
        }

        public synchronized void attachObserver(OccupancyObserver o) {
            if (!observers.contains(o)) observers.add(o);
        }

        public synchronized void detachObserver(OccupancyObserver o) {
            observers.remove(o);
        }

        private synchronized void notifyObservers() {
            for (OccupancyObserver o : new ArrayList<>(observers)) {
                try { o.update(this, occupied); } catch (Exception ignore) {}
            }
        }

        public synchronized Booking getBooking() { return booking; }
        public synchronized void setBooking(Booking b) { booking = b; }

        public synchronized int getCurrentPeople() { return currentPeople; }

        public synchronized void addPeople(int count) {
            if (count < 0) throw new IllegalArgumentException("Count must be non-negative.");
            int prev = this.currentPeople;
            this.currentPeople = count;
            boolean prevOccupied = this.occupied;
            this.occupied = (this.currentPeople >= OCCUPANCY_THRESHOLD);

            if (this.occupied && !prevOccupied) {
                occupiedSince = LocalDateTime.now();
                System.out.println(String.format("Room %d is now occupied by %d persons. AC and lights turned on.", id, currentPeople));
                cancelBookingStartCheck();
                cancelUnoccupiedReleaseTask();
            } else if (!this.occupied && prevOccupied) {
                if (occupiedSince != null) {
                    Duration d = Duration.between(occupiedSince, LocalDateTime.now());
                    totalOccupiedMinutes += d.toMinutes();
                    occupiedSince = null;
                }
                System.out.println(String.format("Room %d is now unoccupied. AC and lights turned off.", id));
                // if booking active schedule release after 5 minutes
                Booking b = this.booking;
                if (b != null && b.isActiveAt(LocalDateTime.now())) {
                    scheduleUnoccupiedRelease(5, TimeUnit.MINUTES);
                }
            } else {
                if (!this.occupied) {
                    System.out.println(String.format("Room %d occupancy insufficient to mark as occupied.", id));
                }
            }

            notifyObservers();
        }

        public synchronized long getTotalOccupiedMinutes() { return totalOccupiedMinutes; }

        public synchronized void scheduleUnoccupiedRelease(long delay, TimeUnit unit) {
            cancelUnoccupiedReleaseTask();
            unoccupiedReleaseTask = Office.getInstance().scheduler.schedule(() -> {
                synchronized (Room.this) {
                    if (!occupied && booking != null && booking.isActiveAt(LocalDateTime.now())) {
                        booking.release();
                        booking = null;
                        System.out.println(String.format("Room %d is now unoccupied. Booking released. AC and lights off.", id));
                        NotificationCenter.notifyUser(booking.getUser(), "Your booking for Room " + id + " was released due to inactivity.");
                    }
                }
            }, delay, unit);
        }

        public synchronized void cancelUnoccupiedReleaseTask() {
            if (unoccupiedReleaseTask != null) {
                unoccupiedReleaseTask.cancel(false);
                unoccupiedReleaseTask = null;
            }
        }

        public synchronized void scheduleBookingStartCheck(Booking b) {
            cancelBookingStartCheck();
            long delaySeconds = Duration.between(LocalDateTime.now(), b.getStart().plusMinutes(5)).getSeconds();
            if (delaySeconds < 0) delaySeconds = 0;
            bookingStartCheckTask = Office.getInstance().scheduler.schedule(() -> {
                synchronized (Room.this) {
                    if (booking == b && !occupied) {
                        b.release();
                        booking = null;
                        System.out.println(String.format("Room %d booking auto-released because room not occupied within 5 minutes of booking start.", id));
                        NotificationCenter.notifyUser(b.getUser(), "Your booking for Room " + id + " was released automatically (no show).");
                    }
                }
            }, delaySeconds, TimeUnit.SECONDS);
        }

        public synchronized void cancelBookingStartCheck() {
            if (bookingStartCheckTask != null) {
                bookingStartCheckTask.cancel(false);
                bookingStartCheckTask = null;
            }
        }

        public synchronized void cancelScheduledTasks() {
            cancelUnoccupiedReleaseTask();
            cancelBookingStartCheck();
        }

    }

    public static class Office {
        private static Office instance;
        public final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(4);
        private final Map<Integer, Room> rooms = new ConcurrentHashMap<>();
        private final List<Booking> bookings = Collections.synchronizedList(new ArrayList<>());
        private final Map<String, String> users = new ConcurrentHashMap<>();
        private volatile String loggedInUser = null;
        private final AtomicInteger totalBookings = new AtomicInteger(0);

        private Office() {
 
            users.put("admin", "admin123");
        }

        public static synchronized Office getInstance() {
            if (instance == null) instance = new Office();
            return instance;
        }


        public synchronized void configureRooms(int count) {
            if (count <= 0) throw new IllegalArgumentException("Invalid room count.");
            rooms.clear();
            for (int i = 1; i <= count; ++i) {
                Room r = new Room(i);
                r.attachObserver(new Light());
                r.attachObserver(new AC());
                rooms.put(i, r);
            }
            System.out.println("Office configured with " + count + " meeting rooms: " + rooms.keySet());
        }

        public synchronized void setRoomCapacity(int roomId, int capacity) {
            Room r = rooms.get(roomId);
            if (r == null) throw new IllegalArgumentException("Invalid room number.");
            r.setMaxCapacity(capacity);
            System.out.println("Room " + roomId + " maximum capacity set to " + capacity + ".");
        }

        public Room findRoom(int roomId) { return rooms.get(roomId); }

        public synchronized Booking bookRoom(int roomId, LocalDateTime start, int durationMinutes, String user) {
            Room r = rooms.get(roomId);
            if (r == null) throw new IllegalArgumentException("Invalid room number.");
            LocalDateTime end = start.plusMinutes(durationMinutes);
            for (Booking b : bookings) {
                if (b.getRoomId() == roomId && !b.isReleased() && b.overlaps(start, end)) {
                    throw new IllegalStateException("Room already booked during this time. Cannot book.");
                }
            }
            Booking newB = new Booking(roomId, start, Duration.ofMinutes(durationMinutes), user);
            bookings.add(newB);
            r.setBooking(newB);
            r.scheduleBookingStartCheck(newB);
            totalBookings.incrementAndGet();
            System.out.println(String.format("Room %d booked from %s for %d minutes.", roomId, start.format(DateTimeFormatter.ofPattern("HH:mm")), durationMinutes));
            return newB;
        }

        public synchronized void cancelBooking(int roomId) {
            Room r = rooms.get(roomId);
            if (r == null) throw new IllegalArgumentException("Invalid room number.");
            Booking b = r.getBooking();
            if (b == null || b.isReleased()) throw new IllegalStateException("Room is not booked. Cannot cancel booking.");
            b.release();
            r.setBooking(null);
            r.cancelScheduledTasks();
            System.out.println("Booking for Room " + roomId + " cancelled successfully.");
            NotificationCenter.notifyUser(b.getUser(), "Your booking for Room " + roomId + " has been cancelled by user.");
        }

        public synchronized String usageSummary() {
            StringBuilder sb = new StringBuilder();
            List<Integer> ids = new ArrayList<>(rooms.keySet());
            Collections.sort(ids);
            for (int id : ids) {
                Room r = rooms.get(id);
                sb.append(String.format("Room %d: total occupied minutes = %d, last people = %d, max cap = %d%n",
                        id, r.getTotalOccupiedMinutes(), r.getCurrentPeople(), r.getMaxCapacity()));
            }
            sb.append("Total bookings ever made: ").append(totalBookings.get());
            return sb.toString();
        }

  
        public synchronized void registerUser(String username, String password) {
            if (users.containsKey(username)) throw new IllegalStateException("User already exists.");
            users.put(username, password);
            System.out.println("User '" + username + "' registered.");
        }

        public synchronized void login(String username, String password) {
            if (!users.containsKey(username) || !users.get(username).equals(password))
                throw new IllegalArgumentException("Invalid username or password");
            loggedInUser = username;
            System.out.println("User '" + username + "' logged in.");
        }

        public synchronized void logout() {
            if (loggedInUser == null) {
                System.out.println("No user currently logged in.");
                return;
            }
            System.out.println("User '" + loggedInUser + "' logged out.");
            loggedInUser = null;
        }

        public synchronized void requireLogin() {
            if (loggedInUser == null) throw new SecurityException("Authentication required. Please login.");
        }

        public synchronized String getLoggedInUser() { return loggedInUser; }
    }

    
    public interface Command {
        void execute(String[] args);
    }

    public static class LoginCommand implements Command {
        @Override public void execute(String[] args) {
            if (args.length < 2) { System.out.println("Usage: login <username> <password>"); return; }
            try { Office.getInstance().login(args[0], args[1]); }
            catch (Exception e) { System.out.println("Login failed: " + e.getMessage()); }
        }
    }

    public static class LogoutCommand implements Command {
        @Override public void execute(String[] args) { Office.getInstance().logout(); }
    }

    public static class RegisterCommand implements Command {
        @Override public void execute(String[] args) {
            if (args.length < 2) { System.out.println("Usage: register <username> <password>"); return; }
            try { Office.getInstance().registerUser(args[0], args[1]); }
            catch (Exception e) { System.out.println("Registration failed: " + e.getMessage()); }
        }
    }

    public static class ConfigRoomsCommand implements Command {
        @Override public void execute(String[] args) {
            try {
                Office.getInstance().requireLogin();
                if (args.length != 1) { System.out.println("Usage: config room count <n>"); return; }
                int n = Integer.parseInt(args[0]);
                Office.getInstance().configureRooms(n);
            } catch (SecurityException se) { System.out.println(se.getMessage()); }
            catch (Exception e) { System.out.println("Error: " + e.getMessage()); }
        }
    }

    public static class SetCapacityCommand implements Command {
        @Override public void execute(String[] args) {
            try {
                Office.getInstance().requireLogin();
                if (args.length != 2) { System.out.println("Usage: config room max capacity <id> <cap>"); return; }
                int id = Integer.parseInt(args[0]);
                int cap = Integer.parseInt(args[1]);
                Office.getInstance().setRoomCapacity(id, cap);
            } catch (SecurityException se) { System.out.println(se.getMessage()); }
            catch (Exception e) { System.out.println("Error: " + e.getMessage()); }
        }
    }

    public static class AddOccupantCommand implements Command {
        @Override public void execute(String[] args) {
            if (args.length != 2) { System.out.println("Usage: add occupant <room_id> <count>"); return; }
            try {
                int roomId = Integer.parseInt(args[0]);
                int count = Integer.parseInt(args[1]);
                Room r = Office.getInstance().findRoom(roomId);
                if (r == null) { System.out.println("Room " + roomId + " does not exist."); return; }
                r.addPeople(count);
            } catch (Exception e) { System.out.println("Error: " + e.getMessage()); }
        }
    }

    public static class BlockRoomCommand implements Command {
        private static final DateTimeFormatter TF = DateTimeFormatter.ofPattern("H:mm");
        @Override public void execute(String[] args) {
            try {
                Office.getInstance().requireLogin();
                if (args.length != 3) { System.out.println("Usage: block room <room_id> <HH:MM> <duration_minutes>"); return; }
                int roomId = Integer.parseInt(args[0]);
                String timeStr = args[1];
                int duration = Integer.parseInt(args[2]);
                LocalTime t;
                try { t = LocalTime.parse(timeStr, TF); }
                catch (DateTimeParseException dte) { System.out.println("Invalid time format. Use HH:MM"); return; }
                LocalDate today = LocalDate.now();
                LocalDateTime start = LocalDateTime.of(today, t);
                // if start is earlier than now assume today in past — still allowed; checks happen on schedule
                Office.getInstance().bookRoom(roomId, start, duration, Office.getInstance().getLoggedInUser());
            } catch (SecurityException se) { System.out.println(se.getMessage()); }
            catch (Exception e) { System.out.println("Error: " + e.getMessage()); }
        }
    }

    public static class CancelRoomCommand implements Command {
        @Override public void execute(String[] args) {
            try {
                Office.getInstance().requireLogin();
                if (args.length != 1) { System.out.println("Usage: cancel room <room_id>"); return; }
                int roomId = Integer.parseInt(args[0]);
                Office.getInstance().cancelBooking(roomId);
            } catch (SecurityException se) { System.out.println(se.getMessage()); }
            catch (Exception e) { System.out.println("Error: " + e.getMessage()); }
        }
    }

    public static class RoomStatusCommand implements Command {
        @Override public void execute(String[] args) {
            if (args.length != 1) { System.out.println("Usage: room status <room_id>"); return; }
            try {
                int id = Integer.parseInt(args[0]);
                Room r = Office.getInstance().findRoom(id);
                if (r == null) { System.out.println("Room " + id + " does not exist."); return; }
                String status = r.occupied ? "occupied" : "unoccupied";
                Booking b = r.getBooking();
                String bookingInfo = (b != null && !b.isReleased()) ? b.toString() : "No active booking";
                System.out.println(String.format("Room %d status: %s. People: %d. Booking: %s", id, status, r.getCurrentPeople(), bookingInfo));
            } catch (Exception e) { System.out.println("Error: " + e.getMessage()); }
        }
    }

    public static class ShowStatsCommand implements Command {
        @Override public void execute(String[] args) {
            System.out.println("Usage summary:");
            System.out.println(Office.getInstance().usageSummary());
        }
    }

    public static class ConsoleApp {
        private final Map<String, Command> simpleCommands = new HashMap<>();
        private final Map<String, Command> unaryCommands = new HashMap<>();

        public ConsoleApp() {
            simpleCommands.put("login", new LoginCommand());
            simpleCommands.put("logout", new LogoutCommand());
            simpleCommands.put("register", new RegisterCommand());
           
            simpleCommands.put("help", args -> printHelp());
            simpleCommands.put("exit", args -> { System.out.println("Exiting."); System.exit(0); });
            simpleCommands.put("quit", args -> { System.out.println("Exiting."); System.exit(0); });
        }

        public void run() {
            System.out.println("Smart Office Console App (Java)");
            System.out.println("Note: default admin user: username='admin', password='admin123'");
            System.out.println("Type 'help' for commands.\n");
            Scanner sc = new Scanner(System.in);
            while (true) {
                System.out.print("> ");
                String line;
                try {
                    line = sc.nextLine().trim();
                } catch (NoSuchElementException e) {
                    System.out.println("\nGoodbye.");
                    break;
                }
                if (line.isEmpty()) continue;
                handle(line);
            }
            sc.close();
        }

        private void handle(String raw) {
            String[] tokens = splitKeepingQuotes(raw);
            String cmd = tokens[0].toLowerCase();
            try {
                switch (cmd) {
                    case "login": simpleCommands.get("login").execute(Arrays.copyOfRange(tokens, 1, tokens.length)); break;
                    case "logout": simpleCommands.get("logout").execute(new String[]{}); break;
                    case "register": simpleCommands.get("register").execute(Arrays.copyOfRange(tokens, 1, tokens.length)); break;
                    case "config": handleConfig(Arrays.copyOfRange(tokens, 1, tokens.length)); break;
                    case "add": handleAdd(Arrays.copyOfRange(tokens, 1, tokens.length)); break;
                    case "block": handleBlock(Arrays.copyOfRange(tokens, 1, tokens.length)); break;
                    case "cancel": handleCancel(Arrays.copyOfRange(tokens, 1, tokens.length)); break;
                    case "room": handleRoom(Arrays.copyOfRange(tokens, 1, tokens.length)); break;
                    case "show": handleShow(Arrays.copyOfRange(tokens, 1, tokens.length)); break;
                    case "help": printHelp(); break;
                    case "exit": System.exit(0); break;
                    case "quit": System.exit(0); break;
                    default:
                        System.out.println("Unknown command. Type 'help' for available commands.");
                }
            } catch (Exception e) {
                System.out.println("Error: " + e.getMessage());
            }
        }

        private void handleConfig(String[] args) {
            if (args.length == 0) { System.out.println("Usage: config room ..."); return; }
            if (!args[0].equals("room")) { System.out.println("Usage: config room ..."); return; }
            if (args.length >= 2 && args[1].equals("count") && args.length == 3) {
                new ConfigRoomsCommand().execute(new String[]{args[2]});
                return;
            }
            if (args.length == 5 && args[1].equals("max") && args[2].equals("capacity")) {
                new SetCapacityCommand().execute(new String[]{args[3], args[4]});
                return;
            }
            System.out.println("Invalid config command. Examples:\n  config room count 3\n  config room max capacity 1 10");
        }

        private void handleAdd(String[] args) {
            if (args.length == 0 || !args[0].equals("occupant") || args.length != 3) {
                System.out.println("Usage: add occupant <room_id> <count>");
                return;
            }
            new AddOccupantCommand().execute(new String[]{args[1], args[2]});
        }

        private void handleBlock(String[] args) {
            if (args.length == 0 || !args[0].equals("room") || args.length != 4) {
                System.out.println("Usage: block room <room_id> <HH:MM> <duration_minutes>");
                return;
            }
            new BlockRoomCommand().execute(new String[]{args[1], args[2], args[3]});
        }

        private void handleCancel(String[] args) {
            if (args.length == 0 || !args[0].equals("room") || args.length != 2) {
                System.out.println("Usage: cancel room <room_id>");
                return;
            }
            new CancelRoomCommand().execute(new String[]{args[1]});
        }

        private void handleRoom(String[] args) {
            if (args.length == 0 || !args[0].equals("status") || args.length != 2) {
                System.out.println("Usage: room status <room_id>");
                return;
            }
            new RoomStatusCommand().execute(new String[]{args[1]});
        }

        private void handleShow(String[] args) {
            if (args.length == 0 || !args[0].equals("stats")) {
                System.out.println("Usage: show stats");
                return;
            }
            new ShowStatsCommand().execute(new String[]{});
        }

        private void printHelp() {
            System.out.println("Available commands:");
            System.out.println("  login <username> <password>");
            System.out.println("  logout");
            System.out.println("  register <username> <password>");
            System.out.println("  config room count <n>");
            System.out.println("  config room max capacity <id> <cap>");
            System.out.println("  block room <room_id> <HH:MM> <min>");
            System.out.println("  cancel room <room_id>");
            System.out.println("  add occupant <room_id> <count>");
            System.out.println("  room status <room_id>");
            System.out.println("  show stats");
            System.out.println("  help");
            System.out.println("  exit | quit");
            System.out.println("\nNotes:");
            System.out.println("- Rooms are considered \"occupied\" when the occupant count >= 2.");
            System.out.println("- If a booked room isn't occupied within 5 minutes of the booking start time, the booking is released automatically.");
            System.out.println("- If a booked, active room becomes unoccupied for 5 minutes while booking is active, booking is released.");
        }

        // basic tokenizer (handles quoted tokens if needed)
        private String[] splitKeepingQuotes(String input) {
            List<String> tokens = new ArrayList<>();
            StringBuilder sb = new StringBuilder();
            boolean inQuote = false;
            for (int i = 0; i < input.length(); ++i) {
                char c = input.charAt(i);
                if (c == '"') { inQuote = !inQuote; continue; }
                if (Character.isWhitespace(c) && !inQuote) {
                    if (sb.length() > 0) { tokens.add(sb.toString()); sb.setLength(0); }
                } else sb.append(c);
            }
            if (sb.length() > 0) tokens.add(sb.toString());
            return tokens.toArray(new String[0]);
        }

    }

    public static void main(String[] args) {
        ConsoleApp app = new ConsoleApp();
        app.run();
    }
}
