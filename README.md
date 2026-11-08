# Project Completion Calculator Using Java Servlet, JSP and JDBC
Name: Al-Shahariar
 
ID:IT-24057
 
## Objective
 
Develop a web-based Project Completion Calculator for the ICT Lab Resource Booking System project using Java Servlet, JSP, manual JDBC and Apache Tomcat. The application calculates the total days between a start date and an end date (or the end date from a number of days), shows the duration on a timeline chart, saves each project in a MySQL database, and lets the user add tasks to a project, mark them done and see the progress percentage.
 
## Technologies Used
 
| Technology | Version / Purpose |
| --- | --- |
| Java JDK | 25 |
| Apache Tomcat | 10.1.61 |
| Jakarta Servlet API | HTTP request and response handling |
| JSP | Displaying pages (view) |
| JDBC (DriverManager, PreparedStatement) | Database access without any framework |
| MySQL | 8.4 (Laragon) – stores saved projects and their tasks |
| Maven | Building the WAR file |
| Chart.js | Timeline chart (loaded from CDN) |
| HTML, CSS and JavaScript | User interface, styling and chart |
| Visual Studio Code | Source code editing and terminal |
| PowerShell | Build and server startup |
 
No Spring Boot, Spring MVC, Spring REST, Hibernate or JPA is used.
 
## Features
 
- **Task 1 – Calculator:** form with Project Name, Start Date and End Date, or only the Number of days.
- Calculates total days between two dates; start and end day are both counted (01 Nov to 10 Nov = 10 days).
- If only the number of days is given, calculates the end date (today is used when Start Date is empty).
- Shows a summary, e.g. `Project X: 01 Nov 2026 to 10 Nov 2026 = 10 days`.
- **Task 2 – Chart:** horizontal timeline bar with Chart.js on a 1-month axis. If the chart cannot render, the text summary is shown instead.
- **Task 3 – Database:** a Save button stores project name, start date, end date and total days in MySQL using manual JDBC.
- A Saved Projects page lists all saved projects with a timeline of all of them.
- **Task 4 – Tasks and progress:** each saved project has its own page where the user can add tasks and mark them done (or undo).
- Shows progress as `done tasks / all tasks × 100`, e.g. `3 of 4 tasks done = 75% complete`, with a progress bar.
- The Saved Projects page shows each project's progress, e.g. `75% (3/4)`, and each project name opens its task page.
- Handles invalid input (empty name, End Date before Start Date, wrong number of days, empty task title).
- Uses `PreparedStatement` (no SQL injection) and try-with-resources (connections always closed).
- User input is HTML-escaped before being displayed.
 
## Database Design
 
Database: `project_calc`, Table: `projects`
 
| Column | Type | Description |
| --- | --- | --- |
| `id` | INT, AUTO_INCREMENT, PRIMARY KEY | Row number |
| `project_name` | VARCHAR(150) | Project name |
| `start_date` | DATE | Start date |
| `end_date` | DATE | End date |
| `total_days` | INT | Total days (start and end included) |
| `created_at` | DATETIME | Time of saving |
 
Table: `tasks` (one project has many tasks)
 
| Column | Type | Description |
| --- | --- | --- |
| `id` | INT, AUTO_INCREMENT, PRIMARY KEY | Row number |
| `project_id` | INT, FOREIGN KEY → `projects.id` (ON DELETE CASCADE) | Project of this task |
| `title` | VARCHAR(200) | Task title |
| `done` | BOOLEAN | `0` = not done, `1` = done |
| `created_at` | DATETIME | Time of adding |
 
**Progress formula:** `progress % = done tasks × 100 / total tasks` (0% when there are no tasks). The counts come from one SQL query with `LEFT JOIN tasks`, `COUNT(t.id)` and `SUM(CASE WHEN t.done THEN 1 ELSE 0 END)`, grouped by project.
 
## Source Code
 
### schema.sql
 
`db/schema.sql`
 
```sql
-- Project Completion Calculator - database script
-- Run:  mysql -u root -p < db/schema.sql
 
-- 1. Create the database (only if it does not exist yet).
CREATE DATABASE IF NOT EXISTS project_calc
    CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
 
USE project_calc;
 
-- 2. One table: every saved calculation is one row.
CREATE TABLE IF NOT EXISTS projects (
    id           INT AUTO_INCREMENT PRIMARY KEY,          -- auto number 1, 2, 3 ...
    project_name VARCHAR(150) NOT NULL,
    start_date   DATE NOT NULL,
    end_date     DATE NOT NULL,
    total_days   INT  NOT NULL,                           -- counted with start and end day included
    created_at   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CHECK (end_date >= start_date),
    CHECK (total_days > 0)
);
 
-- 3. One example row: the schedule of this capstone project (ICT Lab Resource Booking System).
INSERT INTO projects (project_name, start_date, end_date, total_days)
SELECT 'ICT Lab Resource Booking System', '2026-10-24', '2026-10-28', 5
WHERE NOT EXISTS (SELECT 1 FROM projects);
 
-- 4. Tasks of each project. One project has many tasks.
--    done = 0 (not done) or 1 (done).
--    ON DELETE CASCADE: if a project is deleted, its tasks are deleted too.
CREATE TABLE IF NOT EXISTS tasks (
    id         INT AUTO_INCREMENT PRIMARY KEY,
    project_id INT NOT NULL,
    title      VARCHAR(200) NOT NULL,
    done       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE
);
```
 
### db.properties
 
`src/main/resources/db.properties`
 
```properties
# Database settings. Change user/password to match your MySQL.
# (Laragon / XAMPP default: user = root, password = empty)
url=jdbc:mysql://localhost:3306/project_calc
user=root
password=
```
 
### DBConnection.java
 
`src/main/java/com/projectcalc/db/DBConnection.java`
 
```java
package com.projectcalc.db;
 
import java.io.IOException;
import java.io.InputStream;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;
import java.util.Properties;
 
/**
 * The only place that opens a database connection.
 * It reads url/user/password from db.properties and uses plain DriverManager.
 */
public class DBConnection {
 
    private static final Properties CONFIG = new Properties();
 
    // "static block" runs only once, when the class is first used.
    static {
        // try-with-resources: the stream is closed automatically at the end of the block.
        try (InputStream in = DBConnection.class.getResourceAsStream("/db.properties")) {
            if (in == null) {
                throw new IllegalStateException("db.properties not found in src/main/resources");
            }
            CONFIG.load(in);
 
            // Load the MySQL driver class. Inside Tomcat this line is needed,
            // because Tomcat does not always find the driver by itself.
            Class.forName("com.mysql.cj.jdbc.Driver");
        } catch (IOException | ClassNotFoundException e) {
            throw new IllegalStateException("Cannot set up the database connection", e);
        }
    }
 
    /**
     * Opens a new connection. The caller MUST close it,
     * so always call it inside try-with-resources:
     *   try (Connection con = DBConnection.getConnection()) { ... }
     */
    public static Connection getConnection() throws SQLException {
        return DriverManager.getConnection(
                CONFIG.getProperty("url"),
                CONFIG.getProperty("user"),
                CONFIG.getProperty("password"));
    }
}
```
 
