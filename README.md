import java.sql.*;
import java.util.Scanner;

public class VotingSystem {
    static final String URL = "jdbc:mysql://localhost:3306/votingdb";
    static final String USER = "root";
    static final String PASS = "1234";

    static Scanner sc = new Scanner(System.in);

    public static void main(String[] args) {
        try (Connection conn = DriverManager.getConnection(URL, USER, PASS)) {
            System.out.println("✅ Database connected successfully!");
        } catch (SQLException e) {
            System.out.println("❌ Failed to connect to the database.");
            e.printStackTrace();
            return;
        }

        while (true) {
            System.out.println("\n===== Online Voting System =====");
            System.out.println("1. Register Voter");
            System.out.println("2. Login & Vote");
            System.out.println("3. View Results");
            System.out.println("4. Admin Panel");
            System.out.println("5. Register Candidate");
            System.out.println("6. Exit");
            System.out.print("Choose an option: ");
            int choice = sc.nextInt();
            sc.nextLine(); // consume newline

            switch (choice) {
                case 1 -> registerVoter();
                case 2 -> loginAndVote();
                case 3 -> viewResults();
                case 4 -> adminPanel();
                case 5 -> registerCandidate();
                case 6 -> {
                    System.out.println("👋 Goodbye!");
                    return;
                }
                default -> System.out.println("❗ Invalid choice.");
            }
        }
    }

    static void registerVoter() {
        System.out.print("Enter Name: ");
        String name = sc.nextLine();
        System.out.print("Enter Email: ");
        String email = sc.nextLine();
        System.out.print("Enter Password: ");
        String pass = sc.nextLine();

        try (Connection conn = DriverManager.getConnection(URL, USER, PASS)) {
            PreparedStatement ps = conn.prepareStatement("INSERT INTO voters (name, email, password) VALUES (?, ?, ?)");
            ps.setString(1, name);
            ps.setString(2, email);
            ps.setString(3, pass);
            ps.executeUpdate();
            System.out.println("✅ Registered successfully!");
        } catch (SQLException e) {
            System.out.println("❌ Registration failed: " + e.getMessage());
        }
    }

    static void loginAndVote() {
        System.out.print("Enter Email: ");
        String email = sc.nextLine();
        System.out.print("Enter Password: ");
        String pass = sc.nextLine();

        try (Connection conn = DriverManager.getConnection(URL, USER, PASS)) {
            PreparedStatement ps = conn.prepareStatement("SELECT id FROM voters WHERE email=? AND password=?");
            ps.setString(1, email);
            ps.setString(2, pass);
            ResultSet rs = ps.executeQuery();

            if (rs.next()) {
                int voterId = rs.getInt("id");

                PreparedStatement checkVote = conn.prepareStatement("SELECT * FROM votes WHERE voter_id = ?");
                checkVote.setInt(1, voterId);
                ResultSet voteCheck = checkVote.executeQuery();

                if (voteCheck.next()) {
                    System.out.println("❌ You have already voted.");
                    return;
                }

                System.out.println("\n🔸 Candidates:");
                Statement stmt = conn.createStatement();
                ResultSet candidates = stmt.executeQuery("SELECT * FROM candidates");
                while (candidates.next()) {
                    System.out.println(candidates.getInt("id") + ". " + candidates.getString("name") +
                            " (" + candidates.getString("party") + ")");
                }

                System.out.print("Enter Candidate ID to vote: ");
                int cid = sc.nextInt();
                sc.nextLine();

                PreparedStatement castVote = conn.prepareStatement("INSERT INTO votes (voter_id, candidate_id) VALUES (?, ?)");
                castVote.setInt(1, voterId);
                castVote.setInt(2, cid);
                castVote.executeUpdate();

                System.out.println("✅ Vote cast successfully!");
            } else {
                System.out.println("❌ Invalid credentials.");
            }
        } catch (SQLException e) {
            System.out.println("❌ Error: " + e.getMessage());
        }
    }

