import javax.swing.*;
import javax.swing.border.*;
import javax.swing.table.*;
import java.awt.*;
import java.awt.event.*;
import java.time.*;
import java.time.format.*;
import java.util.*;
import java.util.List;
import java.util.function.*;

/*
 * Student Consultation Appointment and Record Management System
 * OOP 101 - Batangas State University (TEAMBOOGSH)
 *
 * HOW TO RUN (VS Code or terminal):
 *   javac Main.java
 *   java Main
 *
 * DEMO ACCOUNTS (password for all: pass1234)
 *   Student    25-01234  Juan D. Dela Cruz      (adviser: Prof. Maria L. Santos)
 *   Student    25-01310  Ana B. Garcia          (adviser: Prof. Maria L. Santos)
 *   Adviser    A-001     Prof. Maria L. Santos
 *   Instructor I-001     Ms. Fatima Marie P. Agdon
 *   Instructor I-002     Mr. Carlo R. Reyes
 */
public class Main {
    public static void main(String[] args) {
        UIManager.put("swing.boldMetal", Boolean.FALSE);
        SwingUtilities.invokeLater(() -> {
            AppFrame.applyDefaultFont(new Font("SansSerif", Font.PLAIN, 13));
            new AppFrame(new DataStore()).setVisible(true);
        });
    }
}

/* =====================================================================
 *  EXCEPTION + ENUM + FORMAT HELPERS
 * ===================================================================== */

class AppException extends Exception {
    AppException(String message) {
        super(message);
    }
}

enum Status {
    PENDING, FORWARDED, APPROVED, COMPLETED, DECLINED, CANCELLED;

    boolean isActive() {
        return this == PENDING || this == FORWARDED || this == APPROVED;
    }
}

final class Fmt {
    static final String DASH = "\u2013";
    static final String ARROW = " \u203A ";
    private static final Locale LOC = Locale.ENGLISH;

    static final DateTimeFormatter DATE_OUT = DateTimeFormatter.ofPattern("MMM d, yyyy", LOC);
    static final DateTimeFormatter TIME_OUT = DateTimeFormatter.ofPattern("h:mm a", LOC);
    private static final DateTimeFormatter DATE_IN = new DateTimeFormatterBuilder()
            .parseCaseInsensitive().appendPattern("MMM d, yyyy").toFormatter(LOC);
    private static final DateTimeFormatter TIME_IN = new DateTimeFormatterBuilder()
            .parseCaseInsensitive().appendPattern("h:mm a").toFormatter(LOC);

    private Fmt() {
    }

    static LocalDate parseDate(String text) throws AppException {
        try {
            return LocalDate.parse(text.trim().replaceAll("\\s+", " "), DATE_IN);
        } catch (DateTimeParseException e) {
            throw new AppException("Invalid date. Please use a format like: Oct 21, 2026");
        }
    }

    static LocalTime parseTime(String text) throws AppException {
        try {
            return LocalTime.parse(text.trim().replaceAll("\\s+", " ").toUpperCase(), TIME_IN);
        } catch (DateTimeParseException e) {
            throw new AppException("Invalid time. Please use a format like: 10:00 AM");
        }
    }

    /** Parses "10:00 AM - 11:00 AM" (hyphen or en dash) into {start, end}. */
    static LocalTime[] parseRange(String text) throws AppException {
        String[] parts = text.trim().split("\\s*[-\u2013\u2014]\\s*|\\s+to\\s+");
        if (parts.length != 2) {
            throw new AppException("Invalid time slot. Please use a format like: 10:00 AM - 11:00 AM");
        }
        return new LocalTime[]{parseTime(parts[0]), parseTime(parts[1])};
    }
}

/* =====================================================================
 *  MODEL CLASSES (Encapsulation, Inheritance, Abstraction, Polymorphism)
 * ===================================================================== */

/** Abstract base class: shared user information and behavior. */
abstract class User {
    private final String userId;
    private final String name;
    private final String password;
    private final String role;

    User(String userId, String name, String password, String role) {
        this.userId = userId;
        this.name = name;
        this.password = password;
        this.role = role;
    }

    String getUserId() {
        return userId;
    }

    String getName() {
        return name;
    }

    String getRole() {
        return role;
    }

    boolean login(String id, String pw) {
        return id != null && pw != null && userId.equalsIgnoreCase(id.trim()) && password.equals(pw);
    }

    String logout() {
        return name + " has logged out.";
    }

    /** Polymorphism: every role returns its own menu {label, description}. */
    abstract String[][] displayMenu();

    @Override
    public String toString() {
        return name;
    }
}

class Student extends User {
    private final String program;
    private final String adviserId;

    Student(String id, String name, String pw, String program, String adviserId) {
        super(id, name, pw, "Student");
        this.program = program;
        this.adviserId = adviserId;
    }

    String getStudentId() {
        return getUserId();
    }

    String getProgram() {
        return program;
    }

    String getAdviserId() {
        return adviserId;
    }

    Appointment bookAppointment(DataStore ds, Slot slot, String purpose) throws AppException {
        return ds.book(this, slot, purpose);
    }

    List<Appointment> checkStatus(DataStore ds) {
        List<Appointment> out = new ArrayList<>();
        for (Appointment a : ds.getAppointments()) {
            if (a.getStudentId().equals(getStudentId())) {
                out.add(a);
            }
        }
        return out;
    }

    void cancelAppointment(Appointment a) throws AppException {
        if (a == null || !a.getStudentId().equals(getStudentId())) {
            throw new AppException("Please select one of your own appointments.");
        }
        if (!a.getStatus().isActive()) {
            throw new AppException("Only requests that are not yet completed can be cancelled.");
        }
        a.setStatus(Status.CANCELLED);
    }

    List<Appointment> viewRecords(DataStore ds) {
        List<Appointment> out = new ArrayList<>();
        for (Appointment a : checkStatus(ds)) {
            if (a.getStatus() == Status.COMPLETED) {
                out.add(a);
            }
        }
        return out;
    }

    @Override
    String[][] displayMenu() {
        return new String[][]{
                {"Book Appointment", "Request a consultation with an instructor"},
                {"Check Appointment Status", "See if a request is pending, forwarded, approved, etc."},
                {"Cancel Appointment", "Cancel a request that is not yet completed"},
                {"View Consultation Records", "See past completed consultations"},
                {"Logout", "Return to the login screen"}
        };
    }
}

class Adviser extends User {
    private final String advisedSection;

    Adviser(String id, String name, String pw, String advisedSection) {
        super(id, name, pw, "Adviser");
        this.advisedSection = advisedSection;
    }

    String getAdviserId() {
        return getUserId();
    }

    String getAdvisedSection() {
        return advisedSection;
    }

    /** Pending requests that came from this adviser's advisees. */
    List<Appointment> reviewRequest(DataStore ds) {
        List<Appointment> out = new ArrayList<>();
        for (Appointment a : monitorAdvisees(ds)) {
            if (a.getStatus() == Status.PENDING) {
                out.add(a);
            }
        }
        return out;
    }

    void forwardRequest(Appointment a) throws AppException {
        if (a == null || a.getStatus() != Status.PENDING) {
            throw new AppException("Only PENDING requests can be forwarded.");
        }
        a.setForwardedBy(getName());
        a.setStatus(Status.FORWARDED);
    }