### Project.java
 
`src/main/java/com/projectcalc/model/Project.java`
 
```java
package com.projectcalc.model;
 
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
 
/**
 * A plain data class: one project with its dates and total days.
 * Same fields as the "projects" table.
 */
public class Project {
 
    // Format used on screen, e.g. "01 Nov 2026"
    private static final DateTimeFormatter SHOW = DateTimeFormatter.ofPattern("dd MMM yyyy");
 
    private int id;               // 0 = not saved yet
    private String name;
    private LocalDate startDate;
    private LocalDate endDate;
    private int totalDays;
 
    public Project(int id, String name, LocalDate startDate, LocalDate endDate, int totalDays) {
        this.id = id;
        this.name = name;
        this.startDate = startDate;
        this.endDate = endDate;
        this.totalDays = totalDays;
    }
 
    public int getId()              { return id; }
    public String getName()         { return name; }
    public LocalDate getStartDate() { return startDate; }
    public LocalDate getEndDate()   { return endDate; }
    public int getTotalDays()       { return totalDays; }
 
    // Dates as text for the page, e.g. "01 Nov 2026"
    public String getStartText()    { return startDate.format(SHOW); }
    public String getEndText()      { return endDate.format(SHOW); }
 
    // Dates as "number of days since 1970-01-01". The chart JavaScript uses these
    // numbers for its x-axis, so it does not need any extra date library.
    public long getStartEpochDay()  { return startDate.toEpochDay(); }
    public long getEndEpochDay()    { return endDate.toEpochDay(); }
 
    /** Summary line, e.g. "Project X: 01 Nov 2026 to 10 Nov 2026 = 10 days" */
    public String getSummary() {
        return name + ": " + getStartText() + " to " + getEndText()
                + " = " + totalDays + (totalDays == 1 ? " day" : " days");
    }
 
    // ---------- Task progress ----------
    // How many tasks this project has, and how many of them are done.
    // ProjectDao fills these with COUNT(...) from the "tasks" table.
    private int taskTotal;
    private int taskDone;
 
    public int getTaskTotal()       { return taskTotal; }
    public int getTaskDone()        { return taskDone; }
 
    public void setTaskCounts(int total, int done) {
        this.taskTotal = total;
        this.taskDone = done;
    }
 
    /** Progress in percent: done / total * 100, e.g. 3 of 4 tasks = 75. No tasks = 0. */
    public int getProgress() {
        if (taskTotal == 0) {
            return 0;
        }
        return Math.round(taskDone * 100f / taskTotal);
    }
}
```
 
### Task.java
 
`src/main/java/com/projectcalc/model/Task.java`
 
```java
package com.projectcalc.model;
 
/**
 * One task of a project (one row of the "tasks" table).
 */
public class Task {
 
    private int id;
    private int projectId;      // which project this task belongs to
    private String title;
    private boolean done;       // true = task is completed
 
    public Task(int id, int projectId, String title, boolean done) {
        this.id = id;
        this.projectId = projectId;
        this.title = title;
        this.done = done;
    }
 
    public int getId()          { return id; }
    public int getProjectId()   { return projectId; }
    public String getTitle()    { return title; }
    public boolean isDone()     { return done; }
}
```
 
### DateCalculator.java
 
`src/main/java/com/projectcalc/util/DateCalculator.java`
 
```java
package com.projectcalc.util;
 
import com.projectcalc.model.Project;
 
import java.time.LocalDate;
import java.time.format.DateTimeParseException;
import java.time.temporal.ChronoUnit;
 
/**
 * All the calculation rules are here, in one small class.
 * Both CalculateServlet and SaveServlet use it, so the rules are written only once.
 *
 * Counting rule: the start day and the end day are BOTH counted.
 *   01 Nov to 10 Nov = 10 days  (not 9)
 */
public class DateCalculator {
 
    /**
     * Builds a Project from the raw form values (all are Strings from the request).
     *
     * Case 1: End Date is filled      -> total days = days between start and end + 1
     * Case 2: Number of Days is filled -> end date  = start + (days - 1)
     *         (if Start Date is empty in case 2, today is used as the start)
     *
     * Throws IllegalArgumentException with a readable message if the input is wrong.
     * The servlet shows that message on the page.
     */
    public static Project calculate(String name, String start, String end, String days) {
        name = trim(name);
        start = trim(start);
        end = trim(end);
        days = trim(days);
 
        if (name.isEmpty()) {
            throw new IllegalArgumentException("Project name is required.");
        }
        if (name.length() > 150) {
            throw new IllegalArgumentException("Project name must be 150 characters or less.");
        }
 
        // ---- Case 1: start date + end date ----
        if (!end.isEmpty()) {
            if (start.isEmpty()) {
                throw new IllegalArgumentException("Start Date is required when End Date is given.");
            }
            LocalDate startDate = parseDate(start, "Start Date");
            LocalDate endDate = parseDate(end, "End Date");
            if (endDate.isBefore(startDate)) {
                throw new IllegalArgumentException("End Date cannot be before Start Date.");
            }
            // ChronoUnit.DAYS.between gives the gap (01 -> 10 = 9), so we add 1.
            int total = (int) ChronoUnit.DAYS.between(startDate, endDate) + 1;
            return new Project(0, name, startDate, endDate, total);
        }
 
        // ---- Case 2: number of days only ----
        if (!days.isEmpty()) {
            int total;
            try {
                total = Integer.parseInt(days);
            } catch (NumberFormatException e) {
                throw new IllegalArgumentException("Number of days must be a whole number.");
            }
            if (total < 1 || total > 36500) {
                throw new IllegalArgumentException("Number of days must be between 1 and 36500.");
            }
            LocalDate startDate = start.isEmpty() ? LocalDate.now() : parseDate(start, "Start Date");
            LocalDate endDate = startDate.plusDays(total - 1);   // start day is day 1
            return new Project(0, name, startDate, endDate, total);
        }
 
        throw new IllegalArgumentException("Enter an End Date, or enter the Number of days.");
    }
 
    // Browser date fields send dates as "yyyy-MM-dd", which LocalDate.parse understands.
    private static LocalDate parseDate(String text, String fieldName) {
        try {
            return LocalDate.parse(text);
        } catch (DateTimeParseException e) {
            throw new IllegalArgumentException(fieldName + " is not a valid date (use yyyy-mm-dd).");
        }
    }
 
    // null-safe trim: a missing form field becomes ""
    private static String trim(String s) {
        return s == null ? "" : s.trim();
    }
}
```
 