    static void viewResults() {
        String query = """
                SELECT c.name, c.party, COUNT(v.id) AS vote_count
                FROM candidates c
                LEFT JOIN votes v ON c.id = v.candidate_id
                GROUP BY c.id
                ORDER BY vote_count DESC
                """;

        try (Connection conn = DriverManager.getConnection(URL, USER, PASS);
             Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery(query)) {

            System.out.println("\n📊 Voting Results:");
            while (rs.next()) {
                System.out.printf("%s (%s): %d votes%n",
                        rs.getString("name"),
                        rs.getString("party"),
                        rs.getInt("vote_count"));
            }
        } catch (SQLException e) {
            System.out.println("❌ Failed to load results: " + e.getMessage());
        }
    }

    static void adminPanel() {
        System.out.print("Enter Admin Username: ");
        String username = sc.nextLine();
        System.out.print("Enter Password: ");
        String password = sc.nextLine();

        try (Connection conn = DriverManager.getConnection(URL, USER, PASS)) {
            PreparedStatement ps = conn.prepareStatement("SELECT * FROM admins WHERE username = ? AND password = ?");
            ps.setString(1, username);
            ps.setString(2, password);
            ResultSet rs = ps.executeQuery();

            if (!rs.next()) {
                System.out.println("❌ Invalid admin credentials.");
                return;
            }

            while (true) {
                System.out.println("\n===== Admin Panel =====");
                System.out.println("1. View All Candidates");
                System.out.println("2. Add Candidate");
                System.out.println("3. View Results");
                System.out.println("4. Exit to Main Menu");
                System.out.print("Choose an option: ");
                int choice = sc.nextInt();
                sc.nextLine();

                switch (choice) {
                    case 1 -> viewCandidates();
                    case 2 -> addCandidate();
                    case 3 -> viewResults();
                    case 4 -> {
                        return;
                    }
                    default -> System.out.println("❗ Invalid choice.");
                }
            }
        } catch (SQLException e) {
            System.out.println("❌ Admin access error: " + e.getMessage());
        }
    }

    static void viewCandidates() {
        try (Connection conn = DriverManager.getConnection(URL, USER, PASS);
             Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery("SELECT * FROM candidates")) {

            System.out.println("\n🧑‍💼 Candidates List:");
            while (rs.next()) {
                System.out.printf("ID: %d | Name: %s | Party: %s%n",
                        rs.getInt("id"),
                        rs.getString("name"),
                        rs.getString("party"));
            }
        } catch (SQLException e) {
            System.out.println("❌ Error fetching candidates: " + e.getMessage());
        }
    }

    static void addCandidate() {
        System.out.print("Enter Candidate Name: ");
        String name = sc.nextLine();
        System.out.print("Enter Party: ");
        String party = sc.nextLine();

        try (Connection conn = DriverManager.getConnection(URL, USER, PASS)) {
            PreparedStatement ps = conn.prepareStatement("INSERT INTO candidates (name, party) VALUES (?, ?)");
            ps.setString(1, name);
            ps.setString(2, party);
            ps.executeUpdate();
            System.out.println("✅ Candidate added successfully.");
        } catch (SQLException e) {
            System.out.println("❌ Failed to add candidate: " + e.getMessage());
        }
    }

    static void registerCandidate() {
        System.out.print("Enter Candidate Name: ");
        String name = sc.nextLine();
        System.out.print("Enter Party Name: ");
        String party = sc.nextLine();

        try (Connection conn = DriverManager.getConnection(URL, USER, PASS)) {
            PreparedStatement ps = conn.prepareStatement("INSERT INTO candidates (name, party) VALUES (?, ?)");
            ps.setString(1, name);
            ps.setString(2, party);
            ps.executeUpdate();
            System.out.println("✅ Candidate registered successfully!");
        } catch (SQLException e) {
            System.out.println("❌ Failed to register candidate: " + e.getMessage());
        }
    }
}