    void declineRequest(Appointment a) throws AppException {
        if (a == null || a.getStatus() != Status.PENDING) {
            throw new AppException("Only PENDING requests can be declined.");
        }
        a.setStatus(Status.DECLINED);
    }

    /** Every consultation (any status) that belongs to this adviser's advisees. */
    List<Appointment> monitorAdvisees(DataStore ds) {
        List<Appointment> out = new ArrayList<>();
        for (Appointment a : ds.getAppointments()) {
            User u = ds.findUser(a.getStudentId());
            if (u instanceof Student && ((Student) u).getAdviserId().equals(getAdviserId())) {
                out.add(a);
            }
        }
        return out;
    }

    @Override
    String[][] displayMenu() {
        return new String[][]{
                {"Review Student Requests", "See pending requests from advisees"},
                {"Forward Request to Instructor", "Send a reviewed request to the concerned instructor"},
                {"Monitor Advisees' Consultations", "View status and records of advisees' consultations"},
                {"Logout", "Return to the login screen"}
        };
    }
}

class Instructor extends User {
    private final String subject;

    Instructor(String id, String name, String pw, String subject) {
        super(id, name, pw, "Instructor");
        this.subject = subject;
    }

    String getInstructorId() {
        return getUserId();
    }

    String getSubject() {
        return subject;
    }

    Slot setSchedule(DataStore ds, LocalDate date, LocalTime start, LocalTime end) throws AppException {
        return ds.addSlot(getInstructorId(), date, start, end);
    }

    void removeSlot(DataStore ds, Slot slot) throws AppException {
        ds.removeSlot(slot);
    }

    List<Appointment> getByStatus(DataStore ds, Status status) {
        List<Appointment> out = new ArrayList<>();
        for (Appointment a : ds.getAppointments()) {
            if (a.getInstructorId().equals(getInstructorId()) && a.getStatus() == status) {
                out.add(a);
            }
        }
        return out;
    }

    /** action: "approve", "decline" or "reschedule" (newSlot is only used for reschedule). */
    void manageAppointment(DataStore ds, Appointment a, String action, Slot newSlot) throws AppException {
        if (a == null || a.getStatus() != Status.FORWARDED) {
            throw new AppException("Only FORWARDED requests can be managed.");
        }
        if (action.equals("approve")) {
            a.setStatus(Status.APPROVED);
        } else if (action.equals("decline")) {
            a.setStatus(Status.DECLINED);
        } else if (action.equals("reschedule")) {
            if (newSlot == null) {
                throw new AppException("Please choose a new open slot.");
            }
            if (newSlot == a.getSlot() || ds.isTaken(newSlot)) {
                throw new AppException("This slot is already taken.");
            }
            a.setSlot(newSlot);
            a.setStatus(Status.APPROVED);
        } else {
            throw new AppException("Invalid choice.");
        }
    }

    void recordConsultation(Appointment a, String notes) throws AppException {
        if (a == null) {
            throw new AppException("Please select an approved appointment.");
        }
        if (a.getStatus() != Status.APPROVED) {
            throw new AppException("Only APPROVED appointments can be recorded as completed.");
        }
        if (notes == null || notes.trim().isEmpty()) {
            throw new AppException("Please enter the consultation notes.");
        }
        a.setNotes(notes.trim());
        a.setStatus(Status.COMPLETED);
    }

    @Override
    String[][] displayMenu() {
        return new String[][]{
                {"Set Available Schedule", "Add or remove open consultation slots"},
                {"Manage Forwarded Requests", "Approve, reschedule, or decline requests"},
                {"Record Completed Consultation", "Save notes of a finished consultation"},
                {"View Consultation Records", "Search and review past consultations"},
                {"Logout", "Return to the login screen"}
        };
    }
}

/** An open consultation slot of one instructor. */
class Slot {
    private final String instructorId;
    private final LocalDate date;
    private final LocalTime start;
    private final LocalTime end;

    Slot(String instructorId, LocalDate date, LocalTime start, LocalTime end) {
        this.instructorId = instructorId;
        this.date = date;
        this.start = start;
        this.end = end;
    }

    String getInstructorId() {
        return instructorId;
    }

    LocalDate getDate() {
        return date;
    }

    LocalTime getStart() {
        return start;
    }

    LocalTime getEnd() {
        return end;
    }

    boolean overlaps(Slot o) {
        return instructorId.equals(o.instructorId) && date.equals(o.date)
                && start.isBefore(o.end) && o.start.isBefore(end);
    }

    String getDateTimeLabel() {
        return date.format(Fmt.DATE_OUT) + " " + Fmt.DASH + " " + start.format(Fmt.TIME_OUT);
    }

    String getRangeLabel() {
        return start.format(Fmt.TIME_OUT) + " " + Fmt.DASH + " " + end.format(Fmt.TIME_OUT);
    }
}

/** One appointment request / consultation record. */
class Appointment {
    private final String studentId;
    private final String instructorId;
    private Slot slot;
    private final String purpose;
    private Status status = Status.PENDING;
    private String notes = "";
    private String forwardedBy = "";

    Appointment(String studentId, Slot slot, String purpose) {
        this.studentId = studentId;
        this.instructorId = slot.getInstructorId();
        this.slot = slot;
        this.purpose = purpose;
    }

    String getStudentId() {
        return studentId;
    }

    String getInstructorId() {
        return instructorId;
    }

    Slot getSlot() {
        return slot;
    }

    void setSlot(Slot slot) {
        this.slot = slot;
    }

    String getPurpose() {
        return purpose;
    }

    Status getStatus() {
        return status;
    }

    void setStatus(Status status) {
        this.status = status;
    }

    String getNotes() {
        return notes;
    }

    void setNotes(String notes) {
        this.notes = notes;
    }

    String getForwardedBy() {
        return forwardedBy;
    }

    void setForwardedBy(String forwardedBy) {
        this.forwardedBy = forwardedBy;
    }
}

/* =====================================================================
 *  DATA STORE (ArrayLists for users, slots, appointments)
 * ===================================================================== */

class DataStore {
    private final List<User> users = new ArrayList<>();
    private final List<Slot> slots = new ArrayList<>();
    private final List<Appointment> appointments = new ArrayList<>();

    DataStore() {
        seed();
    }

    User authenticate(String role, String id, String pw) throws AppException {
        for (User u : users) {
            if (u.getRole().equals(role) && u.login(id, pw)) {
                return u;
            }
        }
        throw new AppException("Login failed. Please check your details and try again.");
    }

    User findUser(String id) {
        for (User u : users) {
            if (u.getUserId().equals(id)) {
                return u;
            }
        }
        return null;
    }

    List<Instructor> getInstructors() {
        List<Instructor> out = new ArrayList<>();
        for (User u : users) {
            if (u instanceof Instructor) {
                out.add((Instructor) u);
            }
        }
        return out;
    }

    List<Appointment> getAppointments() {
        return appointments;
    }

    List<Slot> slotsOf(String instructorId) {
        List<Slot> out = new ArrayList<>();
        for (Slot s : slots) {
            if (s.getInstructorId().equals(instructorId)) {
                out.add(s);
            }
        }
        out.sort(Comparator.comparing(Slot::getDate).thenComparing(Slot::getStart));
        return out;
    }