### Html.java
 
`src/main/java/com/projectcalc/util/Html.java`
 
```java
package com.projectcalc.util;
 
/**
 * Makes text safe to print inside an HTML page.
 * Example: a project named  <script>alert(1)</script>
 * is shown as plain text instead of running as code (prevents XSS).
 */
public class Html {
 
    public static String escape(Object value) {
        if (value == null) {
            return "";
        }
        String s = value.toString();
        StringBuilder out = new StringBuilder(s.length());
        for (char ch : s.toCharArray()) {
            switch (ch) {
                case '<':  out.append("&lt;");   break;
                case '>':  out.append("&gt;");   break;
                case '&':  out.append("&amp;");  break;
                case '"':  out.append("&quot;"); break;
                case '\'': out.append("&#39;");  break;
                default:   out.append(ch);
            }
        }
        return out.toString();
    }
}
```
 
### ProjectDao.java
 
`src/main/java/com/projectcalc/dao/ProjectDao.java`
 
```java
package com.projectcalc.dao;
 
import com.projectcalc.db.DBConnection;
import com.projectcalc.model.Project;
 
import java.sql.Connection;
import java.sql.Date;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;
 
/**
 * DAO = Data Access Object. All SQL for the "projects" table is here.
 * Only PreparedStatement is used: the "?" values are sent separately from the SQL,
 * so user input can never change the SQL (no SQL injection).
 */
public class ProjectDao {
 
    /** Inserts one project and returns its new id. */
    public int save(Project p) throws SQLException {
        String sql = "INSERT INTO projects (project_name, start_date, end_date, total_days) "
                   + "VALUES (?, ?, ?, ?)";
 
        // try-with-resources: Connection, PreparedStatement and ResultSet
        // are closed automatically, even if an error happens.
        try (Connection con = DBConnection.getConnection();
             PreparedStatement ps = con.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS)) {
 
            ps.setString(1, p.getName());
            ps.setDate(2, Date.valueOf(p.getStartDate()));   // LocalDate -> java.sql.Date
            ps.setDate(3, Date.valueOf(p.getEndDate()));
            ps.setInt(4, p.getTotalDays());
            ps.executeUpdate();
 
            // Read the AUTO_INCREMENT id that MySQL gave to the new row.
            try (ResultSet keys = ps.getGeneratedKeys()) {
                return keys.next() ? keys.getInt(1) : 0;
            }
        }
    }
 
    // Columns of a project + its task counts.
    // LEFT JOIN keeps projects that have no tasks yet (their counts are 0).
    // GROUP BY p.id makes one result row per project.
    private static final String SELECT_WITH_PROGRESS =
            "SELECT p.id, p.project_name, p.start_date, p.end_date, p.total_days, "
          + "       COUNT(t.id) AS task_total, "
          + "       SUM(CASE WHEN t.done THEN 1 ELSE 0 END) AS task_done "
          + "FROM projects p LEFT JOIN tasks t ON t.project_id = p.id ";
 
    private static final String GROUP_BY =
            " GROUP BY p.id, p.project_name, p.start_date, p.end_date, p.total_days";
 
    /** Returns all saved projects (with task progress), newest first. */
    public List<Project> findAll() throws SQLException {
        String sql = SELECT_WITH_PROGRESS + GROUP_BY + " ORDER BY p.id DESC";
        List<Project> list = new ArrayList<>();
 
        try (Connection con = DBConnection.getConnection();
             PreparedStatement ps = con.prepareStatement(sql);
             ResultSet rs = ps.executeQuery()) {
 
            // Each row of the result becomes one Project object.
            while (rs.next()) {
                list.add(readProject(rs));
            }
        }
        return list;
    }
 
    /** Returns one project (with task progress), or null if the id does not exist. */
    public Project findById(int id) throws SQLException {
        String sql = SELECT_WITH_PROGRESS + "WHERE p.id = ?" + GROUP_BY;
 
        try (Connection con = DBConnection.getConnection();
             PreparedStatement ps = con.prepareStatement(sql)) {
 
            ps.setInt(1, id);
 
            try (ResultSet rs = ps.executeQuery()) {
                return rs.next() ? readProject(rs) : null;
            }
        }
    }
 
    // Turns the current result row into a Project object.
    private Project readProject(ResultSet rs) throws SQLException {
        Project p = new Project(
                rs.getInt("id"),
                rs.getString("project_name"),
                rs.getDate("start_date").toLocalDate(),   // java.sql.Date -> LocalDate
                rs.getDate("end_date").toLocalDate(),
                rs.getInt("total_days"));
        p.setTaskCounts(rs.getInt("task_total"), rs.getInt("task_done"));
        return p;
    }
}
```
 
### TaskDao.java
 
`src/main/java/com/projectcalc/dao/TaskDao.java`
 
```java
package com.projectcalc.dao;
 
import com.projectcalc.db.DBConnection;
import com.projectcalc.model.Task;
 
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;
 
/**
 * All SQL for the "tasks" table.
 * Same style as ProjectDao: PreparedStatement + try-with-resources.
 */
public class TaskDao {
 
    /** All tasks of one project, in the order they were added. */
    public List<Task> findByProject(int projectId) throws SQLException {
        String sql = "SELECT id, project_id, title, done FROM tasks WHERE project_id = ? ORDER BY id";
        List<Task> list = new ArrayList<>();
 
        try (Connection con = DBConnection.getConnection();
             PreparedStatement ps = con.prepareStatement(sql)) {
 
            ps.setInt(1, projectId);
 
            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) {
                    list.add(new Task(
                            rs.getInt("id"),
                            rs.getInt("project_id"),
                            rs.getString("title"),
                            rs.getBoolean("done")));
                }
            }
        }
        return list;
    }
 
    /** Adds a new task (not done yet) to a project. */
    public void add(int projectId, String title) throws SQLException {
        String sql = "INSERT INTO tasks (project_id, title, done) VALUES (?, ?, FALSE)";
 
        try (Connection con = DBConnection.getConnection();
             PreparedStatement ps = con.prepareStatement(sql)) {
 
            ps.setInt(1, projectId);
            ps.setString(2, title);
            ps.executeUpdate();
        }
    }
 
    /**
     * Marks a task done, or not done again (done = NOT done).
     * "AND project_id = ?" makes sure the task really belongs to this project.
     */
    public void toggle(int taskId, int projectId) throws SQLException {
        String sql = "UPDATE tasks SET done = NOT done WHERE id = ? AND project_id = ?";
 
        try (Connection con = DBConnection.getConnection();
             PreparedStatement ps = con.prepareStatement(sql)) {
 
            ps.setInt(1, taskId);
            ps.setInt(2, projectId);
            ps.executeUpdate();
        }
    }
}
```
 
