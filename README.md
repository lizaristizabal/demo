# demo

this is for my software engineering class

Hi my name is Liz and I am from Colombia

<?php
// find_interpreter.php

// Start the session
session_start();

// Check if user is logged in AND is a client
if (!isset($_SESSION['loggedin']) || $_SESSION['role'] != 'client') {
    header("Location: login.php");
    exit;
}

// 1. Include the database connection
require 'db_connect.php';

// 2. Check if form was submitted
$available_interpreters = [];
if ($_SERVER["REQUEST_METHOD"] == "POST") {

    // 3. Get all the data from the form
    $language_id = (int)$_POST['language_id'];
    $certification_id = (int)$_POST['certification_id'];
    $appointment_time_str = $_POST['appointment_time'];
    $duration = (int)$_POST['duration'];

    // Convert string time to a PHP DateTime object
    $appointment_start_time = new DateTime($appointment_time_str);
    
    // Calculate the requested end time
    $appointment_end_time = new DateTime($appointment_time_str);
    $appointment_end_time->add(new DateInterval('PT' . $duration . 'M'));

    // Format times for the database query
    $db_start_time = $appointment_start_time->format('Y-m-d H:i:s');
    $db_end_time = $appointment_end_time->format('Y-m-d H:i:s');

    // 4. Get ClientID from session (needed for booking)
    $client_user_id = $_SESSION['user_id'];
    $client_sql = "SELECT ClientID FROM Clients WHERE UserID = ?";
    $stmt = $conn->prepare($client_sql);
    $stmt->bind_param("i", $client_user_id);
    $stmt->execute();
    $result = $stmt->get_result();
    $client_id = $result->fetch_assoc()['ClientID'];
    $stmt->close();

    // 5. Save details to session for the next page (book_appointment.php)
    $_SESSION['booking_details'] = [
        'client_id' => $client_id,
        'language_id' => $language_id,
        'certification_id' => $certification_id,
        'appointment_time' => $db_start_time,
        'duration' => $duration
    ];

    /*
     * IMPROVEMENT #4 IMPLEMENTATION
     *
     * Instead of calculating appointment end times dynamically using:
     * DATE_ADD(app.DateTime, INTERVAL app.Duration MINUTE)
     *
     * We use a stored computed column "EndDateTime" in the Appointments table.
     * This allows MySQL to use indexes efficiently and improves conflict detection performance.
     */

    $sql = "SELECT
                i.InterpreterID, i.FirstName, i.LastName
            FROM Interpreters AS i
            INNER JOIN InterpreterLanguages AS il ON i.InterpreterID = il.InterpreterID
            INNER JOIN InterpreterCertifications AS ic ON i.InterpreterID = ic.InterpreterID
            INNER JOIN Availability AS a ON i.InterpreterID = a.InterpreterID
            
            LEFT JOIN Appointments AS app 
                ON i.InterpreterID = app.InterpreterID 
                AND app.Status != 'Cancelled'
                AND app.DateTime < ? 
                AND app.EndDateTime > ?  -- OPTIMIZED: using stored EndDateTime instead of DATE_ADD()
                
            WHERE
                il.LanguageID = ?
                AND ic.CertificationID = ?
                AND (
                    ? BETWEEN a.StartDateTime AND a.EndDateTime
                    AND
                    ? BETWEEN a.StartDateTime AND a.EndDateTime
                )
                AND app.AppointmentID IS NULL
                
            GROUP BY i.InterpreterID, i.FirstName, i.LastName";

    if ($stmt = $conn->prepare($sql)) {
        
        $stmt->bind_param("ssiiss",
            $db_end_time,
            $db_start_time,
            $language_id,
            $certification_id,
            $db_start_time,
            $db_end_time
        );

        $stmt->execute();
        $result = $stmt->get_result();
        
        while($row = $result->fetch_assoc()) {
            $available_interpreters[] = $row;
        }

        $stmt->close();
    }

    $conn->close();
}
?>

<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Find an Interpreter</title>

<style>
body { 
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
    margin: 0;
    background-color: #f4f7f6;
}

.navbar {
    background-color: #fff;
    padding: 1em;
    border-bottom: 1px solid #ddd;
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.navbar h1 { margin: 0; font-size: 1.5em; }

.navbar a { text-decoration: none; color: #007bff; font-weight: bold; }

.container { 
    max-width: 800px; 
    margin: 2em auto; 
}

.card {
    padding: 2em;
    background-color: #fff;
    border-radius: 8px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.05);
    margin-bottom: 2em;
}

.interpreter-list {
    list-style: none;
    padding: 0;
}

.interpreter-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1em;
    border: 1px solid #ddd;
    border-radius: 4px;
    margin-bottom: 1em;
}

.interpreter-item h3 {
    margin: 0;
}

.btn { 
    background-color: #28a745; 
    color: white; 
    padding: 0.75em 1.5em; 
    border: none; 
    border-radius: 4px;
    cursor: pointer;
    font-size: 1em;
    font-weight: bold;
    text-decoration: none;
}

.btn:hover {
    background-color: #218838;
}

.btn-link {
    background: none;
    border: none;
    color: #007bff;
    padding: 0;
    font-size: 1em;
    cursor: pointer;
    text-decoration: underline;
}
</style>
</head>

<body>

<div class="navbar">
<h1>Available Interpreters</h1>
<a href="client_dashboard.php">Back to Dashboard</a>
</div>

<div class="container">

<div class="card">

<?php if (empty($available_interpreters)): ?>

<h2>No Matches Found</h2>

<p>
Unfortunately, no interpreters match your specific criteria or are available at that time.
Please try a different time or set of requirements.
</p>

<a href="client_dashboard.php" class="btn-link">Try again</a>

<?php else: ?>

<h2>Available Interpreters for Your Request</h2>

<p>
The following interpreters are qualified and available for your selected time slot.
</p>

<ul class="interpreter-list">

<?php foreach ($available_interpreters as $interpreter): ?>

<li class="interpreter-item">

<h3>
<?php echo htmlspecialchars($interpreter['FirstName'] . ' ' . $interpreter['LastName']); ?>
</h3>

<form action="book_appointment.php" method="POST">

<input type="hidden" name="interpreter_id"
value="<?php echo $interpreter['InterpreterID']; ?>">

<button type="submit" class="btn">Book Now</button>

</form>

</li>

<?php endforeach; ?>

</ul>

<?php endif; ?>

</div>

</div>

</body>
</html>