    Slot findSlot(String instructorId, LocalDate d, LocalTime s, LocalTime e) {
        for (Slot x : slots) {
            if (x.getInstructorId().equals(instructorId) && x.getDate().equals(d)
                    && x.getStart().equals(s) && x.getEnd().equals(e)) {
                return x;
            }
        }
        return null;
    }

    boolean isTaken(Slot slot) {
        for (Appointment a : appointments) {
            if (a.getSlot() == slot && a.getStatus().isActive()) {
                return true;
            }
        }
        return false;
    }

    Slot addSlot(String instructorId, LocalDate d, LocalTime s, LocalTime e) throws AppException {
        if (!e.isAfter(s)) {
            throw new AppException("The end time must be later than the start time.");
        }
        Slot n = new Slot(instructorId, d, s, e);
        for (Slot x : slots) {
            if (x.overlaps(n)) {
                throw new AppException("This time slot overlaps with an existing slot.");
            }
        }
        slots.add(n);
        return n;
    }

    void removeSlot(Slot s) throws AppException {
        if (s == null) {
            throw new AppException("That slot does not exist.");
        }
        if (isTaken(s)) {
            throw new AppException("This slot has an active appointment and cannot be removed.");
        }
        slots.remove(s);
    }

    Appointment book(Student st, Slot slot, String purpose) throws AppException {
        if (slot == null) {
            throw new AppException("Please select an available date and time.");
        }
        if (purpose == null || purpose.trim().isEmpty()) {
            throw new AppException("Please enter the purpose of your consultation.");
        }
        for (Appointment a : appointments) {
            if (a.getSlot() == slot && a.getStatus().isActive()) {
                if (a.getStudentId().equals(st.getStudentId())) {
                    throw new AppException("You already have a request for this slot.");
                }
                throw new AppException("This slot is already taken.");
            }
        }
        Appointment ap = new Appointment(st.getStudentId(), slot, purpose.trim());
        appointments.add(ap);
        return ap;
    }

    /* ---------- sample data so the screens look like the Expected Output ---------- */
    private Slot seedSlot(String ins, int m, int d, int h1, int min1, int h2, int min2) {
        Slot s = new Slot(ins, LocalDate.of(2026, m, d), LocalTime.of(h1, min1), LocalTime.of(h2, min2));
        slots.add(s);
        return s;
    }

    private void seedAppt(String stu, Slot s, String purpose, Status st, String notes, String fwd) {
        Appointment a = new Appointment(stu, s, purpose);
        a.setStatus(st);
        a.setNotes(notes);
        a.setForwardedBy(fwd);
        appointments.add(a);
    }

    private void seed() {
        String pw = "pass1234";
        users.add(new Student("25-03667", "Glenn B. Tabujara", pw, "BSIT", "A-1001"));
        users.add(new Student("25-02270", "Hailey M. Marquez", pw, "BSIT", "A-1001"));
        users.add(new Student("25-09706", "Valerie A. Umali", pw, "BSIT", "A-1001"));
        users.add(new Adviser("A-001", "Prof. Sheine O. Abe", pw, "IT-2107"));
        users.add(new Instructor("I-001", "Ms. Fatima Marie P. Agdon", pw, "OOP 101"));
        users.add(new Instructor("I-002", "Mr. Gerald P. Dano", pw, "PT 101"));

        Slot a1 = seedSlot("I-001", 10, 21, 10, 0, 11, 0);
        Slot a2 = seedSlot("I-001", 10, 8, 9, 0, 10, 0);
        seedSlot("I-001", 10, 22, 9, 0, 10, 0);
        seedSlot("I-001", 10, 28, 14, 0, 15, 0);
        Slot r1 = seedSlot("I-002", 10, 23, 14, 0, 15, 0);
        Slot r2 = seedSlot("I-002", 10, 22, 13, 0, 14, 0);
        Slot r3 = seedSlot("I-002", 10, 15, 15, 0, 16, 0);
        seedSlot("I-002", 10, 29, 10, 0, 11, 0);

        seedAppt("25-01234", a1, "Grade Inquiry", Status.PENDING, "", "");
        seedAppt("25-01234", r1, "Project Advice", Status.APPROVED, "", "Prof. Maria L. Santos");
        seedAppt("25-01234", a2, "Quiz Concern", Status.COMPLETED,
                "Reviewed Quiz 1 answers; student to revise the Chapter 2 exercises.", "Prof. Maria L. Santos");
        seedAppt("25-01310", r2, "Project Advice", Status.PENDING, "", "");
        seedAppt("25-01310", r3, "Project Advice", Status.COMPLETED,
                "Discussed the project scope and agreed on the submission timeline.", "Prof. Maria L. Santos");
    }
}

/* =====================================================================
 *  GUI
 * ===================================================================== */

class AppFrame extends JFrame {
    static final Color NAVY = new Color(0x1F3864);
    static final Color LABEL = new Color(0xD9E2F3);
    static final Color GRAY = new Color(0xF2F2F2);
    static final Color LINE = new Color(0x8C8C8C);
    static final Color SELECT = new Color(0xBDD7EE);
    static final Color BG = new Color(0xE9EDF4);
    static final Color ERR = new Color(0xC00000);
    static final Color OK = new Color(0x1E7B34);
    static final Color HINT = new Color(0x555555);
    static final Color TAG_RED = new Color(0xD9434B);
    static final Font BASE = new Font("SansSerif", Font.PLAIN, 13);
    static final Font BOLD = new Font("SansSerif", Font.BOLD, 13);

    private final DataStore ds;
    private User current;
    private final JPanel host = new JPanel(new GridBagLayout());

    static void applyDefaultFont(Font f) {
        for (Object key : Collections.list(UIManager.getDefaults().keys())) {
            if (UIManager.get(key) instanceof javax.swing.plaf.FontUIResource) {
                UIManager.put(key, new javax.swing.plaf.FontUIResource(f));
            }
        }
    }

    AppFrame(DataStore ds) {
        this.ds = ds;
        setTitle("Student Consultation Appointment and Record Management System");
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setSize(1000, 720);
        setMinimumSize(new Dimension(900, 640));
        setLocationRelativeTo(null);

        JPanel root = new JPanel(new BorderLayout());
        setContentPane(root);

        JPanel banner = new JPanel(new GridLayout(2, 1));
        banner.setBackground(Color.WHITE);
        banner.setBorder(new CompoundBorder(new MatteBorder(0, 0, 3, 0, Color.BLACK), new EmptyBorder(8, 10, 8, 10)));
        JLabel u1 = new JLabel("BATANGAS STATE UNIVERSITY", SwingConstants.CENTER);
        u1.setFont(new Font("Serif", Font.BOLD, 20));
        JLabel u2 = new JLabel("College of Informatics and Computing Sciences", SwingConstants.CENTER);
        u2.setFont(BOLD);
        banner.add(u1);
        banner.add(u2);
        root.add(banner, BorderLayout.NORTH);

        host.setBackground(BG);
        root.add(host, BorderLayout.CENTER);

        JLabel tag = new JLabel("Leading Innovations, Transforming Lives, Building the Nation", SwingConstants.CENTER);
        tag.setFont(new Font("SansSerif", Font.BOLD | Font.ITALIC, 13));
        tag.setForeground(TAG_RED);
        tag.setBorder(new EmptyBorder(8, 0, 8, 0));
        tag.setOpaque(true);
        tag.setBackground(Color.WHITE);
        root.add(tag, BorderLayout.SOUTH);

        showLogin();
    }