### CalculateServlet.java
 
`src/main/java/com/projectcalc/servlet/CalculateServlet.java`
 
```java
package com.projectcalc.servlet;
 
import com.projectcalc.model.Project;
import com.projectcalc.util.DateCalculator;
 
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
 
import java.io.IOException;
 
/**
 * Task 1 - the calculator.
 * GET  /calculate -> shows the empty form
 * POST /calculate -> calculates and shows the form again with the result (summary + chart)
 *
 * @WebServlet tells Tomcat which URL belongs to this servlet (no web.xml entry needed).
 */
@WebServlet("/calculate")
public class CalculateServlet extends HttpServlet {
 
    // Path of the JSP page. It is inside WEB-INF, so the browser cannot open it directly;
    // only a servlet can forward to it.
    private static final String PAGE = "/WEB-INF/views/calculator.jsp";
 
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        // Fill the form with this capstone project as an example.
        req.setAttribute("name", "ICT Lab Resource Booking System");
        req.getRequestDispatcher(PAGE).forward(req, resp);
    }
 
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        req.setCharacterEncoding("UTF-8");
 
        // 1. Read the form fields (the "name" attributes of the <input> tags).
        String name  = req.getParameter("name");
        String start = req.getParameter("startDate");
        String end   = req.getParameter("endDate");
        String days  = req.getParameter("days");
 
        // 2. Send the typed values back, so the form keeps them after submit.
        req.setAttribute("name", name);
        req.setAttribute("startDate", start);
        req.setAttribute("endDate", end);
        req.setAttribute("days", days);
 
        // 3. Calculate. A wrong input gives an error message instead of a result.
        try {
            Project result = DateCalculator.calculate(name, start, end, days);
            req.setAttribute("result", result);
        } catch (IllegalArgumentException e) {
            req.setAttribute("error", e.getMessage());
        }
 
        // 4. Show the JSP page.
        req.getRequestDispatcher(PAGE).forward(req, resp);
    }
}
```
 
### SaveServlet.java
 
`src/main/java/com/projectcalc/servlet/SaveServlet.java`
 
```java
package com.projectcalc.servlet;
 
import com.projectcalc.dao.ProjectDao;
import com.projectcalc.model.Project;
import com.projectcalc.util.DateCalculator;
 
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
 
import java.io.IOException;
import java.sql.SQLException;
 
/**
 * Task 3 - the "Save" button.
 * POST /save -> saves the project into MySQL, then goes to the list page.
 */
@WebServlet("/save")
public class SaveServlet extends HttpServlet {
 
    private final ProjectDao dao = new ProjectDao();
 
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        req.setCharacterEncoding("UTF-8");
 
        // The Save form sends the start and end date of the result.
        // We calculate again here instead of trusting a "total days" value from the browser,
        // because anyone can change hidden form fields.
        Project p;
        try {
            p = DateCalculator.calculate(
                    req.getParameter("name"),
                    req.getParameter("startDate"),
                    req.getParameter("endDate"),
                    null);
        } catch (IllegalArgumentException e) {
            resp.sendError(HttpServletResponse.SC_BAD_REQUEST, e.getMessage());
            return;
        }
 
        int newId;
        try {
            newId = dao.save(p);
        } catch (SQLException e) {
            // ServletException makes Tomcat show an error page and write the cause to its log.
            throw new ServletException("Could not save the project. Is MySQL running?", e);
        }
 
        // Go to the new project's page, where the user can add tasks.
        // Redirect (not forward): if the user presses F5 there,
        // the browser does not send the Save form again (no duplicate rows).
        resp.sendRedirect(req.getContextPath() + "/project?id=" + newId + "&saved=1");
    }
}
```
 
### ProjectListServlet.java
 
`src/main/java/com/projectcalc/servlet/ProjectListServlet.java`
 
```java
package com.projectcalc.servlet;
 
import com.projectcalc.dao.ProjectDao;
 
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
 
import java.io.IOException;
import java.sql.SQLException;
 
/**
 * Task 3 - the page that lists all saved projects.
 * GET /projects -> reads every row from MySQL and shows them in a table (+ timeline chart).
 */
@WebServlet("/projects")
public class ProjectListServlet extends HttpServlet {
 
    private final ProjectDao dao = new ProjectDao();
 
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        try {
            req.setAttribute("projects", dao.findAll());
        } catch (SQLException e) {
            throw new ServletException("Could not load projects. Is MySQL running?", e);
        }
        // "saved=1" is added by SaveServlet, so the page can show "Project saved."
        req.setAttribute("saved", "1".equals(req.getParameter("saved")));
        req.getRequestDispatcher("/WEB-INF/views/projects.jsp").forward(req, resp);
    }
}
```
 
### ProjectDetailServlet.java
 
`src/main/java/com/projectcalc/servlet/ProjectDetailServlet.java`
 