    /* ------------------------------------------------------------------
     *  Layout helpers
     * ------------------------------------------------------------------ */

    /** width > 0 : centered card of that width; width == 0 : fills the window. */
    private void setScreen(JComponent c, int width) {
        host.removeAll();
        GridBagConstraints g = new GridBagConstraints();
        g.gridx = 0;
        g.gridy = 0;
        g.weightx = 1;
        g.weighty = 1;
        if (width > 0) {
            c.setPreferredSize(new Dimension(width, c.getPreferredSize().height));
            g.fill = GridBagConstraints.NONE;
            g.anchor = GridBagConstraints.CENTER;
        } else {
            g.fill = GridBagConstraints.BOTH;
            g.insets = new Insets(14, 24, 14, 24);
        }
        host.add(c, g);
        host.revalidate();
        host.repaint();
    }

    private JPanel screen(String title, JComponent body, JComponent footer) {
        JPanel p = new JPanel(new BorderLayout());
        p.setBackground(Color.WHITE);
        p.setBorder(new LineBorder(LINE));
        JLabel h = new JLabel(title);
        h.setOpaque(true);
        h.setBackground(NAVY);
        h.setForeground(Color.WHITE);
        h.setFont(BOLD);
        h.setBorder(new EmptyBorder(9, 12, 9, 12));
        p.add(h, BorderLayout.NORTH);
        p.add(body, BorderLayout.CENTER);
        if (footer != null) {
            p.add(footer, BorderLayout.SOUTH);
        }
        return p;
    }

    private JPanel bar(JButton... buttons) {
        JPanel p = new JPanel(new FlowLayout(FlowLayout.CENTER, 12, 8));
        p.setBackground(GRAY);
        p.setBorder(new MatteBorder(1, 0, 0, 0, LINE));
        for (JButton b : buttons) {
            p.add(b);
        }
        return p;
    }

    private JButton btn(String text) {
        JButton b = new JButton(text);
        b.setFont(BOLD);
        b.setFocusPainted(false);
        b.setMargin(new Insets(5, 16, 5, 16));
        b.setCursor(Cursor.getPredefinedCursor(Cursor.HAND_CURSOR));
        return b;
    }

    private JLabel msgLabel(String text) {
        JLabel l = new JLabel(text);
        l.setFont(BASE);
        l.setForeground(HINT);
        l.setBorder(new EmptyBorder(6, 10, 6, 10));
        return l;
    }

    private void setMsg(JLabel l, String text, boolean error) {
        l.setForeground(error ? ERR : OK);
        l.setText("<html>" + text.replace("&", "&amp;").replace("<", "&lt;") + "</html>");
    }

    private JPanel form(String[] labels, JComponent[] fields) {
        JPanel p = new JPanel(new GridBagLayout());
        p.setBackground(Color.WHITE);
        for (int i = 0; i < labels.length; i++) {
            GridBagConstraints g = new GridBagConstraints();
            g.gridy = i;
            g.fill = GridBagConstraints.BOTH;
            g.gridx = 0;
            g.weightx = 0;
            g.weighty = (fields[i] instanceof JScrollPane) ? 1 : 0;
            JLabel l = new JLabel(labels[i]);
            l.setFont(BOLD);
            l.setOpaque(true);
            l.setBackground(LABEL);
            l.setBorder(new CompoundBorder(new LineBorder(LINE), new EmptyBorder(6, 10, 6, 10)));
            l.setPreferredSize(new Dimension(170, 38));
            p.add(l, g);

            g = new GridBagConstraints();
            g.gridy = i;
            g.gridx = 1;
            g.weightx = 1;
            g.weighty = (fields[i] instanceof JScrollPane) ? 1 : 0;
            g.fill = GridBagConstraints.BOTH;
            JPanel w = new JPanel(new BorderLayout());
            w.setBackground(Color.WHITE);
            w.setBorder(new CompoundBorder(new LineBorder(LINE), new EmptyBorder(3, 6, 3, 6)));
            if (fields[i] instanceof JTextField) {
                fields[i].setBorder(new EmptyBorder(3, 2, 3, 2));
            }
            w.add(fields[i], BorderLayout.CENTER);
            p.add(w, g);
        }
        return p;
    }

    private JTable table(String... cols) {
        DefaultTableModel m = new DefaultTableModel(cols, 0) {
            @Override
            public boolean isCellEditable(int r, int c) {
                return false;
            }
        };
        JTable t = new JTable(m);
        t.setRowHeight(34);
        t.setGridColor(LINE);
        t.setShowGrid(true);
        t.setSelectionMode(ListSelectionModel.SINGLE_SELECTION);
        t.setSelectionBackground(SELECT);
        t.setSelectionForeground(Color.BLACK);
        t.setFillsViewportHeight(true);
        t.getTableHeader().setReorderingAllowed(false);
        t.getTableHeader().setPreferredSize(new Dimension(100, 34));
        t.getTableHeader().setDefaultRenderer(new DefaultTableCellRenderer() {
            @Override
            public Component getTableCellRendererComponent(JTable tb, Object v, boolean s, boolean f, int r, int c) {
                JLabel l = (JLabel) super.getTableCellRendererComponent(tb, v, false, false, r, c);
                l.setBackground(LABEL);
                l.setForeground(Color.BLACK);
                l.setFont(BOLD);
                l.setBorder(new CompoundBorder(new MatteBorder(0, 0, 1, 1, LINE), new EmptyBorder(0, 8, 0, 8)));
                return l;
            }
        });
        t.setDefaultRenderer(Object.class, new DefaultTableCellRenderer() {
            @Override
            public Component getTableCellRendererComponent(JTable tb, Object v, boolean s, boolean f, int r, int c) {
                JLabel l = (JLabel) super.getTableCellRendererComponent(tb, v, s, false, r, c);
                l.setBorder(new EmptyBorder(2, 8, 2, 8));
                return l;
            }
        });
        return t;
    }

    private void widths(JTable t, int... w) {
        for (int i = 0; i < w.length && i < t.getColumnCount(); i++) {
            t.getColumnModel().getColumn(i).setPreferredWidth(w[i]);
        }
    }

    private JScrollPane scroll(JComponent c, int height) {
        JScrollPane sp = new JScrollPane(c);
        sp.setBorder(new LineBorder(LINE));
        sp.getViewport().setBackground(Color.WHITE);
        sp.setPreferredSize(new Dimension(600, height));
        return sp;
    }

    private JTextArea area(int rows, boolean editable) {
        JTextArea ta = new JTextArea(rows, 30);
        ta.setLineWrap(true);
        ta.setWrapStyleWord(true);
        ta.setEditable(editable);
        ta.setFont(BASE);
        ta.setBorder(new EmptyBorder(4, 6, 4, 6));
        return ta;
    }

    private void fill(JTable t, List<String[]> rows) {
        DefaultTableModel m = (DefaultTableModel) t.getModel();
        m.setRowCount(0);
        for (String[] r : rows) {
            m.addRow(r);
        }
    }

    @SuppressWarnings("unchecked")
    private <T> void render(JComboBox<T> cb, Function<T, String> f) {
        cb.setRenderer(new DefaultListCellRenderer() {
            @Override
            public Component getListCellRendererComponent(JList<?> l, Object v, int i, boolean s, boolean fo) {
                super.getListCellRendererComponent(l, v, i, s, fo);
                setText(v == null ? "" : f.apply((T) v));
                return this;
            }
        });
    }