```java
package com.projectcalc.servlet;
 
import com.projectcalc.dao.ProjectDao;
import com.projectcalc.dao.TaskDao;
import com.projectcalc.model.Project;
 
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
 
import java.io.IOException;
import java.sql.SQLException;
 
/**
 * One project's page: its tasks, the add-task form and the progress bar.
 *
 * GET  /project?id=5                          -> show project 5 and its tasks
 * POST /project  (action=add,    id, title)   -> add a task
 * POST /project  (action=toggle, id, taskId)  -> mark a task done / not done
 */
@WebServlet("/project")
public class ProjectDetailServlet extends HttpServlet {
 
    private final ProjectDao projectDao = new ProjectDao();
    private final TaskDao taskDao = new TaskDao();
 
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        int id = readInt(req.getParameter("id"));
 
        try {
            Project project = projectDao.findById(id);
            if (project == null) {
                resp.sendError(HttpServletResponse.SC_NOT_FOUND, "Project not found.");
                return;
            }
            req.setAttribute("project", project);
            req.setAttribute("tasks", taskDao.findByProject(id));
        } catch (SQLException e) {
            throw new ServletException("Could not load the project. Is MySQL running?", e);
        }
 
        // Messages after a redirect: saved=1 (from SaveServlet), error=title (empty task title)
        req.setAttribute("saved", "1".equals(req.getParameter("saved")));
        req.setAttribute("titleError", "title".equals(req.getParameter("error")));
        req.getRequestDispatcher("/WEB-INF/views/project.jsp").forward(req, resp);
    }
 
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        req.setCharacterEncoding("UTF-8");
 
        int projectId = readInt(req.getParameter("id"));
        String action = req.getParameter("action");
        String back = req.getContextPath() + "/project?id=" + projectId;
 
        try {
            if (projectDao.findById(projectId) == null) {
                resp.sendError(HttpServletResponse.SC_NOT_FOUND, "Project not found.");
                return;
            }
 
            if ("add".equals(action)) {
                String title = req.getParameter("title");
                title = (title == null) ? "" : title.trim();
 
                // Server-side check: title must not be empty or too long.
                if (title.isEmpty() || title.length() > 200) {
                    resp.sendRedirect(back + "&error=title");
                    return;
                }
                taskDao.add(projectId, title);
 
            } else if ("toggle".equals(action)) {
                taskDao.toggle(readInt(req.getParameter("taskId")), projectId);
 
            } else {
                resp.sendError(HttpServletResponse.SC_BAD_REQUEST, "Unknown action.");
                return;
            }
        } catch (SQLException e) {
            throw new ServletException("Could not update tasks. Is MySQL running?", e);
        }
 
        // Back to the same page (POST -> Redirect -> GET, so F5 does not repeat the action).
        resp.sendRedirect(back);
    }
 
    // Converts a text id to a number. A wrong value becomes 0, which no row has (-> "not found").
    private int readInt(String text) {
        try {
            return Integer.parseInt(text);
        } catch (NumberFormatException e) {
            return 0;
        }
    }
}
```
 
### calculator.jsp
 
`src/main/webapp/WEB-INF/views/calculator.jsp`
 
```jsp
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<%@ page import="com.projectcalc.model.Project, com.projectcalc.util.Html" %>
<%
    // Values put into the request by CalculateServlet (null when not set).
    String ctx    = request.getContextPath();
    String name   = (String) request.getAttribute("name");
    String start  = (String) request.getAttribute("startDate");
    String end    = (String) request.getAttribute("endDate");
    String days   = (String) request.getAttribute("days");
    String error  = (String) request.getAttribute("error");
    Project result = (Project) request.getAttribute("result");
    // Html.escape(...) is used for every value the user typed, so it is shown as text, never as HTML.
%>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Project Completion Calculator</title>
    <link rel="stylesheet" href="<%= ctx %>/css/style.css">
</head>
<body>
<main class="container">
 
    <nav>
        <a class="active" href="<%= ctx %>/calculate">Calculator</a>
        <a href="<%= ctx %>/projects">Saved Projects</a>
    </nav>
 
    <h1>Project Completion Calculator</h1>
    <p class="hint">Enter a Start Date and an End Date, <b>or</b> enter only the Number of days.
        Start and end day are both counted.</p>
 
    <%-- ===== Task 1: the form. It is sent to CalculateServlet (POST /calculate) ===== --%>
    <form method="post" action="<%= ctx %>/calculate" class="card">
        <label for="name">Project Name</label>
        <input id="name" name="name" type="text" maxlength="150" required value="<%= Html.escape(name) %>">
 
        <div class="row">
            <div>
                <label for="startDate">Start Date</label>
                <input id="startDate" name="startDate" type="date" value="<%= Html.escape(start) %>">
            </div>
            <div>
                <label for="endDate">End Date</label>
                <input id="endDate" name="endDate" type="date" value="<%= Html.escape(end) %>">
            </div>
        </div>
 
        <p class="or">— OR —</p>
 
        <label for="days">Number of days <span class="hint">(if Start Date is empty, today is used)</span></label>
        <input id="days" name="days" type="number" min="1" max="36500" value="<%= Html.escape(days) %>">
 
        <button type="submit">Calculate</button>
    </form>
 
    <%-- Error message from DateCalculator (wrong input) --%>
    <% if (error != null) { %>
        <p class="error" role="alert"><%= Html.escape(error) %></p>
    <% } %>
 
    <%-- ===== Result: only shown after a successful calculation ===== --%>
    <% if (result != null) { %>
        <section class="card">
            <h2>Result</h2>
 
            <%-- Task 2: the chart. JavaScript reads the data-* values from the canvas. --%>
            <div class="chart-box">
                <canvas id="timelineChart"
                        data-name="<%= Html.escape(result.getName()) %>"
                        data-start="<%= result.getStartEpochDay() %>"
                        data-end="<%= result.getEndEpochDay() %>"
                        data-days="<%= result.getTotalDays() %>"
                        data-summary="<%= Html.escape(result.getSummary()) %>"></canvas>
            </div>
 
            <%-- Text summary, always shown below the chart. --%>
            <p id="summary" class="summary"><%= Html.escape(result.getSummary()) %></p>
 
            <%-- Task 3: Save button. The values are sent to SaveServlet (POST /save). --%>
            <form method="post" action="<%= ctx %>/save">
                <input type="hidden" name="name" value="<%= Html.escape(result.getName()) %>">
                <input type="hidden" name="startDate" value="<%= result.getStartDate() %>">
                <input type="hidden" name="endDate" value="<%= result.getEndDate() %>">
                <button type="submit">Save</button>
            </form>
        </section>
    <% } %>
 
</main>
 
<%-- Chart.js from CDN, then our own script. If the CDN fails, our script shows the text instead. --%>
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.1/dist/chart.umd.js"></script>
<script src="<%= ctx %>/js/timeline.js"></script>
<script>
    // Draw the chart only when there is a result on the page.
    var canvas = document.getElementById('timelineChart');
    if (canvas) {
        drawTimeline(canvas, [{
            label: canvas.dataset.name,
            start: Number(canvas.dataset.start),
            end: Number(canvas.dataset.end),
            days: Number(canvas.dataset.days),
            summary: canvas.dataset.summary
        }]);
    }
</script>
</body>
</html>
```
 
### projects.jsp
 
`src/main/webapp/WEB-INF/views/projects.jsp`
 
```jsp
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<%@ page import="java.util.List, com.projectcalc.model.Project, com.projectcalc.util.Html" %>
<%
    // Values put into the request by ProjectListServlet.
    String ctx = request.getContextPath();
    @SuppressWarnings("unchecked")
    List<Project> projects = (List<Project>) request.getAttribute("projects");
    boolean saved = Boolean.TRUE.equals(request.getAttribute("saved"));
%>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Saved Projects</title>
    <link rel="stylesheet" href="<%= ctx %>/css/style.css">
</head>
<body>
<main class="container">
 
    <nav>
        <a href="<%= ctx %>/calculate">Calculator</a>
        <a class="active" href="<%= ctx %>/projects">Saved Projects</a>
    </nav>
 
    <h1>Saved Projects</h1>
 
    <% if (saved) { %>
        <p class="success" role="status">Project saved.</p>
    <% } %>
 
    <% if (projects.isEmpty()) { %>
        <p>No projects saved yet. <a href="<%= ctx %>/calculate">Calculate one</a>.</p>
    <% } else { %>
 
        <%-- Timeline of all saved projects (same chart script as the calculator) --%>
        <div class="card chart-box tall">
            <canvas id="timelineChart"></canvas>
        </div>
 
        <%-- The table. Each row also carries data-* values that the chart script reads. --%>
        <table class="card">
            <thead>
            <tr><th>#</th><th>Project Name</th><th>Start Date</th><th>End Date</th><th>Total Days</th><th>Progress</th></tr>
            </thead>
            <tbody>
            <% for (Project p : projects) { %>
                <tr class="project-row"
                    data-name="<%= Html.escape(p.getName()) %>"
                    data-start="<%= p.getStartEpochDay() %>"
                    data-end="<%= p.getEndEpochDay() %>"
                    data-days="<%= p.getTotalDays() %>"
                    data-summary="<%= Html.escape(p.getSummary()) %>">
                    <td><%= p.getId() %></td>
                    <%-- Click the name to open the project's tasks --%>
                    <td><a href="<%= ctx %>/project?id=<%= p.getId() %>"><%= Html.escape(p.getName()) %></a></td>
                    <td><%= p.getStartText() %></td>
                    <td><%= p.getEndText() %></td>
                    <td><%= p.getTotalDays() %></td>
                    <%-- Task progress, e.g. "75% (3/4)" with a small bar --%>
                    <td>
                        <div class="progress small"><div class="progress-fill" style="width: <%= p.getProgress() %>%"></div></div>
                        <%= p.getProgress() %>% (<%= p.getTaskDone() %>/<%= p.getTaskTotal() %>)
                    </td>
                </tr>
            <% } %>
            </tbody>
        </table>
    <% } %>
 
</main>
 
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.1/dist/chart.umd.js"></script>
<script src="<%= ctx %>/js/timeline.js"></script>
<script>
    var canvas = document.getElementById('timelineChart');
    if (canvas) {
        // Build the chart data from the table rows (oldest project at the top).
        var items = [];
        document.querySelectorAll('.project-row').forEach(function (row) {
            items.unshift({
                label: row.dataset.name,
                start: Number(row.dataset.start),
                end: Number(row.dataset.end),
                days: Number(row.dataset.days),
                summary: row.dataset.summary
            });
        });
        drawTimeline(canvas, items);
    }
</script>
</body>
</html>
```
 
### project.jsp
 
`src/main/webapp/WEB-INF/views/project.jsp`
 
```jsp
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<%@ page import="java.util.List, com.projectcalc.model.Project, com.projectcalc.model.Task, com.projectcalc.util.Html" %>
<%
    // Values put into the request by ProjectDetailServlet.
    String ctx = request.getContextPath();
    Project project = (Project) request.getAttribute("project");
    @SuppressWarnings("unchecked")
    List<Task> tasks = (List<Task>) request.getAttribute("tasks");
    boolean saved = Boolean.TRUE.equals(request.getAttribute("saved"));
    boolean titleError = Boolean.TRUE.equals(request.getAttribute("titleError"));
%>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title><%= Html.escape(project.getName()) %> - Tasks</title>
    <link rel="stylesheet" href="<%= ctx %>/css/style.css">
</head>
<body>
<main class="container">
 
    <nav>
        <a href="<%= ctx %>/calculate">Calculator</a>
        <a href="<%= ctx %>/projects">Saved Projects</a>
    </nav>
 
    <h1><%= Html.escape(project.getName()) %></h1>
    <p class="hint"><%= project.getStartText() %> to <%= project.getEndText() %>
        = <%= project.getTotalDays() %> <%= project.getTotalDays() == 1 ? "day" : "days" %></p>
 
    <% if (saved) { %>
        <p class="success" role="status">Project saved. Now add its tasks.</p>
    <% } %>
 
    <%-- ===== Progress: done tasks / all tasks ===== --%>
    <section class="card">
        <h2>Progress</h2>
        <div class="progress" role="progressbar" aria-valuemin="0" aria-valuemax="100"
             aria-valuenow="<%= project.getProgress() %>">
            <div class="progress-fill" style="width: <%= project.getProgress() %>%"></div>
        </div>
        <p class="summary">
            <%= project.getTaskDone() %> of <%= project.getTaskTotal() %> tasks done
            = <%= project.getProgress() %>% complete
        </p>
    </section>
 
    <%-- ===== Task list ===== --%>
    <section class="card">
        <h2>Tasks</h2>
 
        <% if (tasks.isEmpty()) { %>
            <p class="hint">No tasks yet. Add the first task below.</p>
        <% } else { %>
            <ul class="task-list">
            <% for (Task t : tasks) { %>
                <li class="<%= t.isDone() ? "done" : "" %>">
                    <span class="task-title"><%= Html.escape(t.getTitle()) %></span>
 
                    <%-- Each task has a small form: clicking the button marks it done (or undo). --%>
                    <form method="post" action="<%= ctx %>/project">
                        <input type="hidden" name="action" value="toggle">
                        <input type="hidden" name="id" value="<%= project.getId() %>">
                        <input type="hidden" name="taskId" value="<%= t.getId() %>">
                        <button type="submit" class="<%= t.isDone() ? "btn-light" : "" %>">
                            <%= t.isDone() ? "Undo" : "Mark done" %>
                        </button>
                    </form>
                </li>
            <% } %>
            </ul>
        <% } %>
 
        <%-- Add task form --%>
        <form method="post" action="<%= ctx %>/project" class="add-task">
            <input type="hidden" name="action" value="add">
            <input type="hidden" name="id" value="<%= project.getId() %>">
            <label for="title">New task</label>
            <div class="row">
                <input id="title" name="title" type="text" maxlength="200" required
                       placeholder="e.g. Design database">
                <button type="submit">Add Task</button>
            </div>
        </form>
 
        <% if (titleError) { %>
            <p class="error" role="alert">Task title is required (max 200 characters).</p>
        <% } %>
    </section>
 
</main>
</body>
</html>
```
 
### timeline.js
 
`src/main/webapp/js/timeline.js`
 