    /* ------------------------------------------------------------------
     *  Display helpers
     * ------------------------------------------------------------------ */

    private String nameOf(String id) {
        User u = ds.findUser(id);
        return u == null ? id : u.getName();
    }

    private String studentLabel(Appointment a) {
        return nameOf(a.getStudentId()) + " (" + a.getStudentId() + ")";
    }

    private String studentStatusText(Status s) {
        switch (s) {
            case PENDING:
                return "PENDING (waiting for adviser)";
            case FORWARDED:
                return "FORWARDED (waiting for instructor)";
            case COMPLETED:
                return "COMPLETED (see Records)";
            default:
                return s.name();
        }
    }

    private String header(String screenName) {
        return current.getRole() + Fmt.ARROW + screenName + " | Logged in as: "
                + current.getName() + " (" + current.getRole() + ")";
    }

    /* ------------------------------------------------------------------
     *  LOGIN
     * ------------------------------------------------------------------ */

    private void showLogin() {
        current = null;
        JComboBox<String> role = new JComboBox<>(new String[]{"-- Select role --", "Student", "Adviser", "Instructor"});
        JTextField id = new JTextField();
        JPasswordField pw = new JPasswordField();
        JLabel msg = msgLabel(" ");
        JButton login = btn("Login");
        JButton exit = btn("Exit");

        Runnable doLogin = () -> {
            try {
                if (role.getSelectedIndex() == 0) {
                    throw new AppException("Login failed. Please check your details and try again.");
                }
                current = ds.authenticate((String) role.getSelectedItem(), id.getText(), new String(pw.getPassword()));
                showMenu();
            } catch (AppException ex) {
                setMsg(msg, ex.getMessage(), true);
                pw.setText("");
            }
        };
        login.addActionListener(e -> doLogin.run());
        id.addActionListener(e -> doLogin.run());
        pw.addActionListener(e -> doLogin.run());
        exit.addActionListener(e -> System.exit(0));

        JPanel body = new JPanel(new BorderLayout());
        body.setBackground(Color.WHITE);
        body.setBorder(new EmptyBorder(14, 14, 6, 14));
        body.add(form(new String[]{"Role", "User ID", "Password"}, new JComponent[]{role, id, pw}), BorderLayout.CENTER);
        JLabel demo = new JLabel("<html><div style='color:#555555'>Demo logins (password: pass1234) &mdash; "
                + "Student: 25-01234 or 25-01310 &nbsp;|&nbsp; Adviser: A-001 &nbsp;|&nbsp; Instructor: I-001 or I-002</div></html>");
        demo.setFont(new Font("SansSerif", Font.PLAIN, 11));
        demo.setBorder(new EmptyBorder(8, 0, 0, 0));
        JPanel south = new JPanel(new BorderLayout());
        south.setOpaque(false);
        south.add(msg, BorderLayout.NORTH);
        south.add(demo, BorderLayout.SOUTH);
        body.add(south, BorderLayout.SOUTH);

        setScreen(screen("Student Consultation Appointment and Record Management System " + Fmt.DASH + " Login",
                body, bar(login, exit)), 700);
        getRootPane().setDefaultButton(login);
        id.requestFocusInWindow();
    }

    /* ------------------------------------------------------------------
     *  ROLE-BASED MAIN MENU
     * ------------------------------------------------------------------ */

    private void showMenu() {
        getRootPane().setDefaultButton(null);
        String[][] items = current.displayMenu();
        JPanel list = new JPanel(new GridLayout(items.length, 1));
        list.setBackground(Color.WHITE);
        for (int i = 0; i < items.length; i++) {
            final String label = items[i][0];
            final JPanel row = new JPanel(new BorderLayout());
            row.setBackground(Color.WHITE);
            row.setBorder(new MatteBorder(0, 0, 1, 0, LINE));
            row.setPreferredSize(new Dimension(100, 58));
            row.setCursor(Cursor.getPredefinedCursor(Cursor.HAND_CURSOR));

            JLabel num = new JLabel(String.valueOf(i + 1), SwingConstants.CENTER);
            num.setFont(BOLD);
            num.setPreferredSize(new Dimension(64, 58));
            num.setBorder(new MatteBorder(0, 0, 0, 1, LINE));

            JLabel name = new JLabel(label);
            name.setFont(BOLD);
            name.setBorder(new EmptyBorder(0, 12, 0, 8));
            JLabel desc = new JLabel("<html>" + items[i][1] + "</html>");
            desc.setBorder(new CompoundBorder(new MatteBorder(0, 1, 0, 0, LINE), new EmptyBorder(0, 12, 0, 8)));
            JPanel mid = new JPanel(new GridLayout(1, 2));
            mid.setOpaque(false);
            mid.add(name);
            mid.add(desc);

            row.add(num, BorderLayout.WEST);
            row.add(mid, BorderLayout.CENTER);
            row.addMouseListener(new MouseAdapter() {
                @Override
                public void mouseClicked(MouseEvent e) {
                    menuAction(label);
                }

                @Override
                public void mouseEntered(MouseEvent e) {
                    row.setBackground(LABEL);
                }

                @Override
                public void mouseExited(MouseEvent e) {
                    row.setBackground(Color.WHITE);
                }
            });
            list.add(row);
        }
        JPanel body = new JPanel(new BorderLayout());
        body.setBackground(Color.WHITE);
        body.add(list, BorderLayout.CENTER);
        JLabel hint = msgLabel("Click an option to continue.");
        body.add(hint, BorderLayout.SOUTH);

        setScreen(screen(current.getRole() + " Main Menu | Logged in as: " + current.getName()
                + " (" + current.getRole() + ")", body, null), 760);
    }

    private void menuAction(String label) {
        switch (label) {
            case "Book Appointment":
                showBook();
                break;
            case "Check Appointment Status":
            case "Cancel Appointment":
                showStatus();
                break;
            case "View Consultation Records":
                if (current instanceof Student) {
                    showStudentRecords();
                } else {
                    showInstructorRecords();
                }
                break;
            case "Review Student Requests":
            case "Forward Request to Instructor":
                showReview();
                break;
            case "Monitor Advisees' Consultations":
                showMonitor();
                break;
            case "Set Available Schedule":
                showSchedule();
                break;
            case "Manage Forwarded Requests":
                showManage();
                break;
            case "Record Completed Consultation":
                showRecord();
                break;
            case "Logout":
                current.logout();
                showLogin();
                break;
            default:
                break;
        }
    }

    /* ------------------------------------------------------------------
     *  STUDENT SCREENS
     * ------------------------------------------------------------------ */