```javascript
/*
 * Task 2 - timeline (horizontal bar) chart with Chart.js.
 *
 * Each item looks like:
 *   { label: "Project X", start: 20393, end: 20402, days: 10, summary: "Project X: ... = 10 days" }
 * start/end are "days since 1970-01-01" (sent by the servlet), so we do not need a date library.
 *
 * If Chart.js did not load (no internet / CDN blocked) or drawing fails,
 * the canvas is replaced by the text summary.
 */
 
var DAY_MS = 24 * 60 * 60 * 1000;   // milliseconds in one day
 
// Turn "days since 1970" into text like "01 Nov"
function dayToText(epochDay) {
    return new Date(epochDay * DAY_MS).toLocaleDateString('en-GB', {
        day: '2-digit', month: 'short', timeZone: 'UTC'
    });
}
 
// Hide the chart and show the summary lines as plain text instead.
function showTextInstead(canvas, items) {
    var box = document.createElement('div');
    box.className = 'chart-fallback';
    box.innerHTML = '<p><b>Chart could not be shown. Summary:</b></p>';
    items.forEach(function (item) {
        var p = document.createElement('p');
        p.textContent = item.summary;     // textContent = safe, no HTML injection
        box.appendChild(p);
    });
    canvas.parentNode.replaceChild(box, canvas);
}
 
function drawTimeline(canvas, items) {
    // Chart.js missing -> text fallback
    if (typeof Chart === 'undefined') {
        showTextInstead(canvas, items);
        return;
    }
 
    try {
        // Smallest start and largest end of all projects.
        var firstStart = Math.min.apply(null, items.map(function (i) { return i.start; }));
        var lastEnd = Math.max.apply(null, items.map(function (i) { return i.end; }));
 
        // The x-axis shows whole months: from the 1st day of the start month
        // to the last day of the end month (at least 1 full month).
        var s = new Date(firstStart * DAY_MS);
        var e = new Date(lastEnd * DAY_MS);
        var minDay = Date.UTC(s.getUTCFullYear(), s.getUTCMonth(), 1) / DAY_MS;      // 1st of start month
        var maxDay = Date.UTC(e.getUTCFullYear(), e.getUTCMonth() + 1, 1) / DAY_MS;  // 1st of next month
 
        new Chart(canvas, {
            type: 'bar',
            data: {
                labels: items.map(function (i) { return i.label; }),
                datasets: [{
                    label: 'Duration',
                    // A "floating bar": [from, to]. +1 so the end day is drawn as a full day.
                    data: items.map(function (i) { return [i.start, i.end + 1]; }),
                    backgroundColor: '#2a6fdb',
                    borderRadius: 4,
                    barThickness: 28
                }]
            },
            options: {
                indexAxis: 'y',              // horizontal bars = timeline look
                responsive: true,
                maintainAspectRatio: false,
                plugins: {
                    legend: { display: false },
                    tooltip: {
                        callbacks: {
                            // Tooltip shows the same summary text as the page.
                            label: function (ctx) { return items[ctx.dataIndex].summary; }
                        }
                    }
                },
                scales: {
                    x: {
                        min: minDay,
                        max: maxDay,
                        // Put a tick every few days, counted from the 1st of the month
                        // (01, 04, 07 ...), so the labels are evenly spaced.
                        afterBuildTicks: function (axis) {
                            var step = Math.max(1, Math.ceil((maxDay - minDay) / 12));
                            var ticks = [];
                            for (var d = minDay; d <= maxDay; d += step) {
                                ticks.push({ value: d });
                            }
                            axis.ticks = ticks;
                        },
                        ticks: {
                            autoSkip: false,
                            callback: function (value) { return dayToText(value); }
                        },
                        title: { display: true, text: 'Date' }
                    }
                }
            }
        });
    } catch (e) {
        console.error(e);
        showTextInstead(canvas, items);
    }
}
```
 
### style.css
 
`src/main/webapp/css/style.css`
 
```css
/* Simple styles for both pages */
* { box-sizing: border-box; }
body { margin: 0; font-family: Arial, sans-serif; background: #f4f6f9; color: #222; }
.container { max-width: 760px; margin: 0 auto; padding: 24px 16px; }
 
nav { display: flex; gap: 8px; margin-bottom: 16px; }
nav a { padding: 8px 14px; border-radius: 6px; text-decoration: none; color: #2a6fdb; background: #fff; }
nav a.active { background: #2a6fdb; color: #fff; }
 
h1 { margin: 0 0 6px; font-size: 26px; }
h2 { margin-top: 0; font-size: 20px; }
.hint { color: #666; font-size: 14px; }
 
.card { background: #fff; border-radius: 8px; padding: 18px; margin: 16px 0;
        box-shadow: 0 1px 3px rgba(0,0,0,.1); }
 
label { display: block; font-weight: bold; margin: 10px 0 4px; }
input { width: 100%; padding: 8px; border: 1px solid #bbb; border-radius: 6px; font-size: 15px; }
.row { display: flex; gap: 12px; }
.row > div { flex: 1; }
.or { text-align: center; color: #888; margin: 14px 0 0; }
 
button { margin-top: 14px; padding: 10px 20px; border: 0; border-radius: 6px;
         background: #2a6fdb; color: #fff; font-size: 15px; cursor: pointer; }
button:hover { background: #1f58b3; }
 
.error   { background: #fde8e8; color: #9b1c1c; padding: 10px; border-radius: 6px; }
.success { background: #e6f6ea; color: #1e6b34; padding: 10px; border-radius: 6px; }
.summary { font-size: 18px; font-weight: bold; }
 
.chart-box { position: relative; height: 140px; }
.chart-box.tall { height: 320px; }
.chart-fallback { background: #fff8e1; padding: 10px; border-radius: 6px; }
.chart-fallback p { margin: 4px 0; }
 
table { width: 100%; border-collapse: collapse; }
th, td { padding: 8px; border-bottom: 1px solid #ddd; text-align: left; }
th { background: #f0f3f8; }
 
/* ---------- Tasks and progress ---------- */
.progress { height: 18px; background: #e3e8f0; border-radius: 9px; overflow: hidden; }
.progress.small { height: 8px; width: 90px; margin-bottom: 4px; }
.progress-fill { height: 100%; background: #22a35a; }
 
.task-list { list-style: none; padding: 0; margin: 0 0 8px; }
.task-list li { display: flex; align-items: center; justify-content: space-between; gap: 12px;
                padding: 8px 0; border-bottom: 1px solid #eee; }
.task-list li.done .task-title { text-decoration: line-through; color: #888; }
.task-list button { margin-top: 0; padding: 6px 12px; font-size: 14px; }
.btn-light { background: #e3e8f0; color: #333; }
.btn-light:hover { background: #cfd6e2; }
 
.add-task .row { align-items: center; }
.add-task .row button { margin-top: 0; flex: 0 0 auto; }
```
 
### index.jsp
 
`src/main/webapp/index.jsp`
 
```jsp
<%-- Start page: just sends the browser to the calculator servlet. --%>
<% response.sendRedirect(request.getContextPath() + "/calculate"); %>
```
 
### web.xml
 
`src/main/webapp/WEB-INF/web.xml`
 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
  Deployment descriptor. The servlets are registered with @WebServlet in the Java code,
  so this file only sets the start page.
-->
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee https://jakarta.ee/xml/ns/jakartaee/web-app_6_0.xsd"
         version="6.0">
 
    <display-name>Project Completion Calculator</display-name>
 
    <welcome-file-list>
        <welcome-file>index.jsp</welcome-file>
    </welcome-file-list>
</web-app>
```
 
### pom.xml
 
`pom.xml`
 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
  Maven build file.
  packaging = war  -> Maven builds target/project-calculator.war for Tomcat.
  No Spring, no Hibernate: only the Servlet API and the MySQL driver.
-->
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
 
    <groupId>com.projectcalc</groupId>
    <artifactId>project-calculator</artifactId>
    <version>1.0</version>
    <packaging>war</packaging>
 
    <properties>
        <maven.compiler.release>17</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
 
    <dependencies>
        <!-- Servlet API. "provided" = Tomcat already has it, so it is NOT copied into the WAR. -->
        <dependency>
            <groupId>jakarta.servlet</groupId>
            <artifactId>jakarta.servlet-api</artifactId>
            <version>6.0.0</version>
            <scope>provided</scope>
        </dependency>
 
        <!-- MySQL JDBC driver, used with plain DriverManager. -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <version>8.4.0</version>
        </dependency>
    </dependencies>
 
    <build>
        <!-- WAR file name = URL path on Tomcat: http://localhost:8080/project-calculator/ -->
        <finalName>project-calculator</finalName>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <version>3.4.0</version>
            </plugin>
        </plugins>
    </build>
</project>
```
 
## Compilation and Execution
 
### 1. Open the Project
 
Open the project directory in Visual Studio Code:
 
```text
C:\laragon\www\project-calculator
```
 
Open Laragon and click **Start All** to start MySQL.
 
### 2. Create the Database
 
Open the VS Code PowerShell terminal and run it once. Run it again after adding the task feature, to create the `tasks` table (existing data is kept):
 
```powershell
Get-Content db\schema.sql | & (Get-ChildItem C:\laragon\bin\mysql\*\bin\mysql.exe | Select-Object -First 1).FullName -u root
```
 
### 3. Build and Deploy the WAR
 
Stop an existing Tomcat instance before rebuilding. Run:
 
```powershell
mvn clean package
Copy-Item target\project-calculator.war "D:\Downloads\apache-tomcat-10.1.61\webapps\" -Force
```
 
### 4. Start Tomcat
 
```powershell
$TC = "D:\Downloads\apache-tomcat-10.1.61"
$env:JAVA_HOME = "C:\Program Files\Java\jdk-25"
$env:CATALINA_HOME = $TC
& "$TC\bin\catalina.bat" run
```
 
Keep the terminal open while using the application. Press **Ctrl + C** to stop Tomcat.
 
### 5. Open the Calculator
 
Visit [http://localhost:8080/project-calculator/](http://localhost:8080/project-calculator/).
 
Enter a project name and dates (or number of days), click **Calculate**, then click **Save**. The project page opens: type a task and click **Add Task**, then click **Mark done** on finished tasks to see the progress. Saved projects are shown at [http://localhost:8080/project-calculator/projects](http://localhost:8080/project-calculator/projects) and in HeidiSQL (Laragon → Database → `project_calc` → `projects` → Data).
 
## Working Procedure
 
1. `index.jsp` redirects the browser to `/calculate`.
2. `CalculateServlet.doGet()` shows the calculator form through `calculator.jsp`.
3. The form submits the project name, dates and number of days through a POST request.
4. `CalculateServlet.doPost()` reads the request using UTF-8 encoding.
5. `DateCalculator.calculate()` validates the input and calculates total days (`ChronoUnit.DAYS.between + 1`) or the end date (`start.plusDays(days - 1)`).
6. The servlet puts the `Project` result into the request and forwards to `calculator.jsp`, which shows the summary.
7. `timeline.js` draws the Chart.js timeline; if Chart.js is not loaded, it shows the text summary instead.
8. The Save button sends the result to `SaveServlet`, which calculates again and calls `ProjectDao.save()`.
9. `ProjectDao.save()` opens a connection with `DBConnection.getConnection()` and runs an `INSERT` with `PreparedStatement` inside try-with-resources.
10. `SaveServlet` redirects to `/project?id=…`, the new project's page.
11. `ProjectDetailServlet.doGet()` loads the project with `ProjectDao.findById()` (which also counts total and done tasks) and its tasks with `TaskDao.findByProject()`. `project.jsp` shows the task list, the progress bar and `X of Y tasks done = Z% complete`.
12. **Add Task** posts `action=add` to `ProjectDetailServlet`, which checks the title and calls `TaskDao.add()`.
13. **Mark done / Undo** posts `action=toggle`, and `TaskDao.toggle()` runs `UPDATE tasks SET done = NOT done`.
14. After each action the servlet redirects back to the project page, so the progress is recalculated from the database.
15. `ProjectListServlet` calls `ProjectDao.findAll()` and `projects.jsp` shows all projects with their progress and the timeline chart.
 
## Screenshots
 
<img width="461" height="415" alt="Screenshot 2026-11-07 190853" src="https://github.com/user-attachments/assets/d1ca5f20-9edb-41ab-8d2c-a762c778a983" />
<img width="441" height="417" alt="Screenshot 2026-11-07 190913" src="https://github.com/user-attachments/assets/af613e6f-5fb3-4cb8-b71e-c1bd31b25014" />
<img width="436" height="406" alt="Screenshot 2026-11-07 190930" src="https://github.com/user-attachments/assets/e42c34b3-aa19-4bbd-9d32-dac4a041141e" />
<img width="953" height="506" alt="Screenshot 2026-11-07 190940" src="https://github.com/user-attachments/assets/cc0b5fe3-7ee1-4f66-ad60-3ad2e8c7b6c9" />
 
<img width="307" height="385" alt="Screenshot 2026-11-08 180902" src="https://github.com/user-attachments/assets/ae476312-ebf9-44f4-9fba-bd5e1147e08a" />

 
## Conclusion
 
The application demonstrates form processing, request handling, date calculation, input validation, chart visualization, task tracking with progress percentage and database storage with two related tables using Java Servlet, JSP and manual JDBC on Apache Tomcat without any framework. `PreparedStatement` and try-with-resources keep database access safe and simple, and the text summary fallback keeps the result visible even when the chart cannot load.
 
 