    private void showBook() {
        Student st = (Student) current;
        JComboBox<Instructor> inst = new JComboBox<>(ds.getInstructors().toArray(new Instructor[0]));
        JComboBox<Slot> slots = new JComboBox<>();
        render(slots, s -> s.getDateTimeLabel() + (ds.isTaken(s) ? "   (taken)" : ""));
        JComboBox<String> purpose = new JComboBox<>(new String[]{
                "Grade Inquiry", "Project Advice", "Quiz Concern", "Academic Advice", "Other"});
        purpose.setEditable(true);
        JLabel msg = msgLabel("Choose an instructor, an available date and time, and the purpose.");

        Runnable load = () -> {
            slots.removeAllItems();
            Instructor i = (Instructor) inst.getSelectedItem();
            if (i != null) {
                for (Slot s : ds.slotsOf(i.getInstructorId())) {
                    slots.addItem(s);
                }
            }
        };
        inst.addActionListener(e -> load.run());
        load.run();

        JButton book = btn("Book Appointment");
        JButton back = btn("Back to Menu");
        book.addActionListener(e -> {
            try {
                Slot s = (Slot) slots.getSelectedItem();
                String p = String.valueOf(purpose.getEditor().getItem()).trim();
                Appointment a = st.bookAppointment(ds, s, p);
                setMsg(msg, "Appointment request sent to your adviser. Status: " + a.getStatus() + ".", false);
                slots.repaint();
            } catch (AppException ex) {
                setMsg(msg, ex.getMessage(), true);
            }
        });
        back.addActionListener(e -> showMenu());

        JPanel body = new JPanel(new BorderLayout());
        body.setBackground(Color.WHITE);
        body.setBorder(new EmptyBorder(14, 14, 6, 14));
        body.add(form(new String[]{"Instructor", "Date and Time", "Purpose"},
                new JComponent[]{inst, slots, purpose}), BorderLayout.CENTER);
        body.add(msg, BorderLayout.SOUTH);
        setScreen(screen(header("Book Appointment"), body, bar(book, back)), 720);
    }

    private void showStatus() {
        Student st = (Student) current;
        JTable t = table("No.", "Instructor", "Date / Time", "Purpose", "Status");
        widths(t, 45, 210, 190, 130, 230);
        List<Appointment> rows = new ArrayList<>();
        JLabel msg = msgLabel("Select a request in the table to cancel it.");

        Runnable refresh = () -> {
            rows.clear();
            rows.addAll(st.checkStatus(ds));
            List<String[]> data = new ArrayList<>();
            int n = 1;
            for (Appointment a : rows) {
                data.add(new String[]{String.valueOf(n++), nameOf(a.getInstructorId()),
                        a.getSlot().getDateTimeLabel(), a.getPurpose(), studentStatusText(a.getStatus())});
            }
            fill(t, data);
        };
        refresh.run();

        JButton cancel = btn("Cancel Selected Appointment");
        JButton back = btn("Back to Menu");
        cancel.addActionListener(e -> {
            int r = t.getSelectedRow();
            if (r < 0) {
                setMsg(msg, "Please select an appointment first.", true);
                return;
            }
            if (JOptionPane.showConfirmDialog(this, "Cancel the selected appointment?", "Confirm",
                    JOptionPane.YES_NO_OPTION) != JOptionPane.YES_OPTION) {
                return;
            }
            try {
                st.cancelAppointment(rows.get(r));
                refresh.run();
                setMsg(msg, "Appointment cancelled.", false);
            } catch (AppException ex) {
                setMsg(msg, ex.getMessage(), true);
            }
        });
        back.addActionListener(e -> showMenu());

        JPanel body = new JPanel(new BorderLayout());
        body.setBackground(Color.WHITE);
        body.setBorder(new EmptyBorder(12, 12, 4, 12));
        body.add(scroll(t, 260), BorderLayout.CENTER);
        body.add(msg, BorderLayout.SOUTH);
        setScreen(screen(header("My Appointments"), body, bar(cancel, back)), 0);
    }

    private void showStudentRecords() {
        Student st = (Student) current;
        recordsScreen("Consultation Records",
                new String[]{"Instructor", "Date / Time", "Purpose", "Status"},
                new int[]{45, 230, 190, 140, 120},
                () -> st.viewRecords(ds),
                a -> new String[]{nameOf(a.getInstructorId()), a.getSlot().getDateTimeLabel(),
                        a.getPurpose(), a.getStatus().name()},
                "Search Records");
    }

    /* ------------------------------------------------------------------
     *  ADVISER SCREENS
     * ------------------------------------------------------------------ */

    private void showReview() {
        Adviser adv = (Adviser) current;
        JTable t = table("No.", "Student", "Instructor", "Date / Time", "Purpose");
        widths(t, 45, 210, 210, 170, 130);
        List<Appointment> rows = new ArrayList<>();
        JLabel msg = msgLabel(" ");

        Runnable refresh = () -> {
            rows.clear();
            rows.addAll(adv.reviewRequest(ds));
            List<String[]> data = new ArrayList<>();
            int n = 1;
            for (Appointment a : rows) {
                data.add(new String[]{String.valueOf(n++), studentLabel(a), nameOf(a.getInstructorId()),
                        a.getSlot().getDateTimeLabel(), a.getPurpose()});
            }
            fill(t, data);
            if (rows.isEmpty()) {
                setMsg(msg, "No pending requests from your advisees.", false);
            }
        };
        refresh.run();

        JButton forward = btn("Forward to Instructor");
        JButton decline = btn("Decline");
        JButton back = btn("Back to Menu");
        forward.addActionListener(e -> {
            int r = t.getSelectedRow();
            if (r < 0) {
                setMsg(msg, "Please select a request first.", true);
                return;
            }
            try {
                Appointment a = rows.get(r);
                adv.forwardRequest(a);
                refresh.run();
                setMsg(msg, "Request forwarded to " + nameOf(a.getInstructorId()) + ". Status: " + a.getStatus() + ".", false);
            } catch (AppException ex) {
                setMsg(msg, ex.getMessage(), true);
            }
        });
        decline.addActionListener(e -> {
            int r = t.getSelectedRow();
            if (r < 0) {
                setMsg(msg, "Please select a request first.", true);
                return;
            }
            if (JOptionPane.showConfirmDialog(this, "Decline the selected request?", "Confirm",
                    JOptionPane.YES_NO_OPTION) != JOptionPane.YES_OPTION) {
                return;
            }
            try {
                Appointment a = rows.get(r);
                adv.declineRequest(a);
                refresh.run();
                setMsg(msg, "Request declined. Status: " + a.getStatus() + ".", false);
            } catch (AppException ex) {
                setMsg(msg, ex.getMessage(), true);
            }
        });
        back.addActionListener(e -> showMenu());

        JLabel note = msgLabel("<html>Select a request, then choose an action. Forwarding changes the status from PENDING to "
                + "FORWARDED. Declining returns the request to the student with the status DECLINED.</html>");
        JPanel south = new JPanel(new BorderLayout());
        south.setOpaque(false);
        south.add(note, BorderLayout.NORTH);
        south.add(msg, BorderLayout.SOUTH);

        JPanel body = new JPanel(new BorderLayout());
        body.setBackground(Color.WHITE);
        body.setBorder(new EmptyBorder(12, 12, 4, 12));
        body.add(scroll(t, 240), BorderLayout.CENTER);
        body.add(south, BorderLayout.SOUTH);
        setScreen(screen(header("Pending Requests"), body, bar(forward, decline, back)), 0);
    }

    private void showMonitor() {
        Adviser adv = (Adviser) current;
        recordsScreen("Advisees' Consultations",
                new String[]{"Student", "Instructor", "Date / Time", "Status"},
                new int[]{45, 190, 210, 190, 120},
                () -> adv.monitorAdvisees(ds),
                a -> new String[]{nameOf(a.getStudentId()), nameOf(a.getInstructorId()),
                        a.getSlot().getDateTimeLabel(), a.getStatus().name()},
                "Search");
    }

    /* ------------------------------------------------------------------
     *  INSTRUCTOR SCREENS
     * ------------------------------------------------------------------ */

    private void showSchedule() {
        Instructor in = (Instructor) current;
        JTextField date = new JTextField();
        JTextField time = new JTextField();
        date.setToolTipText("Example: Oct 21, 2026");
        time.setToolTipText("Example: 10:00 AM - 11:00 AM");
        JLabel msg = msgLabel("Date format: Oct 21, 2026     |     Time slot format: 10:00 AM - 11:00 AM");

        JTable t = table("No.", "Date", "Time Slot", "Availability");
        widths(t, 45, 200, 240, 140);
        List<Slot> rows = new ArrayList<>();
        Runnable refresh = () -> {
            rows.clear();
            rows.addAll(ds.slotsOf(in.getInstructorId()));
            List<String[]> data = new ArrayList<>();
            int n = 1;
            for (Slot s : rows) {
                data.add(new String[]{String.valueOf(n++), s.getDate().format(Fmt.DATE_OUT),
                        s.getRangeLabel(), ds.isTaken(s) ? "Booked" : "Open"});
            }
            fill(t, data);
        };
        refresh.run();
        t.getSelectionModel().addListSelectionListener(e -> {
            int r = t.getSelectedRow();
            if (!e.getValueIsAdjusting() && r >= 0 && r < rows.size()) {
                date.setText(rows.get(r).getDate().format(Fmt.DATE_OUT));
                time.setText(rows.get(r).getRangeLabel().replace(Fmt.DASH, "-"));
            }
        });

        JButton add = btn("Add Slot");
        JButton remove = btn("Remove Slot");
        JButton back = btn("Back to Menu");
        add.addActionListener(e -> {
            try {
                LocalDate d = Fmt.parseDate(date.getText());
                LocalTime[] range = Fmt.parseRange(time.getText());
                in.setSchedule(ds, d, range[0], range[1]);
                refresh.run();
                setMsg(msg, "Slot added: " + d.format(Fmt.DATE_OUT) + ", "
                        + range[0].format(Fmt.TIME_OUT) + " " + Fmt.DASH + " " + range[1].format(Fmt.TIME_OUT), false);
            } catch (AppException ex) {
                setMsg(msg, ex.getMessage(), true);
            }
        });
        remove.addActionListener(e -> {
            try {
                Slot target;
                int r = t.getSelectedRow();
                if (r >= 0 && r < rows.size()) {
                    target = rows.get(r);
                } else {
                    LocalDate d = Fmt.parseDate(date.getText());
                    LocalTime[] range = Fmt.parseRange(time.getText());
                    target = ds.findSlot(in.getInstructorId(), d, range[0], range[1]);
                }
                in.removeSlot(ds, target);
                refresh.run();
                date.setText("");
                time.setText("");
                setMsg(msg, "Slot removed.", false);
            } catch (AppException ex) {
                setMsg(msg, ex.getMessage(), true);
            }
        });
        back.addActionListener(e -> showMenu());

        JPanel body = new JPanel(new BorderLayout(0, 10));
        body.setBackground(Color.WHITE);
        body.setBorder(new EmptyBorder(12, 12, 4, 12));
        body.add(form(new String[]{"Date", "Time Slot"}, new JComponent[]{date, time}), BorderLayout.NORTH);
        body.add(scroll(t, 220), BorderLayout.CENTER);
        body.add(msg, BorderLayout.SOUTH);
        setScreen(screen(header("Set Available Schedule"), body, bar(add, remove, back)), 0);
    }

    private void showManage() {
        Instructor in = (Instructor) current;
        JTable t = table("No.", "Student", "Date / Time", "Purpose", "Forwarded by");
        widths(t, 45, 210, 190, 130, 190);
        List<Appointment> rows = new ArrayList<>();
        JLabel msg = msgLabel(" ");

        Runnable refresh = () -> {
            rows.clear();
            rows.addAll(in.getByStatus(ds, Status.FORWARDED));
            List<String[]> data = new ArrayList<>();
            int n = 1;
            for (Appointment a : rows) {
                data.add(new String[]{String.valueOf(n++), studentLabel(a), a.getSlot().getDateTimeLabel(),
                        a.getPurpose(), a.getForwardedBy()});
            }
            fill(t, data);
            if (rows.isEmpty()) {
                setMsg(msg, "No forwarded requests right now.", false);
            }
        };
        refresh.run();

        JButton approve = btn("Approve");
        JButton resched = btn("Reschedule");
        JButton decline = btn("Decline");
        JButton back = btn("Back to Menu");

        approve.addActionListener(e -> {
            int r = t.getSelectedRow();
            if (r < 0) {
                setMsg(msg, "Please select a request first.", true);
                return;
            }
            try {
                Appointment a = rows.get(r);
                in.manageAppointment(ds, a, "approve", null);
                refresh.run();
                setMsg(msg, "Request approved. Status: " + a.getStatus() + ".", false);
            } catch (AppException ex) {
                setMsg(msg, ex.getMessage(), true);
            }
        });
        decline.addActionListener(e -> {
            int r = t.getSelectedRow();
            if (r < 0) {
                setMsg(msg, "Please select a request first.", true);
                return;
            }
            if (JOptionPane.showConfirmDialog(this, "Decline the selected request?", "Confirm",
                    JOptionPane.YES_NO_OPTION) != JOptionPane.YES_OPTION) {
                return;
            }
            try {
                Appointment a = rows.get(r);
                in.manageAppointment(ds, a, "decline", null);
                refresh.run();
                setMsg(msg, "Request declined. Status: " + a.getStatus() + ".", false);
            } catch (AppException ex) {
                setMsg(msg, ex.getMessage(), true);
            }
        });
        resched.addActionListener(e -> {
            int r = t.getSelectedRow();
            if (r < 0) {
                setMsg(msg, "Please select a request first.", true);
                return;
            }
            Appointment a = rows.get(r);
            List<Slot> open = new ArrayList<>();
            for (Slot s : ds.slotsOf(in.getInstructorId())) {
                if (!ds.isTaken(s)) {
                    open.add(s);
                }
            }
            if (open.isEmpty()) {
                setMsg(msg, "You have no other open slots. Add one under Set Available Schedule first.", true);
                return;
            }
            JComboBox<Slot> choose = new JComboBox<>(open.toArray(new Slot[0]));
            render(choose, s -> s.getDateTimeLabel());
            JPanel pane = new JPanel(new BorderLayout(0, 8));
            pane.add(new JLabel("Choose a new open slot for " + nameOf(a.getStudentId()) + ":"), BorderLayout.NORTH);
            pane.add(choose, BorderLayout.CENTER);
            if (JOptionPane.showConfirmDialog(this, pane, "Reschedule", JOptionPane.OK_CANCEL_OPTION,
                    JOptionPane.PLAIN_MESSAGE) != JOptionPane.OK_OPTION) {
                return;
            }
            try {
                in.manageAppointment(ds, a, "reschedule", (Slot) choose.getSelectedItem());
                refresh.run();
                setMsg(msg, "Request rescheduled to " + a.getSlot().getDateTimeLabel() + ". Status: " + a.getStatus() + ".", false);
            } catch (AppException ex) {
                setMsg(msg, ex.getMessage(), true);
            }
        });
        back.addActionListener(e -> showMenu());

        JLabel note = msgLabel("<html>Approve changes the status from FORWARDED to APPROVED. Reschedule asks for a new open slot. "
                + "Decline sets the status to DECLINED. The student and the adviser see the new status right away.</html>");
        JPanel south = new JPanel(new BorderLayout());
        south.setOpaque(false);
        south.add(note, BorderLayout.NORTH);
        south.add(msg, BorderLayout.SOUTH);

        JPanel body = new JPanel(new BorderLayout());
        body.setBackground(Color.WHITE);
        body.setBorder(new EmptyBorder(12, 12, 4, 12));
        body.add(scroll(t, 240), BorderLayout.CENTER);
        body.add(south, BorderLayout.SOUTH);
        setScreen(screen(header("Forwarded Requests"), body, bar(approve, resched, decline, back)), 0);
    }

    private void showRecord() {
        Instructor in = (Instructor) current;
        JComboBox<Appointment> sel = new JComboBox<>(in.getByStatus(ds, Status.APPROVED).toArray(new Appointment[0]));
        render(sel, a -> studentLabel(a) + "  |  " + a.getSlot().getDateTimeLabel());
        JTextField stu = new JTextField();
        JTextField when = new JTextField();
        JTextField purpose = new JTextField();
        for (JTextField f : new JTextField[]{stu, when, purpose}) {
            f.setEditable(false);
            f.setBackground(Color.WHITE);
        }
        JTextArea notes = area(5, true);
        JScrollPane notesPane = new JScrollPane(notes);
        notesPane.setBorder(null);
        JLabel msg = msgLabel(sel.getItemCount() == 0
                ? "There are no approved appointments to record yet."
                : "Pick an approved appointment, type the notes, then save.");

        Runnable show = () -> {
            Appointment a = (Appointment) sel.getSelectedItem();
            stu.setText(a == null ? "" : studentLabel(a));
            when.setText(a == null ? "" : a.getSlot().getDateTimeLabel());
            purpose.setText(a == null ? "" : a.getPurpose());
        };
        sel.addActionListener(e -> show.run());
        show.run();

        JButton save = btn("Save Record");
        JButton back = btn("Back to Menu");
        save.addActionListener(e -> {
            try {
                Appointment a = (Appointment) sel.getSelectedItem();
                in.recordConsultation(a, notes.getText());
                sel.removeItem(a);
                notes.setText("");
                show.run();
                setMsg(msg, "Consultation saved as COMPLETED. It now appears under View Consultation Records.", false);
            } catch (AppException ex) {
                setMsg(msg, ex.getMessage(), true);
            }
        });
        back.addActionListener(e -> showMenu());

        JPanel body = new JPanel(new BorderLayout());
        body.setBackground(Color.WHITE);
        body.setBorder(new EmptyBorder(14, 14, 6, 14));
        body.add(form(new String[]{"Approved Appointment", "Student", "Date and Time", "Purpose", "Notes"},
                new JComponent[]{sel, stu, when, purpose, notesPane}), BorderLayout.CENTER);
        body.add(msg, BorderLayout.SOUTH);
        setScreen(screen(header("Record Consultation"), body, bar(save, back)), 760);
    }

    private void showInstructorRecords() {
        Instructor in = (Instructor) current;
        recordsScreen("Consultation Records",
                new String[]{"Student", "Date / Time", "Purpose", "Status"},
                new int[]{45, 230, 190, 140, 120},
                () -> in.getByStatus(ds, Status.COMPLETED),
                a -> new String[]{studentLabel(a), a.getSlot().getDateTimeLabel(), a.getPurpose(), a.getStatus().name()},
                "Search Records");
    }

    /* ------------------------------------------------------------------
     *  Shared "table + search + notes" screen (records / monitoring)
     * ------------------------------------------------------------------ */

    private void recordsScreen(String name, String[] cols, int[] w, Supplier<List<Appointment>> source,
                               Function<Appointment, String[]> mapper, String searchLabel) {
        String[] full = new String[cols.length + 1];
        full[0] = "No.";
        System.arraycopy(cols, 0, full, 1, cols.length);
        JTable t = table(full);
        widths(t, w);
        JTextField q = new JTextField(24);
        JTextArea notes = area(3, false);
        List<Appointment> shown = new ArrayList<>();

        Runnable refresh = () -> {
            shown.clear();
            List<String[]> data = new ArrayList<>();
            String key = q.getText().trim().toLowerCase();
            int n = 1;
            for (Appointment a : source.get()) {
                String[] r = mapper.apply(a);
                String hay = (String.join(" ", r) + " " + a.getNotes()).toLowerCase();
                if (key.isEmpty() || hay.contains(key)) {
                    shown.add(a);
                    String[] row = new String[r.length + 1];
                    row[0] = String.valueOf(n++);
                    System.arraycopy(r, 0, row, 1, r.length);
                    data.add(row);
                }
            }
            fill(t, data);
            notes.setText("");
        };
        refresh.run();

        t.getSelectionModel().addListSelectionListener(e -> {
            if (e.getValueIsAdjusting()) {
                return;
            }
            int r = t.getSelectedRow();
            if (r < 0 || r >= shown.size()) {
                notes.setText("");
                return;
            }
            Appointment a = shown.get(r);
            notes.setText(a.getStatus() == Status.COMPLETED ? a.getNotes()
                    : "No notes yet. Notes are saved once the consultation is completed.");
        });

        JButton search = btn(searchLabel);
        JButton clear = btn("Clear");
        JButton back = btn("Back to Menu");
        search.addActionListener(e -> refresh.run());
        q.addActionListener(e -> refresh.run());
        clear.addActionListener(e -> {
            q.setText("");
            refresh.run();
        });
        back.addActionListener(e -> showMenu());

        JPanel top = new JPanel(new FlowLayout(FlowLayout.LEFT, 8, 4));
        top.setOpaque(false);
        JLabel sl = new JLabel("Search:");
        sl.setFont(BOLD);
        top.add(sl);
        top.add(q);

        JPanel notesRow = new JPanel(new BorderLayout());
        notesRow.setOpaque(false);
        notesRow.setBorder(new EmptyBorder(8, 0, 0, 0));
        JLabel nl = new JLabel("Instructor's Notes");
        nl.setFont(BOLD);
        nl.setOpaque(true);
        nl.setBackground(LABEL);
        nl.setBorder(new CompoundBorder(new LineBorder(LINE), new EmptyBorder(6, 10, 6, 10)));
        nl.setPreferredSize(new Dimension(170, 70));
        notesRow.add(nl, BorderLayout.WEST);
        JScrollPane np = new JScrollPane(notes);
        np.setBorder(new LineBorder(LINE));
        notesRow.add(np, BorderLayout.CENTER);

        JPanel body = new JPanel(new BorderLayout());
        body.setBackground(Color.WHITE);
        body.setBorder(new EmptyBorder(10, 12, 8, 12));
        body.add(top, BorderLayout.NORTH);
        body.add(scroll(t, 220), BorderLayout.CENTER);
        body.add(notesRow, BorderLayout.SOUTH);
        setScreen(screen(header(name), body, bar(search, clear, back)), 0);
    }
}
