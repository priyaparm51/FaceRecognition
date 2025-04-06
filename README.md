# FaceRecognition
import cv2
import numpy as np
import mysql.connector
from mysql.connector import Error
import os
import json
import time
import datetime
import base64
import threading
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import csv
import pandas as pd
from flask import Flask, render_template_string, request, redirect, url_for, session, jsonify, flash, send_from_directory
from werkzeug.security import generate_password_hash, check_password_hash
from werkzeug.utils import secure_filename
import schedule
from functools import wraps

# Flask Application Configuration
app = Flask(__name__)
app.secret_key = 'your_secret_key'  # For session management
app.config['UPLOAD_FOLDER'] = 'uploads'
app.config['ALLOWED_EXTENSIONS'] = {'png', 'jpg', 'jpeg', 'csv', 'xlsx'}
app.config['MAX_CONTENT_LENGTH'] = 16 * 1024 * 1024  # 16 MB max-size

# Create uploads directory if it doesn't exist
os.makedirs(app.config['UPLOAD_FOLDER'], exist_ok=True)
os.makedirs('static', exist_ok=True)
os.makedirs('templates', exist_ok=True)

# Global variables for camera and face detection
camera = None
camera_thread = None
is_detecting = False
CAMERA_SOURCE = 0  # Default is laptop camera

# Database Configuration
DB_CONFIG = {
    'host': 'localhost',
    'user': 'root',
    'password': 'vn@vikasnavrang',
    'database': 'attendance_management'
}


# Function to initialize database
def initialize_database():
    try:
        connection = mysql.connector.connect(
            host=DB_CONFIG['host'],
            user=DB_CONFIG['user'],
            password=DB_CONFIG['password']
        )
        
        cursor = connection.cursor()
        
        # Create database if it doesn't exist
        cursor.execute(f"CREATE DATABASE IF NOT EXISTS {DB_CONFIG['database']}")
        print(f"Database {DB_CONFIG['database']} created or already exists")
        
        # Connect to the created database
        connection = mysql.connector.connect(**DB_CONFIG)
        cursor = connection.cursor()
        
        # Create users table
        cursor.execute('''
        CREATE TABLE IF NOT EXISTS users (
            id INT AUTO_INCREMENT PRIMARY KEY,
            username VARCHAR(50) UNIQUE NOT NULL,
            password VARCHAR(255) NOT NULL,
            email VARCHAR(100) UNIQUE NOT NULL,
            role ENUM('admin', 'hod', 'teacher', 'student') NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
        ''')
        
        # Create colleges table
        cursor.execute('''
        CREATE TABLE IF NOT EXISTS colleges (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            address VARCHAR(255),
            phone VARCHAR(20),
            email VARCHAR(100),
            website VARCHAR(100),
            user_id INT,
            FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
        )
        ''')
        
        # Create departments table
        cursor.execute('''
        CREATE TABLE IF NOT EXISTS departments (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            hod VARCHAR(100),
            college_id INT,
            FOREIGN KEY (college_id) REFERENCES colleges(id) ON DELETE CASCADE
        )
        ''')
        
        # Create branches table
        cursor.execute('''
        CREATE TABLE IF NOT EXISTS branches (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            department_id INT,
            FOREIGN KEY (department_id) REFERENCES departments(id) ON DELETE CASCADE
        )
        ''')
        
        # Create students table
        cursor.execute('''
        CREATE TABLE IF NOT EXISTS students (
            id INT AUTO_INCREMENT PRIMARY KEY,
            roll_number VARCHAR(20) UNIQUE NOT NULL,
            name VARCHAR(100) NOT NULL,
            email VARCHAR(100),
            phone VARCHAR(20),
            semester INT,
            branch_id INT,
            face_encoding BLOB,
            user_id INT,
            FOREIGN KEY (branch_id) REFERENCES branches(id) ON DELETE CASCADE,
            FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL
        )
        ''')
        
        # Create subjects table
        cursor.execute('''
        CREATE TABLE IF NOT EXISTS subjects (
            id INT AUTO_INCREMENT PRIMARY KEY,
            code VARCHAR(20) NOT NULL,
            name VARCHAR(100) NOT NULL,
            semester INT NOT NULL,
            branch_id INT,
            FOREIGN KEY (branch_id) REFERENCES branches(id) ON DELETE CASCADE
        )
        ''')
        
        # Create timetable table
        cursor.execute('''
        CREATE TABLE IF NOT EXISTS timetable (
            id INT AUTO_INCREMENT PRIMARY KEY,
            day_of_week ENUM('Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday') NOT NULL,
            start_time TIME NOT NULL,
            end_time TIME NOT NULL,
            subject_id INT,
            branch_id INT,
            semester INT,
            FOREIGN KEY (subject_id) REFERENCES subjects(id) ON DELETE CASCADE,
            FOREIGN KEY (branch_id) REFERENCES branches(id) ON DELETE CASCADE
        )
        ''')
        
        # Create attendance table
        cursor.execute('''
        CREATE TABLE IF NOT EXISTS attendance (
            id INT AUTO_INCREMENT PRIMARY KEY,
            student_id INT,
            subject_id INT,
            date DATE NOT NULL,
            time TIME NOT NULL,
            status ENUM('present', 'absent') DEFAULT 'present',
            FOREIGN KEY (student_id) REFERENCES students(id) ON DELETE CASCADE,
            FOREIGN KEY (subject_id) REFERENCES subjects(id) ON DELETE CASCADE
        )
        ''')
        
        # Create notifications table
        cursor.execute('''
        CREATE TABLE IF NOT EXISTS notifications (
            id INT AUTO_INCREMENT PRIMARY KEY,
            student_id INT,
            message TEXT NOT NULL,
            date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            is_read BOOLEAN DEFAULT FALSE,
            FOREIGN KEY (student_id) REFERENCES students(id) ON DELETE CASCADE
        )
        ''')
        
        # Create session settings table
        cursor.execute('''
        CREATE TABLE IF NOT EXISTS settings (
            id INT AUTO_INCREMENT PRIMARY KEY,
            user_id INT,
            setting_key VARCHAR(50) NOT NULL,
            setting_value TEXT,
            FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
        )
        ''')
        
        connection.commit()
        print("All tables created successfully")
        
        # Check if admin user exists, if not create one
        cursor.execute("SELECT COUNT(*) FROM users WHERE role='admin'")
        admin_count = cursor.fetchone()[0]
        
        if admin_count == 0:
            admin_password = generate_password_hash('admin123')
            cursor.execute('''
            INSERT INTO users (username, password, email, role)
            VALUES ('admin', %s, 'admin@example.com', 'admin')
            ''', (admin_password,))
            connection.commit()
            print("Default admin user created")
        
        connection.close()
        
    except Error as e:
        print(f"Error initializing database: {e}")


# Database Utility Functions
def get_db_connection():
    try:
        connection = mysql.connector.connect(**DB_CONFIG)
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def execute_query(query, params=None, fetchone=False, fetchall=False, commit=False):
    connection = get_db_connection()
    result = None
    
    if connection:
        try:
            cursor = connection.cursor(dictionary=True)
            cursor.execute(query, params or ())
            
            if fetchone:
                result = cursor.fetchone()
            elif fetchall:
                result = cursor.fetchall()
            
            if commit:
                connection.commit()
                result = cursor.lastrowid
                
        except Error as e:
            print(f"Query execution error: {e}")
            if commit:
                connection.rollback()
        finally:
            connection.close()
    
    return result

# Authentication and Authorization Decorators
def login_required(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if 'user_id' not in session:
            return redirect(url_for('login'))
        return f(*args, **kwargs)
    return decorated_function

def role_required(roles):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            if 'user_id' not in session or 'user_role' not in session:
                return redirect(url_for('login'))
            
            if session['user_role'] not in roles:
                return render_template_string('''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Unauthorized - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f5f5f5;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
            text-align: center;
        }
        .error-container {
            background-color: white;
            padding: 40px;
            border-radius: 8px;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
            max-width: 500px;
        }
        h1 {
            font-size: 72px;
            margin: 0;
            color: #ff9800;
        }
        h2 {
            margin: 10px 0 20px 0;
        }
        p {
            margin-bottom: 30px;
            color: #666;
        }
        a {
            display: inline-block;
            padding: 10px 20px;
            background-color: #4CAF50;
            color: white;
            text-decoration: none;
            border-radius: 4px;
        }
        a:hover {
            background-color: #45a049;
        }
    </style>
</head>
<body>
    <div class="error-container">
        <h1>403</h1>
        <h2>Unauthorized Access</h2>
        <p>You don't have permission to access this page. Please contact your administrator if you believe this is an error.</p>
        <a href="{{ url_for('dashboard') }}">Go to Dashboard</a>
    </div>
</body>
</html>'''),unauthorized_html
                
            return f(*args, **kwargs)
        return decorated_function
    return decorator

# Face Recognition Functions
def detect_faces(frame):
    """Detect faces in frame and return face locations"""
    try:
        gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        face_cascade = cv2.CascadeClassifier(cv2.data.haarcascades + 'haarcascade_frontalface_default.xml')
        faces = face_cascade.detectMultiScale(gray, 1.1, 5)
        return faces
    except Exception as e:
        print(f"Error detecting faces: {e}")
        return []

def encode_face(face_img):
    """Create a simplified face encoding"""
    try:
        # Resize to standard size
        face_img = cv2.resize(face_img, (100, 100))
        # Convert to grayscale
        gray = cv2.cvtColor(face_img, cv2.COLOR_BGR2GRAY)
        # Flatten and normalize
        encoding = gray.flatten().astype(np.float32) / 255.0
        return encoding
    except Exception as e:
        print(f"Error encoding face: {e}")
        return None

def compare_faces(known_encoding, face_encoding, tolerance=0.6):
    """Compare face encodings using a simple Euclidean distance metric"""
    try:
        if known_encoding is None or face_encoding is None:
            return False
            
        # Convert BLOB data to numpy array if needed
        if isinstance(known_encoding, bytes):
            known_encoding = np.frombuffer(known_encoding, dtype=np.float32)
            
        # Calculate Euclidean distance
        distance = np.linalg.norm(known_encoding - face_encoding)
        return distance < tolerance
    except Exception as e:
        print(f"Error comparing faces: {e}")
        return False

def camera_feed():
    """Function to run in a separate thread for continuous face detection"""
    global is_detecting, camera
    
    connection = get_db_connection()
    if not connection:
        print("Cannot connect to database in camera thread")
        return
        
    cursor = connection.cursor(dictionary=True)
    
    print("Starting camera feed thread")
    
    while is_detecting:
        try:
            # Check if we need to check attendance based on timetable
            now = datetime.datetime.now()
            current_day = now.strftime('%A')
            current_time = now.strftime('%H:%M:%S')
            
            # Find current class from timetable
            cursor.execute('''
            SELECT t.*, s.id as subject_id, s.name as subject_name, b.id as branch_id, b.name as branch_name 
            FROM timetable t
            JOIN subjects s ON t.subject_id = s.id
            JOIN branches b ON t.branch_id = b.id
            WHERE t.day_of_week = %s 
            AND %s BETWEEN DATE_ADD(t.start_time, INTERVAL 10 MINUTE) AND t.end_time
            ''', (current_day, current_time))
            
            current_classes = cursor.fetchall()
            
            # If no ongoing class, skip face detection
            if not current_classes:
                time.sleep(5)  # Check again after 5 seconds
                continue
                
            # If camera is not initialized, try to initialize
            if camera is None or not camera.isOpened():
                camera = cv2.VideoCapture(CAMERA_SOURCE)
                if not camera.isOpened():
                    print(f"Failed to open camera source {CAMERA_SOURCE}")
                    time.sleep(5)
                    continue
            
            # Read frame from camera
            ret, frame = camera.read()
            if not ret:
                print("Failed to read frame from camera")
                time.sleep(1)
                continue
                
            # Detect faces in frame
            faces = detect_faces(frame)
            
            # For each detected face, try to identify and mark attendance
            for (x, y, w, h) in faces:
                face_roi = frame[y:y+h, x:x+w]
                face_encoding = encode_face(face_roi)
                
                # For each current class, check all students
                for current_class in current_classes:
                    # Get all students in this branch and semester
                    cursor.execute('''
                    SELECT * FROM students
                    WHERE branch_id = %s AND semester = %s
                    ''', (current_class['branch_id'], current_class['semester']))
                    
                    students = cursor.fetchall()
                    
                    for student in students:
                        # Skip if no face encoding stored
                        if not student['face_encoding']:
                            continue
                            
                        # Convert stored encoding from BLOB
                        stored_encoding = np.frombuffer(student['face_encoding'], dtype=np.float32)
                        
                        # Compare face encodings
                        if compare_faces(stored_encoding, face_encoding):
                            # Check if attendance already marked for this class today
                            cursor.execute('''
                            SELECT * FROM attendance 
                            WHERE student_id = %s AND subject_id = %s AND date = CURDATE()
                            ''', (student['id'], current_class['subject_id']))
                            
                            if not cursor.fetchone():
                                # Mark attendance
                                cursor.execute('''
                                INSERT INTO attendance (student_id, subject_id, date, time, status)
                                VALUES (%s, %s, CURDATE(), %s, 'present')
                                ''', (student['id'], current_class['subject_id'], current_time))
                                
                                connection.commit()
                                
                                print(f"Marked attendance for student {student['name']} in {current_class['subject_name']}")
                                
                                # Check attendance percentage
                                cursor.execute('''
                                SELECT 
                                    COUNT(*) as total_classes,
                                    SUM(CASE WHEN status = 'present' THEN 1 ELSE 0 END) as attended_classes
                                FROM attendance
                                WHERE student_id = %s AND subject_id = %s
                                ''', (student['id'], current_class['subject_id']))
                                
                                attendance_data = cursor.fetchone()
                                
                                if attendance_data and attendance_data['total_classes'] > 0:
                                    attendance_percentage = (attendance_data['attended_classes'] / attendance_data['total_classes']) * 100
                                    
                                    # Send notification if attendance is below 30%
                                    if attendance_percentage < 30:
                                        cursor.execute('''
                                        INSERT INTO notifications (student_id, message)
                                        VALUES (%s, %s)
                                        ''', (student['id'], f"Your attendance in {current_class['subject_name']} is below 30%. Please attend classes regularly."))
                                        
                                        connection.commit()
            
            time.sleep(1)  # Small delay between processing frames
                
        except Exception as e:
            print(f"Error in camera thread: {e}")
            time.sleep(5)
            
    # Clean up
    if camera and camera.isOpened():
        camera.release()
        camera = None
        
    if connection:
        connection.close()
        
    print("Camera feed thread terminated")

# Start face detection in a separate thread
def start_face_detection(camera_source=0):
    global camera, camera_thread, is_detecting, CAMERA_SOURCE
    
    if is_detecting:
        return "Face detection already running"
        
    CAMERA_SOURCE = camera_source
    is_detecting = True
    
    camera_thread = threading.Thread(target=camera_feed)
    camera_thread.daemon = True
    camera_thread.start()
    
    return "Face detection started"

# Stop face detection thread
def stop_face_detection():
    global is_detecting, camera, camera_thread
    
    if not is_detecting:
        return "Face detection not running"
        
    is_detecting = False
    
    if camera_thread:
        camera_thread.join(timeout=5)
        camera_thread = None
        
    if camera and camera.isOpened():
        camera.release()
        camera = None
        
    return "Face detection stopped"
    
# Route for handling login
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        
        user = execute_query("SELECT * FROM users WHERE username = %s", (username,), fetchone=True)
        
        if user and check_password_hash(user['password'], password):
            session['user_id'] = user['id']
            session['username'] = user['username']
            session['user_role'] = user['role']
            
            return redirect(url_for('dashboard'))
        else:
            flash('Invalid username or password', 'error')
      
    return render_template_string('''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Login - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f1f1f1;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
        }
        .login-container {
            background-color: #fff;
            padding: 30px;
            border-radius: 8px;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
            width: 300px;
        }
        h2 {
            text-align: center;
            margin-bottom: 20px;
        }
        .input-group {
            margin-bottom: 20px;
        }
        label {
            font-size: 14px;
            color: #555;
            display: block;
            margin-bottom: 5px;
        }
        input {
            width: 100%;
            padding: 10px;
            font-size: 16px;
            border: 1px solid #ccc;
            border-radius: 4px;
        }
        button {
            width: 100%;
            padding: 10px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            font-size: 16px;
            cursor: pointer;
        }
        button:hover {
            background-color: #45a049;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
        .register-link {
            text-align: center;
            margin-top: 15px;
        }
    </style>
</head>
<body>
    <div class="login-container">
        <h2>Login to your account</h2>
        
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                <ul class="flash-messages">
                    {% for category, message in messages %}
                        <li class="flash-message {{ category }}">{{ message }}</li>
                    {% endfor %}
                </ul>
            {% endif %}
        {% endwith %}
        
        <form method="POST" action="{{ url_for('login') }}">
            <div class="input-group">
                <label for="username">Username</label>
                <input type="text" id="username" name="username" required>
            </div>
            <div class="input-group">
                <label for="password">Password</label>
                <input type="password" id="password" name="password" required>
            </div>
            <button type="submit">Login</button>
        </form>
        <div class="register-link">
            <a href="{{ url_for('register') }}">Don't have an account? Register</a>
        </div>
    </div>
</body>
</html>'''),login_html

# Route for registration
@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        email = request.form.get('email')
        role = request.form.get('role', 'hod')  # Default to HOD role
        
        # Check if username already exists
        existing_user = execute_query("SELECT id FROM users WHERE username = %s", (username,), fetchone=True)
        
        if existing_user:
            flash('Username already exists', 'error')
            return render_template_string('register_html')
            
        # Check if email already exists
        existing_email = execute_query("SELECT id FROM users WHERE email = %s", (email,), fetchone=True)
        
        if existing_email:
            flash('Email already exists', 'error')
            return render_template_string()
            
        # Hash the password
        hashed_password = generate_password_hash(password)
        
        # Insert new user
        user_id = execute_query(
            "INSERT INTO users (username, password, email, role) VALUES (%s, %s, %s, %s)",
            (username, hashed_password, email, role),
            commit=True
        )
        
        if user_id:
            flash('Registration successful! Please login.', 'success')
            return redirect(url_for('login'))
        else:
            flash('Registration failed', 'error')
            
    return render_template_string('''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Register - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f1f1f1;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
        }
        .register-container {
            background-color: #fff;
            padding: 30px;
            border-radius: 8px;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
            width: 300px;
        }
        h2 {
            text-align: center;
            margin-bottom: 20px;
        }
        .input-group {
            margin-bottom: 20px;
        }
        label {
            font-size: 14px;
            color: #555;
            display: block;
            margin-bottom: 5px;
        }
        input, select {
            width: 100%;
            padding: 10px;
            font-size: 16px;
            border: 1px solid #ccc;
            border-radius: 4px;
        }
        button {
            width: 100%;
            padding: 10px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            font-size: 16px;
            cursor: pointer;
        }
        button:hover {
            background-color: #45a049;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .login-link {
            text-align: center;
            margin-top: 15px;
        }
    </style>
</head>
<body>
    <div class="register-container">
        <h2>Create an account</h2>
        
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                <ul class="flash-messages">
                    {% for category, message in messages %}
                        <li class="flash-message {{ category }}">{{ message }}</li>
                    {% endfor %}
                </ul>
            {% endif %}
        {% endwith %}
        
        <form method="POST" action="{{ url_for('register') }}">
            <div class="input-group">
                <label for="username">Username</label>
                <input type="text" id="username" name="username" required>
            </div>
            <div class="input-group">
                <label for="password">Password</label>
                <input type="password" id="password" name="password" required>
            </div>
            <div class="input-group">
                <label for="email">Email</label>
                <input type="email" id="email" name="email" required>
            </div>
            <div class="input-group">
                <label for="role">Role</label>
                <select id="role" name="role">
                    <option value="admin">Admin</option>
                    <option value="hod" selected>HOD</option>
                    <option value="teacher">Teacher</option>
                    <option value="student">Student</option>
                </select>
            </div>
            <button type="submit">Register</button>
        </form>
        <div class="login-link">
            <a href="{{ url_for('login') }}">Already have an account? Login</a>
        </div>
    </div>
</body>
</html>'''),register_html

# Route for dashboard
@app.route('/dashboard')
@login_required
def dashboard():
    user_id = session.get('user_id')
    
    # Get college info if exists
    college = execute_query(
        "SELECT * FROM colleges WHERE user_id = %s",
        (user_id,),
        fetchone=True
    )
    
    # Get departments if college exists
    departments = []
    if college:
        departments = execute_query(
            "SELECT * FROM departments WHERE college_id = %s",
            (college['id'],),
            fetchall=True
        )
    
    # Get attendance statistics
    attendance_stats = {}
    
    # If HOD or admin, show overall stats
    if session.get('user_role') in ['admin', 'hod']:
        # Get total students
        student_count = execute_query(
            "SELECT COUNT(*) as count FROM students",
            fetchone=True
        )
        
        # Get present students today
        present_today = execute_query(
            "SELECT COUNT(DISTINCT student_id) as count FROM attendance WHERE date = CURDATE()",
            fetchone=True
        )
        
        # Get subject-wise attendance
        subject_attendance = execute_query(
            """
            SELECT s.name, 
                  COUNT(DISTINCT a.student_id) as present_count,
                  (SELECT COUNT(*) FROM students) as total_students
            FROM attendance a
            JOIN subjects s ON a.subject_id = s.id
            WHERE a.date = CURDATE()
            GROUP BY s.id
            """,
            fetchall=True
        )
        
        attendance_stats = {
            'total_students': student_count['count'] if student_count else 0,
            'present_today': present_today['count'] if present_today else 0,
            'subject_attendance': subject_attendance
        }
    
    return render_template_string('''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Dashboard - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .stats-container {
            display: flex;
            flex-wrap: wrap;
            gap: 20px;
        }
        .stat-card {
            flex: 1;
            min-width: 200px;
            background-color: #f9f9f9;
            border-left: 4px solid #4CAF50;
            padding: 15px;
            border-radius: 5px;
        }
        .stat-value {
            font-size: 24px;
            font-weight: bold;
            margin: 10px 0;
        }
        table {
            width: 100%;
            border-collapse: collapse;
        }
        table, th, td {
            border: 1px solid #ddd;
        }
        th, td {
            padding: 12px;
            text-align: left;
        }
        th {
            background-color: #4CAF50;
            color: white;
        }
        tr:nth-child(even) {
            background-color: #f2f2f2;
        }
        .camera-control {
            display: flex;
            gap: 10px;
            margin-top: 20px;
        }
        .btn {
            padding: 10px 15px;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        .btn-primary {
            background-color: #4CAF50;
            color: white;
        }
        .btn-danger {
            background-color: #f44336;
            color: white;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group select, .form-group input {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Attendance Overview</div>
            <div class="stats-container">
                <div class="stat-card">
                    <div>Total Students</div>
                    <div class="stat-value">{{ attendance_stats.total_students }}</div>
                </div>
                <div class="stat-card">
                    <div>Present Today</div>
                    <div class="stat-value">{{ attendance_stats.present_today }}</div>
                </div>
            </div>
        </div>

        <div class="card">
            <div class="card-header">Today's Attendance by Subject</div>
            <table>
                <thead>
                    <tr>
                        <th>Subject</th>
                        <th>Present</th>
                        <th>Total</th>
                        <th>Percentage</th>
                    </tr>
                </thead>
                <tbody>
                    {% for subject in attendance_stats.subject_attendance %}
                    <tr>
                        <td>{{ subject.name }}</td>
                        <td>{{ subject.present_count }}</td>
                        <td>{{ subject.total_students }}</td>
                        <td>{{ ((subject.present_count / subject.total_students) * 100)|round(1) }}%</td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>

        <div class="card">
            <div class="card-header">Camera Control</div>
            <form method="POST" action="{{ url_for('camera_control') }}">
                <div class="form-group">
                    <label for="camera_source">Camera Source</label>
                    <select id="camera_source" name="camera_source">
                        <option value="0">Default Camera</option>
                        <option value="1">Secondary Camera</option>
                    </select>
                </div>
                <div class="camera-control">
                    <button type="submit" name="action" value="start" class="btn btn-primary">Start Face Detection</button>
                    <button type="submit" name="action" value="stop" class="btn btn-danger">Stop Face Detection</button>
                </div>
            </form>
        </div>
    </div>
</body>
</html>'''
        
  ,college=college,departments=departments,attendance_stats=attendance_stats),dashboard_html

# Route for college setup
@app.route('/setup/college', methods=['GET', 'POST'])
@login_required
def setup_college():
    if request.method == 'POST':
        name = request.form.get('name')
        address = request.form.get('address')
        phone = request.form.get('phone')
        email = request.form.get('email')
        website = request.form.get('website')
        user_id = session.get('user_id')
        
        # Check if college already exists for this user
        existing_college = execute_query(
            "SELECT id FROM colleges WHERE user_id = %s",
            (user_id,),
            fetchone=True
        )
        
        if existing_college:
            # Update existing college
            execute_query(
                """
                UPDATE colleges 
                SET name = %s, address = %s, phone = %s, email = %s, website = %s
                WHERE user_id = %s
                """,
                (name, address, phone, email, website, user_id),
                commit=True
            )
            flash('College information updated successfully', 'success')
        else:
            # Insert new college
            college_id = execute_query(
                """
                INSERT INTO colleges (name, address, phone, email, website, user_id)
                VALUES (%s, %s, %s, %s, %s, %s)
                """,
                (name, address, phone, email, website, user_id),
                commit=True
            )
            
            if college_id:
                flash('College information saved successfully', 'success')
            else:
                flash('Failed to save college information', 'error')
                
        return redirect(url_for('dashboard'))
        
    # GET request - show form
    # Check if college already exists for this user
    college = execute_query(
        "SELECT * FROM colleges WHERE user_id = %s",
        (session.get('user_id'),),
        fetchone=True
    )
    
    return render_template_string('''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>College Setup - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group input, .form-group textarea {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .btn {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">College Information</div>
            
            {% with messages = get_flashed_messages(with_categories=true) %}
                {% if messages %}
                    <ul class="flash-messages">
                        {% for category, message in messages %}
                            <li class="flash-message {{ category }}">{{ message }}</li>
                        {% endfor %}
                    </ul>
                {% endif %}
            {% endwith %}
            
            <form method="POST" action="{{ url_for('setup_college') }}">
                <div class="form-group">
                    <label for="name">College Name</label>
                    <input type="text" id="name" name="name" value="{{ college.name if college else '' }}" required>
                </div>
                <div class="form-group">
                    <label for="address">Address</label>
                    <textarea id="address" name="address" rows="3" required>{{ college.address if college else '' }}</textarea>
                </div>
                <div class="form-group">
                    <label for="phone">Phone</label>
                    <input type="text" id="phone" name="phone" value="{{ college.phone if college else '' }}" required>
                </div>
                <div class="form-group">
                    <label for="email">Email</label>
                    <input type="email" id="email" name="email" value="{{ college.email if college else '' }}" required>
                </div>
                <div class="form-group">
                    <label for="website">Website</label>
                    <input type="url" id="website" name="website" value="{{ college.website if college else '' }}">
                </div>
                <button type="submit" class="btn">Save College Information</button>
            </form>
        </div>
    </div>
</body>
</html>''', college=college),setup_college_html

# Route for department setup
@app.route('/setup/department', methods=['GET', 'POST'])
@login_required
def setup_department():
    user_id = session.get('user_id')
    
    # Get college first
    college = execute_query(
        "SELECT * FROM colleges WHERE user_id = %s",
        (user_id,),
        fetchone=True
    )
    
    if not college:
        flash('Please set up college information first', 'error')
        return redirect(url_for('setup_college'))
    
    if request.method == 'POST':
        name = request.form.get('name')
        hod = request.form.get('hod')
        
        department_id = execute_query(
            """
            INSERT INTO departments (name, hod, college_id)
            VALUES (%s, %s, %s)
            """,
            (name, hod, college['id']),
            commit=True
        )
        
        if department_id:
            flash('Department added successfully', 'success')
        else:
            flash('Failed to add department', 'error')
            
        return redirect(url_for('setup_department'))
    
    # GET request - show form and existing departments
    departments = execute_query(
        "SELECT * FROM departments WHERE college_id = %s",
        (college['id'],),
        fetchall=True
    )
    
    return render_template_string('''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Departments - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group input {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .btn {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        table, th, td {
            border: 1px solid #ddd;
        }
        th, td {
            padding: 12px;
            text-align: left;
        }
        th {
            background-color: #4CAF50;
            color: white;
        }
        tr:nth-child(even) {
            background-color: #f2f2f2;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Add New Department</div>
            
            {% with messages = get_flashed_messages(with_categories=true) %}
                {% if messages %}
                    <ul class="flash-messages">
                        {% for category, message in messages %}
                            <li class="flash-message {{ category }}">{{ message }}</li>
                        {% endfor %}
                    </ul>
                {% endif %}
            {% endwith %}
            
            <form method="POST" action="{{ url_for('setup_department') }}">
                <div class="form-group">
                    <label for="name">Department Name</label>
                    <input type="text" id="name" name="name" required>
                </div>
                <div class="form-group">
                    <label for="hod">Head of Department</label>
                    <input type="text" id="hod" name="hod" required>
                </div>
                <button type="submit" class="btn">Add Department</button>
            </form>
        </div>

        <div class="card">
            <div class="card-header">Existing Departments</div>
            <table>
                <thead>
                    <tr>
                        <th>ID</th>
                        <th>Name</th>
                        <th>HOD</th>
                    </tr>
                </thead>
                <tbody>
                    {% for department in departments %}
                    <tr>
                        <td>{{ department.id }}</td>
                        <td>{{ department.name }}</td>
                        <td>{{ department.hod }}</td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
    </div>
</body>
</html>''', departments=departments),setup_department_html

# Route for branch setup
@app.route('/setup/branch', methods=['GET', 'POST'])
@login_required
def setup_branch():
    user_id = session.get('user_id')
    
    # Get departments
    departments = execute_query(
        """
        SELECT d.* FROM departments d
        JOIN colleges c ON d.college_id = c.id
        WHERE c.user_id = %s
        """,
        (user_id,),
        fetchall=True
    )
    
    if not departments:
        flash('Please set up departments first', 'error')
        return redirect(url_for('setup_department'))
    
    if request.method == 'POST':
        name = request.form.get('name')
        department_id = request.form.get('department_id')
        
        branch_id = execute_query(
            """
            INSERT INTO branches (name, department_id)
            VALUES (%s, %s)
            """,
            (name, department_id),
            commit=True
        )
        
        if branch_id:
            flash('Branch added successfully', 'success')
        else:
            flash('Failed to add branch', 'error')
            
        return redirect(url_for('setup_branch'))
    
    # GET request - show form and existing branches
    branches = execute_query(
        """
        SELECT b.*, d.name as department_name FROM branches b
        JOIN departments d ON b.department_id = d.id
        JOIN colleges c ON d.college_id = c.id
        WHERE c.user_id = %s
        """,
        (user_id,),
        fetchall=True
    )
    
    return render_template_string('''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Branches - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group input, .form-group select {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .btn {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        table, th, td {
            border: 1px solid #ddd;
        }
        th, td {
            padding: 12px;
            text-align: left;
        }
        th {
            background-color: #4CAF50;
            color: white;
        }
        tr:nth-child(even) {
            background-color: #f2f2f2;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Add New Branch</div>
            
            {% with messages = get_flashed_messages(with_categories=true) %}
                {% if messages %}
                    <ul class="flash-messages">
                        {% for category, message in messages %}
                            <li class="flash-message {{ category }}">{{ message }}</li>
                        {% endfor %}
                    </ul>
                {% endif %}
            {% endwith %}
            
            <form method="POST" action="{{ url_for('setup_branch') }}">
                <div class="form-group">
                    <label for="name">Branch Name</label>
                    <input type="text" id="name" name="name" required>
                </div>
                <div class="form-group">
                    <label for="department_id">Department</label>
                    <select id="department_id" name="department_id" required>
                        {% for department in departments %}
                        <option value="{{ department.id }}">{{ department.name }}</option>
                        {% endfor %}
                    </select>
                </div>
                <button type="submit" class="btn">Add Branch</button>
            </form>
        </div>

        <div class="card">
            <div class="card-header">Existing Branches</div>
            <table>
                <thead>
                    <tr>
                        <th>ID</th>
                        <th>Name</th>
                        <th>Department</th>
                    </tr>
                </thead>
                <tbody>
                    {% for branch in branches %}
                    <tr>
                        <td>{{ branch.id }}</td>
                        <td>{{ branch.name }}</td>
                        <td>{{ branch.department_name }}</td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
    </div>
</body>
</html>''', departments=departments, branches=branches),setup_branch_html

# Route for student registration
@app.route('/students', methods=['GET', 'POST'])
@login_required
@role_required(['admin', 'hod'])
def manage_students():
    user_id = session.get('user_id')
    
    # Get branches for this user's college
    branches = execute_query(
        """
        SELECT b.* FROM branches b
        JOIN departments d ON b.department_id = d.id
        JOIN colleges c ON d.college_id = c.id
        WHERE c.user_id = %s
        """,
        (user_id,),
        fetchall=True
    )
    
    if request.method == 'POST':
        if 'csv_file' in request.files:
            # Bulk upload via CSV
            file = request.files['csv_file']
            if file.filename.endswith('.csv'):
                try:
                    # Read CSV
                    students_df = pd.read_csv(file)
                    
                    # Process each student
                    for _, row in students_df.iterrows():
                        # Check if student already exists
                        existing_student = execute_query(
                            "SELECT id FROM students WHERE roll_number = %s",
                            (row['roll_number'],),
                            fetchone=True
                        )
                        
                        if not existing_student:
                            # Create student
                            execute_query(
                                """
                                INSERT INTO students (roll_number, name, email, phone, semester, branch_id)
                                VALUES (%s, %s, %s, %s, %s, %s)
                                """,
                                (row['roll_number'], row['name'], row.get('email', ''), 
                                 row.get('phone', ''), row['semester'], row['branch_id']),
                                commit=True
                            )
                    
                    flash('Students imported successfully', 'success')
                except Exception as e:
                    flash(f'Error importing students: {str(e)}', 'error')
            else:
                flash('Please upload a CSV file', 'error')
        else:
            # Single student addition
            roll_number = request.form.get('roll_number')
            name = request.form.get('name')
            email = request.form.get('email')
            phone = request.form.get('phone')
            semester = request.form.get('semester')
            branch_id = request.form.get('branch_id')
            
            # Check if student already exists
            existing_student = execute_query(
                "SELECT id FROM students WHERE roll_number = %s",
                (roll_number,),
                fetchone=True
            )
            
            if existing_student:
                flash('Student with this roll number already exists', 'error')
            else:
                student_id = execute_query(
                    """
                    INSERT INTO students (roll_number, name, email, phone, semester, branch_id)
                    VALUES (%s, %s, %s, %s, %s, %s)
                    """,
                    (roll_number, name, email, phone, semester, branch_id),
                    commit=True
                )
                
                if student_id:
                    flash('Student added successfully', 'success')
                else:
                    flash('Failed to add student', 'error')
                    
        return redirect(url_for('manage_students'))
    
    # GET request - show form and existing students
    students = execute_query(
        """
        SELECT s.*, b.name as branch_name FROM students s
        JOIN branches b ON s.branch_id = b.id
        JOIN departments d ON b.department_id = d.id
        JOIN colleges c ON d.college_id = c.id
        WHERE c.user_id = %s
        """,
        (user_id,),
        fetchall=True
    )
    
    return render_template_string('''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Students - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group input, .form-group select {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .btn {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        .btn-secondary {
            background-color: #6c757d;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        table, th, td {
            border: 1px solid #ddd;
        }
        th, td {
            padding: 12px;
            text-align: left;
        }
        th {
            background-color: #4CAF50;
            color: white;
        }
        tr:nth-child(even) {
            background-color: #f2f2f2;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
        .action-buttons a {
            color: #4CAF50;
            margin-right: 10px;
            text-decoration: none;
        }
        .csv-upload {
            margin-top: 20px;
            padding: 15px;
            border: 1px dashed #ccc;
            border-radius: 5px;
        }
        .tabs {
            display: flex;
            margin-bottom: 20px;
        }
        .tab {
            padding: 10px 20px;
            cursor: pointer;
            background-color: #f1f1f1;
            margin-right: 5px;
            border-radius: 5px 5px 0 0;
        }
        .tab.active {
            background-color: #4CAF50;
            color: white;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Manage Students</div>
            
            {% with messages = get_flashed_messages(with_categories=true) %}
                {% if messages %}
                    <ul class="flash-messages">
                        {% for category, message in messages %}
                            <li class="flash-message {{ category }}">{{ message }}</li>
                        {% endfor %}
                    </ul>
                {% endif %}
            {% endwith %}
            
            <div class="tabs">
                <div class="tab active">Add Student</div>
                <div class="tab">Bulk Upload</div>
            </div>

            <div id="add-student-form">
                <form method="POST" action="{{ url_for('manage_students') }}">
                    <div class="form-group">
                        <label for="roll_number">Roll Number</label>
                        <input type="text" id="roll_number" name="roll_number" required>
                    </div>
                    <div class="form-group">
                        <label for="name">Name</label>
                        <input type="text" id="name" name="name" required>
                    </div>
                    <div class="form-group">
                        <label for="email">Email</label>
                        <input type="email" id="email" name="email">
                    </div>
                    <div class="form-group">
                        <label for="phone">Phone</label>
                        <input type="text" id="phone" name="phone">
                    </div>
                    <div class="form-group">
                        <label for="semester">Semester</label>
                        <select id="semester" name="semester" required>
                            {% for i in range(1, 9) %}
                            <option value="{{ i }}">Semester {{ i }}</option>
                            {% endfor %}
                        </select>
                    </div>
                    <div class="form-group">
                        <label for="branch_id">Branch</label>
                        <select id="branch_id" name="branch_id" required>
                            {% for branch in branches %}
                            <option value="{{ branch.id }}">{{ branch.name }}</option>
                            {% endfor %}
                        </select>
                    </div>
                    <button type="submit" class="btn">Add Student</button>
                </form>
            </div>

            <div id="bulk-upload-form" style="display: none;">
                <div class="csv-upload">
                    <form method="POST" action="{{ url_for('manage_students') }}" enctype="multipart/form-data">
                        <div class="form-group">
                            <label for="csv_file">Upload CSV File</label>
                            <input type="file" id="csv_file" name="csv_file" accept=".csv" required>
                        </div>
                        <button type="submit" class="btn">Upload CSV</button>
                    </form>
                    <div style="margin-top: 15px;">
                        <a href="#" class="btn btn-secondary">Download Sample CSV</a>
                    </div>
                </div>
            </div>
        </div>

        <div class="card">
            <div class="card-header">Student List</div>
            <table>
                <thead>
                    <tr>
                        <th>Roll No.</th>
                        <th>Name</th>
                        <th>Semester</th>
                        <th>Branch</th>
                        <th>Face Registered</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    {% for student in students %}
                    <tr>
                        <td>{{ student.roll_number }}</td>
                        <td>{{ student.name }}</td>
                        <td>Semester {{ student.semester }}</td>
                        <td>{{ student.branch_name }}</td>
                        <td>{{ 'Yes' if student.face_encoding else 'No' }}</td>
                        <td class="action-buttons">
                            <a href="{{ url_for('capture_student_face', student_id=student.id) }}">Capture Face</a>
                            <a href="#">Edit</a>
                            <a href="#">Delete</a>
                        </td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
    </div>

    <script>
        document.querySelectorAll('.tab').forEach(tab => {
            tab.addEventListener('click', () => {
                document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
                tab.classList.add('active');
                
                if (tab.textContent === 'Add Student') {
                    document.getElementById('add-student-form').style.display = 'block';
                    document.getElementById('bulk-upload-form').style.display = 'none';
                } else {
                    document.getElementById('add-student-form').style.display = 'none';
                    document.getElementById('bulk-upload-form').style.display = 'block';
                }
            });
        });
    </script>
</body>
</html>''', branches=branches, students=students),students_html

# Route for capturing student face
@app.route('/student/capture_face/<int:student_id>', methods=['GET', 'POST'])
@login_required
@role_required(['admin', 'hod'])
def capture_student_face(student_id):
    student = execute_query(
        "SELECT * FROM students WHERE id = %s",
        (student_id,),
        fetchone=True
    )
    
    if not student:
        flash('Student not found', 'error')
        return redirect(url_for('manage_students'))
    
    if request.method == 'POST':
        try:
            # Get base64 image from request
            image_data = request.json.get('image').split(',')[1]
            image_bytes = base64.b64decode(image_data)
            
            # Convert to numpy array
            nparr = np.frombuffer(image_bytes, np.uint8)
            image = cv2.imdecode(nparr, cv2.IMREAD_COLOR)
            
            # Detect face
            faces = detect_faces(image)
            
            if len(faces) == 0:
                return jsonify({'success': False, 'message': 'No face detected'})
            
            if len(faces) > 1:
                return jsonify({'success': False, 'message': 'Multiple faces detected'})
                
            # Extract face
           # Continue from where we left off in capture_student_face()
            # Extract face
            x, y, w, h = faces[0]
            face_roi = image[y:y+h, x:x+w]
            
            # Encode face
            face_encoding = encode_face(face_roi)
            
            if face_encoding is None:
                return jsonify({'success': False, 'message': 'Face encoding failed'})
            
            # Convert to bytes for database storage
            face_blob = face_encoding.tobytes()
            
            # Update student record with face encoding
            execute_query(
                "UPDATE students SET face_encoding = %s WHERE id = %s",
                (face_blob, student_id),
                commit=True
            )
            
            return jsonify({'success': True, 'message': 'Face captured successfully'})
            
        except Exception as e:
            return jsonify({'success': False, 'message': f'Error: {str(e)}'})
    
    return render_template_string('''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Capture Face - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .student-info {
            display: flex;
            margin-bottom: 20px;
        }
        .student-details {
            margin-left: 20px;
        }
        .student-details h3 {
            margin: 0 0 10px 0;
        }
        .camera-container {
            position: relative;
            width: 640px;
            height: 480px;
            margin: 0 auto;
            border: 2px solid #ddd;
        }
        #video {
            width: 100%;
            height: 100%;
            background-color: #eee;
        }
        #canvas {
            display: none;
        }
        .controls {
            text-align: center;
            margin-top: 20px;
        }
        .btn {
            padding: 10px 20px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            font-size: 16px;
            margin: 0 10px;
        }
        .btn-danger {
            background-color: #f44336;
        }
        .btn-secondary {
            background-color: #6c757d;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
        #result-message {
            text-align: center;
            font-weight: bold;
            margin: 20px 0;
            padding: 10px;
            border-radius: 4px;
            display: none;
        }
        .success-message {
            background-color: #d4edda;
            color: #155724;
        }
        .error-message {
            background-color: #f8d7da;
            color: #721c24;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Capture Student Face</div>
            
            {% with messages = get_flashed_messages(with_categories=true) %}
                {% if messages %}
                    <ul class="flash-messages">
                        {% for category, message in messages %}
                            <li class="flash-message {{ category }}">{{ message }}</li>
                        {% endfor %}
                    </ul>
                {% endif %}
            {% endwith %}
            
            <div class="student-info">
                <div>
                    <div style="width: 120px; height: 120px; background-color: #eee; border-radius: 5px;"></div>
                </div>
                <div class="student-details">
                    <h3>{{ student.name }}</h3>
                    <p>Roll Number: {{ student.roll_number }}</p>
                    <p>Semester: {{ student.semester }}</p>
                    <p>Branch: {{ student.branch_name }}</p>
                </div>
            </div>

            <div id="result-message"></div>

            <div class="camera-container">
                <video id="video" autoplay></video>
                <canvas id="canvas"></canvas>
            </div>

            <div class="controls">
                <button id="capture-btn" class="btn">Capture</button>
                <button id="retry-btn" class="btn btn-secondary" style="display: none;">Retry</button>
                <button id="save-btn" class="btn" style="display: none;">Save</button>
                <a href="{{ url_for('manage_students') }}" class="btn btn-danger">Cancel</a>
            </div>
        </div>
    </div>

    <script>
        const video = document.getElementById('video');
        const canvas = document.getElementById('canvas');
        const captureBtn = document.getElementById('capture-btn');
        const retryBtn = document.getElementById('retry-btn');
        const saveBtn = document.getElementById('save-btn');
        const resultMessage = document.getElementById('result-message');
        
        let stream = null;
        let capturedImage = null;

        // Start camera
        async function startCamera() {
            try {
                stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: false });
                video.srcObject = stream;
            } catch (err) {
                console.error("Error accessing camera:", err);
                resultMessage.textContent = "Error accessing camera. Please make sure you have granted camera permissions.";
                resultMessage.className = "error-message";
                resultMessage.style.display = "block";
                captureBtn.disabled = true;
            }
        }

        // Capture image
        captureBtn.addEventListener('click', () => {
            canvas.width = video.videoWidth;
            canvas.height = video.videoHeight;
            canvas.getContext('2d').drawImage(video, 0, 0, canvas.width, canvas.height);
            
            capturedImage = canvas.toDataURL('image/png');
            video.style.display = 'none';
            canvas.style.display = 'block';
            
            captureBtn.style.display = 'none';
            retryBtn.style.display = 'inline-block';
            saveBtn.style.display = 'inline-block';
        });

        // Retry capture
        retryBtn.addEventListener('click', () => {
            video.style.display = 'block';
            canvas.style.display = 'none';
            
            captureBtn.style.display = 'inline-block';
            retryBtn.style.display = 'none';
            saveBtn.style.display = 'none';
            
            resultMessage.style.display = 'none';
        });

        // Save image
        saveBtn.addEventListener('click', async () => {
            try {
                const response = await fetch('/student/capture_face/{{ student.id }}', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify({ image: capturedImage })
                });

                const data = await response.json();
                
                if (data.success) {
                    resultMessage.textContent = data.message;
                    resultMessage.className = "success-message";
                    saveBtn.style.display = 'none';
                } else {
                    resultMessage.textContent = data.message;
                    resultMessage.className = "error-message";
                }
                
                resultMessage.style.display = 'block';
            } catch (err) {
                console.error("Error saving face:", err);
                resultMessage.textContent = "Error saving face data. Please try again.";
                resultMessage.className = "error-message";
                resultMessage.style.display = 'block';
            }
        });

        // Initialize camera when page loads
        window.addEventListener('DOMContentLoaded', startCamera);

        // Stop camera when leaving page
        window.addEventListener('beforeunload', () => {
            if (stream) {
                stream.getTracks().forEach(track => track.stop());
            }
        });
    </script>
</body>
</html>''', student=student),capture_face_html

# Route for subject management
@app.route('/subjects', methods=['GET', 'POST'])
@login_required
@role_required(['admin', 'hod'])
def manage_subjects():
    user_id = session.get('user_id')
    
    # Get branches for this user's college
    branches = execute_query(
        """
        SELECT b.* FROM branches b
        JOIN departments d ON b.department_id = d.id
        JOIN colleges c ON d.college_id = c.id
        WHERE c.user_id = %s
        """,
        (user_id,),
        fetchall=True
    )
    
    if request.method == 'POST':
        code = request.form.get('code')
        name = request.form.get('name')
        semester = request.form.get('semester')
        branch_id = request.form.get('branch_id')
        
        # Check if subject already exists
        existing_subject = execute_query(
            "SELECT id FROM subjects WHERE code = %s AND branch_id = %s",
            (code, branch_id),
            fetchone=True
        )
        
        if existing_subject:
            flash('Subject with this code already exists for this branch', 'error')
        else:
            subject_id = execute_query(
                """
                INSERT INTO subjects (code, name, semester, branch_id)
                VALUES (%s, %s, %s, %s)
                """,
                (code, name, semester, branch_id),
                commit=True
            )
            
            if subject_id:
                flash('Subject added successfully', 'success')
            else:
                flash('Failed to add subject', 'error')
                
        return redirect(url_for('manage_subjects'))
    
    # GET request - show form and existing subjects
    subjects = execute_query(
        """
        SELECT s.*, b.name as branch_name FROM subjects s
        JOIN branches b ON s.branch_id = b.id
        JOIN departments d ON b.department_id = d.id
        JOIN colleges c ON d.college_id = c.id
        WHERE c.user_id = %s
        """,
        (user_id,),
        fetchall=True
    )
    
    return render_template_string('''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Subjects - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group input, .form-group select {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .btn {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        table, th, td {
            border: 1px solid #ddd;
        }
        th, td {
            padding: 12px;
            text-align: left;
        }
        th {
            background-color: #4CAF50;
            color: white;
        }
        tr:nth-child(even) {
            background-color: #f2f2f2;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
        .action-buttons a {
            color: #4CAF50;
            margin-right: 10px;
            text-decoration: none;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Add New Subject</div>
            
            {% with messages = get_flashed_messages(with_categories=true) %}
                {% if messages %}
                    <ul class="flash-messages">
                        {% for category, message in messages %}
                            <li class="flash-message {{ category }}">{{ message }}</li>
                        {% endfor %}
                    </ul>
                {% endif %}
            {% endwith %}
            
            <form method="POST" action="{{ url_for('manage_subjects') }}">
                <div class="form-group">
                    <label for="code">Subject Code</label>
                    <input type="text" id="code" name="code" required>
                </div>
                <div class="form-group">
                    <label for="name">Subject Name</label>
                    <input type="text" id="name" name="name" required>
                </div>
                <div class="form-group">
                    <label for="semester">Semester</label>
                    <select id="semester" name="semester" required>
                        {% for i in range(1, 9) %}
                        <option value="{{ i }}">Semester {{ i }}</option>
                        {% endfor %}
                    </select>
                </div>
                <div class="form-group">
                    <label for="branch_id">Branch</label>
                    <select id="branch_id" name="branch_id" required>
                        {% for branch in branches %}
                        <option value="{{ branch.id }}">{{ branch.name }}</option>
                        {% endfor %}
                    </select>
                </div>
                <button type="submit" class="btn">Add Subject</button>
            </form>
        </div>

        <div class="card">
            <div class="card-header">Subject List</div>
            <table>
                <thead>
                    <tr>
                        <th>Code</th>
                        <th>Name</th>
                        <th>Semester</th>
                        <th>Branch</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    {% for subject in subjects %}
                    <tr>
                        <td>{{ subject.code }}</td>
                        <td>{{ subject.name }}</td>
                        <td>Semester {{ subject.semester }}</td>
                        <td>{{ subject.branch_name }}</td>
                        <td class="action-buttons">
                            <a href="#">Edit</a>
                            <a href="#">Delete</a>
                        </td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
    </div>
</body>
</html>''', branches=branches, subjects=subjects),subjects_html

# Route for timetable management
@app.route('/timetable', methods=['GET', 'POST'])
@login_required
@role_required(['admin', 'hod'])
def manage_timetable():
    user_id = session.get('user_id')
    
    # Get branches and subjects for this user's college
    branches = execute_query(
        """
        SELECT b.* FROM branches b
        JOIN departments d ON b.department_id = d.id
        JOIN colleges c ON d.college_id = c.id
        WHERE c.user_id = %s
        """,
        (user_id,),
        fetchall=True
    )
    
    subjects = execute_query(
        """
        SELECT s.* FROM subjects s
        JOIN branches b ON s.branch_id = b.id
        JOIN departments d ON b.department_id = d.id
        JOIN colleges c ON d.college_id = c.id
        WHERE c.user_id = %s
        """,
        (user_id,),
        fetchall=True
    )
    
    if request.method == 'POST':
        day_of_week = request.form.get('day_of_week')
        start_time = request.form.get('start_time')
        end_time = request.form.get('end_time')
        subject_id = request.form.get('subject_id')
        branch_id = request.form.get('branch_id')
        semester = request.form.get('semester')
        
        # Check for time conflicts
        conflict = execute_query(
            """
            SELECT id FROM timetable
            WHERE day_of_week = %s
            AND branch_id = %s
            AND semester = %s
            AND (
                (%s BETWEEN start_time AND end_time)
                OR (%s BETWEEN start_time AND end_time)
                OR (start_time BETWEEN %s AND %s)
                OR (end_time BETWEEN %s AND %s)
            )
            """,
            (day_of_week, branch_id, semester, 
             start_time, end_time, 
             start_time, end_time, start_time, end_time),
            fetchone=True
        )
        
        if conflict:
            flash('Time conflict with existing timetable entry', 'error')
        else:
            timetable_id = execute_query(
                """
                INSERT INTO timetable (day_of_week, start_time, end_time, subject_id, branch_id, semester)
                VALUES (%s, %s, %s, %s, %s, %s)
                """,
                (day_of_week, start_time, end_time, subject_id, branch_id, semester),
                commit=True
            )
            
            if timetable_id:
                flash('Timetable entry added successfully', 'success')
            else:
                flash('Failed to add timetable entry', 'error')
                
        return redirect(url_for('manage_timetable'))
    
    # GET request - show form and existing timetable
    timetable = execute_query(
        """
        SELECT t.*, s.name as subject_name, b.name as branch_name 
        FROM timetable t
        JOIN subjects s ON t.subject_id = s.id
        JOIN branches b ON t.branch_id = b.id
        JOIN departments d ON b.department_id = d.id
        JOIN colleges c ON d.college_id = c.id
        WHERE c.user_id = %s
        ORDER BY t.day_of_week, t.start_time
        """,
        (user_id,),
        fetchall=True
    )
    
    return render_template_string('''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Timetable - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group input, .form-group select {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .btn {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        table, th, td {
            border: 1px solid #ddd;
        }
        th, td {
            padding: 12px;
            text-align: left;
        }
        th {
            background-color: #4CAF50;
            color: white;
        }
        tr:nth-child(even) {
            background-color: #f2f2f2;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
        .action-buttons a {
            color: #4CAF50;
            margin-right: 10px;
            text-decoration: none;
        }
        .timetable-view {
            margin-top: 30px;
            overflow-x: auto;
        }
        .timetable {
            width: 100%;
            border-collapse: collapse;
        }
        .timetable th, .timetable td {
            border: 1px solid #ddd;
            padding: 8px;
            text-align: center;
        }
        .timetable th {
            background-color: #4CAF50;
            color: white;
        }
        .timetable td {
            height: 60px;
            vertical-align: top;
        }
        .timetable-class {
            background-color: #e8f5e9;
            border-radius: 3px;
            padding: 5px;
            margin: 2px 0;
            font-size: 12px;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Add Timetable Entry</div>
            
            {% with messages = get_flashed_messages(with_categories=true) %}
                {% if messages %}
                    <ul class="flash-messages">
                        {% for category, message in messages %}
                            <li class="flash-message {{ category }}">{{ message }}</li>
                        {% endfor %}
                    </ul>
                {% endif %}
            {% endwith %}
            
            <form method="POST" action="{{ url_for('manage_timetable') }}">
                <div class="form-group">
                    <label for="day_of_week">Day of Week</label>
                    <select id="day_of_week" name="day_of_week" required>
                        <option value="Monday">Monday</option>
                        <option value="Tuesday">Tuesday</option>
                        <option value="Wednesday">Wednesday</option>
                        <option value="Thursday">Thursday</option>
                        <option value="Friday">Friday</option>
                        <option value="Saturday">Saturday</option>
                    </select>
                </div>
                <div class="form-group">
                    <label for="start_time">Start Time</label>
                    <input type="time" id="start_time" name="start_time" required>
                </div>
                <div class="form-group">
                    <label for="end_time">End Time</label>
                    <input type="time" id="end_time" name="end_time" required>
                </div>
                <div class="form-group">
                    <label for="subject_id">Subject</label>
                    <select id="subject_id" name="subject_id" required>
                        {% for subject in subjects %}
                        <option value="{{ subject.id }}">{{ subject.name }} ({{ subject.code }})</option>
                        {% endfor %}
                    </select>
                </div>
                <div class="form-group">
                    <label for="branch_id">Branch</label>
                    <select id="branch_id" name="branch_id" required>
                        {% for branch in branches %}
                        <option value="{{ branch.id }}">{{ branch.name }}</option>
                        {% endfor %}
                    </select>
                </div>
                <div class="form-group">
                    <label for="semester">Semester</label>
                    <select id="semester" name="semester" required>
                        {% for i in range(1, 9) %}
                        <option value="{{ i }}">Semester {{ i }}</option>
                        {% endfor %}
                    </select>
                </div>
                <button type="submit" class="btn">Add Entry</button>
            </form>
        </div>

        <div class="card">
            <div class="card-header">Timetable Entries</div>
            <table>
                <thead>
                    <tr>
                        <th>Day</th>
                        <th>Time</th>
                        <th>Subject</th>
                        <th>Branch</th>
                        <th>Semester</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    {% for entry in timetable %}
                    <tr>
                        <td>{{ entry.day_of_week }}</td>
                        <td>{{ entry.start_time.strftime('%H:%M') }} - {{ entry.end_time.strftime('%H:%M') }}</td>
                        <td>{{ entry.subject_name }}</td>
                        <td>{{ entry.branch_name }}</td>
                        <td>Semester {{ entry.semester }}</td>
                        <td class="action-buttons">
                            <a href="#">Edit</a>
                            <a href="#">Delete</a>
                        </td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>

        <div class="card">
            <div class="card-header">Weekly Timetable View</div>
            <div class="timetable-view">
                <table class="timetable">
                    <thead>
                        <tr>
                            <th>Time</th>
                            <th>Monday</th>
                            <th>Tuesday</th>
                            <th>Wednesday</th>
                            <th>Thursday</th>
                            <th>Friday</th>
                            <th>Saturday</th>
                        </tr>
                    </thead>
                    <tbody>
                        {% for hour in range(8, 18) %}
                        <tr>
                            <td>{{ hour }}:00 - {{ hour+1 }}:00</td>
                            {% for day in ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'] %}
                            <td>
                                {% for entry in timetable %}
                                    {% if entry.day_of_week == day and hour >= entry.start_time.hour and hour < entry.end_time.hour %}
                                    <div class="timetable-class">
                                        {{ entry.subject_name }}<br>
                                        {{ entry.branch_name }} - Sem {{ entry.semester }}
                                    </div>
                                    {% endif %}
                                {% endfor %}
                            </td>
                            {% endfor %}
                        </tr>
                        {% endfor %}
                    </tbody>
                </table>
            </div>
        </div>
    </div>
</body>
</html>''', 
                         branches=branches, 
                         subjects=subjects, 
                         timetable=timetable),timetable_html

# Route for attendance reports
@app.route('/attendance/report', methods=['GET'])
@login_required
@role_required(['admin', 'hod', 'teacher'])
def attendance_report():
    user_id = session.get('user_id')
    role = session.get('user_role')
    
    # Get filter parameters
    branch_id = request.args.get('branch_id')
    semester = request.args.get('semester')
    subject_id = request.args.get('subject_id')
    start_date = request.args.get('start_date')
    end_date = request.args.get('end_date')
    
    # Base query
    query = """
    SELECT s.roll_number, s.name, 
           COUNT(a.id) as total_classes,
           SUM(CASE WHEN a.status = 'present' THEN 1 ELSE 0 END) as attended_classes,
           (SUM(CASE WHEN a.status = 'present' THEN 1 ELSE 0 END) / COUNT(a.id)) * 100 as percentage
    FROM students s
    LEFT JOIN attendance a ON s.id = a.student_id
    JOIN branches b ON s.branch_id = b.id
    JOIN departments d ON b.department_id = d.id
    JOIN colleges c ON d.college_id = c.id
    """
    
    # Conditions based on role
    if role == 'hod':
        query += " WHERE c.user_id = %s"
        params = [user_id]
    elif role == 'teacher':
        # Teachers might only see subjects they teach (need teacher-subject mapping table)
        query += " WHERE c.user_id = %s"
        params = [user_id]
    else:
        params = []
    
    # Add filters
    if branch_id:
        query += " AND s.branch_id = %s" if not params else " WHERE s.branch_id = %s"
        params.append(branch_id)
    
    if semester:
        query += " AND s.semester = %s"
        params.append(semester)
    
    if subject_id:
        query += " AND a.subject_id = %s"
        params.append(subject_id)
    
    if start_date:
        query += " AND a.date >= %s"
        params.append(start_date)
    
    if end_date:
        query += " AND a.date <= %s"
        params.append(end_date)
    
    query += " GROUP BY s.id"
    
    # Execute query
    attendance_data = execute_query(query, params, fetchall=True)
    
    # Get branches for filter dropdown
    branches = execute_query(
        """
        SELECT b.* FROM branches b
        JOIN departments d ON b.department_id = d.id
        JOIN colleges c ON d.college_id = c.id
        WHERE c.user_id = %s
        """,
        (user_id,),
        fetchall=True
    )
    
    # Get subjects for filter dropdown
    subjects = execute_query(
        """
        SELECT s.* FROM subjects s
        JOIN branches b ON s.branch_id = b.id
        JOIN departments d ON b.department_id = d.id
        JOIN colleges c ON d.college_id = c.id
        WHERE c.user_id = %s
        """,
        (user_id,),
        fetchall=True
    )
    
    return render_template_string('''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Attendance Report - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group input, .form-group select {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .btn {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        table, th, td {
            border: 1px solid #ddd;
        }
        th, td {
            padding: 12px;
            text-align: left;
        }
        th {
            background-color: #4CAF50;
            color: white;
        }
        tr:nth-child(even) {
            background-color: #f2f2f2;
        }
        .low-attendance {
            background-color: #ffebee;
        }
        .filter-form {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
            gap: 15px;
            margin-bottom: 20px;
        }
        .filter-actions {
            display: flex;
            justify-content: flex-end;
            gap: 10px;
        }
        .percentage-cell {
            font-weight: bold;
        }
        .percentage-high {
            color: #2e7d32;
        }
        .percentage-medium {
            color: #ff8f00;
        }
        .percentage-low {
            color: #c62828;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <!-- attendance_report.html continued -->
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Attendance Report</div>
            
            <form method="GET" action="{{ url_for('attendance_report') }}">
                <div class="filter-form">
                    <div class="form-group">
                        <label for="branch_id">Branch</label>
                        <select id="branch_id" name="branch_id">
                            <option value="">All Branches</option>
                            {% for branch in branches %}
                            <option value="{{ branch.id }}" {% if request.args.get('branch_id') == branch.id|string %}selected{% endif %}>{{ branch.name }}</option>
                            {% endfor %}
                        </select>
                    </div>
                    <div class="form-group">
                        <label for="semester">Semester</label>
                        <select id="semester" name="semester">
                            <option value="">All Semesters</option>
                            {% for i in range(1, 9) %}
                            <option value="{{ i }}" {% if request.args.get('semester') == i|string %}selected{% endif %}>Semester {{ i }}</option>
                            {% endfor %}
                        </select>
                    </div>
                    <div class="form-group">
                        <label for="subject_id">Subject</label>
                        <select id="subject_id" name="subject_id">
                            <option value="">All Subjects</option>
                            {% for subject in subjects %}
                            <option value="{{ subject.id }}" {% if request.args.get('subject_id') == subject.id|string %}selected{% endif %}>{{ subject.name }}</option>
                            {% endfor %}
                        </select>
                    </div>
                    <div class="form-group">
                        <label for="start_date">Start Date</label>
                        <input type="date" id="start_date" name="start_date" value="{{ request.args.get('start_date', '') }}">
                    </div>
                    <div class="form-group">
                        <label for="end_date">End Date</label>
                        <input type="date" id="end_date" name="end_date" value="{{ request.args.get('end_date', '') }}">
                    </div>
                </div>
                <div class="filter-actions">
                    <button type="submit" class="btn">Apply Filters</button>
                    <a href="{{ url_for('attendance_report') }}" class="btn" style="background-color: #6c757d;">Reset</a>
                </div>
            </form>

            <table>
                <thead>
                    <tr>
                        <th>Roll No.</th>
                        <th>Student Name</th>
                        <th>Classes Attended</th>
                        <th>Total Classes</th>
                        <th>Attendance %</th>
                    </tr>
                </thead>
                <tbody>
                    {% for student in attendance_data %}
                    <tr {% if (student.attended_classes / student.total_classes * 100) < 30 %}class="low-attendance"{% endif %}>
                        <td>{{ student.roll_number }}</td>
                        <td>{{ student.name }}</td>
                        <td>{{ student.attended_classes }}</td>
                        <td>{{ student.total_classes }}</td>
                        <td class="percentage-cell 
                            {% if (student.attended_classes / student.total_classes * 100) >= 75 %}percentage-high
                            {% elif (student.attended_classes / student.total_classes * 100) >= 30 %}percentage-medium
                            {% else %}percentage-low{% endif %}">
                            {{ "%.1f"|format((student.attended_classes / student.total_classes * 100) }}%
                        </td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
    </div>
</body>
</html>''',
                         attendance_data=attendance_data,
                         branches=branches,
                         subjects=subjects),attendance_report_html

# Route for camera control
@app.route('/camera/control', methods=['POST'])
@login_required
@role_required(['admin', 'hod'])
def camera_control():
    action = request.form.get('action')
    camera_source = int(request.form.get('camera_source', 0))
    
    if action == 'start':
        result = start_face_detection(camera_source)
        flash(result, 'success')
    elif action == 'stop':
        result = stop_face_detection()
        flash(result, 'success')
    else:
        flash('Invalid action', 'error')
    
    return redirect(url_for('dashboard'))

# Route for notifications
@app.route('/notifications')
@login_required
def notifications():
    user_id = session.get('user_id')
    role = session.get('user_role')
    
    if role == 'student':
        # Get student ID for this user
        student = execute_query(
            "SELECT id FROM students WHERE user_id = %s",
            (user_id,),
            fetchone=True
        )
        
        if student:
            notifications = execute_query(
                "SELECT * FROM notifications WHERE student_id = %s ORDER BY date DESC",
                (student['id'],),
                fetchall=True
            )
            
            # Mark as read
            execute_query(
                "UPDATE notifications SET is_read = TRUE WHERE student_id = %s",
                (student['id'],),
                commit=True
            )
        else:
            notifications = []
    else:
        notifications = []
    
    return render_template_string('''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Notifications - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .notification {
            padding: 15px;
            border-left: 4px solid #4CAF50;
            background-color: #f9f9f9;
            margin-bottom: 15px;
            border-radius: 4px;
        }
        .notification.unread {
            border-left-color: #f44336;
            background-color: #ffebee;
        }
        .notification-date {
            font-size: 12px;
            color: #666;
            margin-top: 5px;
        }
        .no-notifications {
            text-align: center;
            padding: 20px;
            color: #666;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Notifications</div>
            
            {% if notifications %}
                {% for notification in notifications %}
                <div class="notification {% if not notification.is_read %}unread{% endif %}">
                    <div>{{ notification.message }}</div>
                    <div class="notification-date">{{ notification.date.strftime('%Y-%m-%d %H:%M') }}</div>
                </div>
                {% endfor %}
            {% else %}
                <div class="no-notifications">No notifications to display</div>
            {% endif %}
        </div>
    </div>
</body>
</html>''', notifications=notifications),notifications_html

# Route for settings
@app.route('/settings', methods=['GET', 'POST'])
@login_required
def user_settings():
    user_id = session.get('user_id')
    
    if request.method == 'POST':
        current_password = request.form.get('current_password')
        new_password = request.form.get('new_password')
        confirm_password = request.form.get('confirm_password')
        email = request.form.get('email')
        
        # Get current user data
        user = execute_query(
            "SELECT * FROM users WHERE id = %s",
            (user_id,),
            fetchone=True
        )
        
        if not user:
            flash('User not found', 'error')
            return redirect(url_for('user_settings'))
        
        # Change password if requested
        if current_password and new_password and confirm_password:
            if not check_password_hash(user['password'], current_password):
                flash('Current password is incorrect', 'error')
            elif new_password != confirm_password:
                flash('New passwords do not match', 'error')
            else:
                hashed_password = generate_password_hash(new_password)
                execute_query(
                    "UPDATE users SET password = %s WHERE id = %s",
                    (hashed_password, user_id),
                    commit=True
                )
                flash('Password changed successfully', 'success')
        
        # Update email if changed
        if email and email != user['email']:
            execute_query(
                "UPDATE users SET email = %s WHERE id = %s",
                (email, user_id),
                commit=True
            )
            flash('Email updated successfully', 'success')
        
        return redirect(url_for('user_settings'))
    
    # GET request - show current settings
    user = execute_query(
        "SELECT * FROM users WHERE id = %s",
        (user_id,),
        fetchone=True
    )
    
    return render_template_string('''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Settings - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group input {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .btn {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
        .section-title {
            font-size: 16px;
            font-weight: bold;
            margin: 20px 0 10px 0;
            padding-bottom: 5px;
            border-bottom: 1px solid #eee;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">User Settings</div>
            
            {% with messages = get_flashed_messages(with_categories=true) %}
                {% if messages %}
                    <ul class="flash-messages">
                        {% for category, message in messages %}
                            <li class="flash-message {{ category }}">{{ message }}</li>
                        {% endfor %}
                    </ul>
                {% endif %}
            {% endwith %}
            
            <form method="POST" action="{{ url_for('user_settings') }}">
                <div class="section-title">Account Information</div>
                <div class="form-group">
                    <label for="email">Email</label>
                    <input type="email" id="email" name="email" value="{{ user.email }}" required>
                </div>

                <div class="section-title">Change Password</div>
                <div class="form-group">
                    <label for="current_password">Current Password</label>
                    <input type="password" id="current_password" name="current_password">
                </div>
                <div class="form-group">
                    <label for="new_password">New Password</label>
                    <input type="password" id="new_password" name="new_password">
                </div>
                <div class="form-group">
                    <label for="confirm_password">Confirm New Password</label>
                    <input type="password" id="confirm_password" name="confirm_password">
                </div>

                <button type="submit" class="btn">Save Changes</button>
            </form>
        </div>
    </div>
</body>
</html>''', user=user),settings_html



# Route for logout
@app.route('/logout')
@login_required
def logout():
    session.clear()
    flash('You have been logged out', 'success')
    return redirect(url_for('login'))

# Error handlers
@app.errorhandler(404)
def page_not_found(e):
    return render_template_string('''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Page Not Found - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f5f5f5;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
            text-align: center;
        }
        .error-container {
            background-color: white;
            padding: 40px;
            border-radius: 8px;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
            max-width: 500px;
        }
        h1 {
            font-size: 72px;
            margin: 0;
            color: #4CAF50;
        }
        h2 {
            margin: 10px 0 20px 0;
        }
        p {
            margin-bottom: 30px;
            color: #666;
        }
        a {
            display: inline-block;
            padding: 10px 20px;
            background-color: #4CAF50;
            color: white;
            text-decoration: none;
            border-radius: 4px;
        }
        a:hover {
            background-color: #45a049;
        }
    </style>
</head>
<body>
    <div class="error-container">
        <h1>404</h1>
        <h2>Page Not Found</h2>
        <p>The page you are looking for might have been removed, had its name changed, or is temporarily unavailable.</p>
        <a href="{{ url_for('dashboard') }}">Go to Dashboard</a>
    </div>
</body>
</html>'''), 404

@app.errorhandler(500)
def internal_server_error(e):
    return render_template_string('''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Server Error - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f5f5f5;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
            text-align: center;
        }
        .error-container {
            background-color: white;
            padding: 40px;
            border-radius: 8px;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
            max-width: 500px;
        }
        h1 {
            font-size: 72px;
            margin: 0;
            color: #f44336;
        }
        h2 {
            margin: 10px 0 20px 0;
        }
        p {
            margin-bottom: 30px;
            color: #666;
        }
        a {
            display: inline-block;
            padding: 10px 20px;
            background-color: #4CAF50;
            color: white;
            text-decoration: none;
            border-radius: 4px;
        }
        a:hover {
            background-color: #45a049;
        }
    </style>
</head>
<body>
    <div class="error-container">
        <h1>500</h1>
        <h2>Internal Server Error</h2>
        <p>Something went wrong on our end. We're working to fix the issue. Please try again later.</p>
        <a href="{{ url_for('dashboard') }}">Go to Dashboard</a>
    </div>
</body>
</html>'''), 500

# Background task for sending notifications
def check_attendance_and_notify():
    # Get students with attendance below 30%
    students = execute_query(
        """
        SELECT s.id, s.name, s.email, sub.name as subject_name, 
               COUNT(a.id) as total_classes,
               SUM(CASE WHEN a.status = 'present' THEN 1 ELSE 0 END) as attended_classes,
               (SUM(CASE WHEN a.status = 'present' THEN 1 ELSE 0 END) / COUNT(a.id) * 100 as percentage
        FROM students s
        JOIN attendance a ON s.id = a.student_id
        JOIN subjects sub ON a.subject_id = sub.id
        GROUP BY s.id, sub.id
        HAVING percentage < 30
        """,
        fetchall=True
    )
    
    for student in students:
        # Check if notification already sent today
        existing_notification = execute_query(
            """
            SELECT id FROM notifications 
            WHERE student_id = %s 
            AND message LIKE %s
            AND DATE(date) = CURDATE()
            """,
            (student['id'], f"%{student['subject_name']}%"),
            fetchone=True
        )
        
        if not existing_notification:
            # Add notification
            execute_query(
                """
                INSERT INTO notifications (student_id, message)
                VALUES (%s, %s)
                """,
                (student['id'], 
                 f"Your attendance in {student['subject_name']} is below 30% ({student['percentage']:.1f}%). Please attend classes regularly."),
                commit=True
            )
            
            # Send email if email exists
            if student.get('email'):
                send_notification_email(
                    student['email'],
                    "Low Attendance Alert",
                    f"Dear {student['name']},\n\nYour attendance in {student['subject_name']} is below 30% ({student['percentage']:.1f}%). Please attend classes regularly to avoid consequences.\n\nCollege Attendance System"
                )

# Function to send email notifications
def send_notification_email(to_email, subject, message):
    try:
        # Configure your SMTP server details
        smtp_server = "smtp.example.com"
        smtp_port = 587
        smtp_username = "your_email@example.com"
        smtp_password = "your_password"
        
        # Create message
        msg = MIMEMultipart()
        msg['From'] = smtp_username
        msg['To'] = to_email
        msg['Subject'] = subject
        
        # Add body
        msg.attach(MIMEText(message, 'plain'))
        
        # Connect to server and send
        server = smtplib.SMTP(smtp_server, smtp_port)
        server.starttls()
        server.login(smtp_username, smtp_password)
        server.send_message(msg)
        server.quit()
        
        return True
    except Exception as e:
        print(f"Error sending email: {e}")
        return False

# Schedule the notification task to run daily at 8 PM
schedule.every().day.at("20:00").do(check_attendance_and_notify)

# Function to run scheduled tasks in background
def run_scheduled_tasks():
    while True:
        schedule.run_pending()
        time.sleep(1)

# Start the scheduler thread
scheduler_thread = threading.Thread(target=run_scheduled_tasks)
scheduler_thread.daemon = True
scheduler_thread.start()



login_html = '''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Login - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f1f1f1;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
        }
        .login-container {
            background-color: #fff;
            padding: 30px;
            border-radius: 8px;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
            width: 300px;
        }
        h2 {
            text-align: center;
            margin-bottom: 20px;
        }
        .input-group {
            margin-bottom: 20px;
        }
        label {
            font-size: 14px;
            color: #555;
            display: block;
            margin-bottom: 5px;
        }
        input {
            width: 100%;
            padding: 10px;
            font-size: 16px;
            border: 1px solid #ccc;
            border-radius: 4px;
        }
        button {
            width: 100%;
            padding: 10px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            font-size: 16px;
            cursor: pointer;
        }
        button:hover {
            background-color: #45a049;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
        .register-link {
            text-align: center;
            margin-top: 15px;
        }
    </style>
</head>
<body>
    <div class="login-container">
        <h2>Login to your account</h2>
        
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                <ul class="flash-messages">
                    {% for category, message in messages %}
                        <li class="flash-message {{ category }}">{{ message }}</li>
                    {% endfor %}
                </ul>
            {% endif %}
        {% endwith %}
        
        <form method="POST" action="{{ url_for('login') }}">
            <div class="input-group">
                <label for="username">Username</label>
                <input type="text" id="username" name="username" required>
            </div>
            <div class="input-group">
                <label for="password">Password</label>
                <input type="password" id="password" name="password" required>
            </div>
            <button type="submit">Login</button>
        </form>
        <div class="register-link">
            <a href="{{ url_for('register') }}">Don't have an account? Register</a>
        </div>
    </div>
</body>
</html>'''

register_html = '''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Register - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f1f1f1;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
        }
        .register-container {
            background-color: #fff;
            padding: 30px;
            border-radius: 8px;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
            width: 300px;
        }
        h2 {
            text-align: center;
            margin-bottom: 20px;
        }
        .input-group {
            margin-bottom: 20px;
        }
        label {
            font-size: 14px;
            color: #555;
            display: block;
            margin-bottom: 5px;
        }
        input, select {
            width: 100%;
            padding: 10px;
            font-size: 16px;
            border: 1px solid #ccc;
            border-radius: 4px;
        }
        button {
            width: 100%;
            padding: 10px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            font-size: 16px;
            cursor: pointer;
        }
        button:hover {
            background-color: #45a049;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .login-link {
            text-align: center;
            margin-top: 15px;
        }
    </style>
</head>
<body>
    <div class="register-container">
        <h2>Create an account</h2>
        
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                <ul class="flash-messages">
                    {% for category, message in messages %}
                        <li class="flash-message {{ category }}">{{ message }}</li>
                    {% endfor %}
                </ul>
            {% endif %}
        {% endwith %}
        
        <form method="POST" action="{{ url_for('register') }}">
            <div class="input-group">
                <label for="username">Username</label>
                <input type="text" id="username" name="username" required>
            </div>
            <div class="input-group">
                <label for="password">Password</label>
                <input type="password" id="password" name="password" required>
            </div>
            <div class="input-group">
                <label for="email">Email</label>
                <input type="email" id="email" name="email" required>
            </div>
            <div class="input-group">
                <label for="role">Role</label>
                <select id="role" name="role">
                    <option value="admin">Admin</option>
                    <option value="hod" selected>HOD</option>
                    <option value="teacher">Teacher</option>
                    <option value="student">Student</option>
                </select>
            </div>
            <button type="submit">Register</button>
        </form>
        <div class="login-link">
            <a href="{{ url_for('login') }}">Already have an account? Login</a>
        </div>
    </div>
</body>
</html>'''

dashboard_html = '''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Dashboard - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .stats-container {
            display: flex;
            flex-wrap: wrap;
            gap: 20px;
        }
        .stat-card {
            flex: 1;
            min-width: 200px;
            background-color: #f9f9f9;
            border-left: 4px solid #4CAF50;
            padding: 15px;
            border-radius: 5px;
        }
        .stat-value {
            font-size: 24px;
            font-weight: bold;
            margin: 10px 0;
        }
        table {
            width: 100%;
            border-collapse: collapse;
        }
        table, th, td {
            border: 1px solid #ddd;
        }
        th, td {
            padding: 12px;
            text-align: left;
        }
        th {
            background-color: #4CAF50;
            color: white;
        }
        tr:nth-child(even) {
            background-color: #f2f2f2;
        }
        .camera-control {
            display: flex;
            gap: 10px;
            margin-top: 20px;
        }
        .btn {
            padding: 10px 15px;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        .btn-primary {
            background-color: #4CAF50;
            color: white;
        }
        .btn-danger {
            background-color: #f44336;
            color: white;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group select, .form-group input {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Attendance Overview</div>
            <div class="stats-container">
                <div class="stat-card">
                    <div>Total Students</div>
                    <div class="stat-value">{{ attendance_stats.total_students }}</div>
                </div>
                <div class="stat-card">
                    <div>Present Today</div>
                    <div class="stat-value">{{ attendance_stats.present_today }}</div>
                </div>
            </div>
        </div>

        <div class="card">
            <div class="card-header">Today's Attendance by Subject</div>
            <table>
                <thead>
                    <tr>
                        <th>Subject</th>
                        <th>Present</th>
                        <th>Total</th>
                        <th>Percentage</th>
                    </tr>
                </thead>
                <tbody>
                    {% for subject in attendance_stats.subject_attendance %}
                    <tr>
                        <td>{{ subject.name }}</td>
                        <td>{{ subject.present_count }}</td>
                        <td>{{ subject.total_students }}</td>
                        <td>{{ ((subject.present_count / subject.total_students) * 100)|round(1) }}%</td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>

        <div class="card">
            <div class="card-header">Camera Control</div>
            <form method="POST" action="{{ url_for('camera_control') }}">
                <div class="form-group">
                    <label for="camera_source">Camera Source</label>
                    <select id="camera_source" name="camera_source">
                        <option value="0">Default Camera</option>
                        <option value="1">Secondary Camera</option>
                    </select>
                </div>
                <div class="camera-control">
                    <button type="submit" name="action" value="start" class="btn btn-primary">Start Face Detection</button>
                    <button type="submit" name="action" value="stop" class="btn btn-danger">Stop Face Detection</button>
                </div>
            </form>
        </div>
    </div>
</body>
</html>'''

setup_college_html = '''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>College Setup - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group input, .form-group textarea {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .btn {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">College Information</div>
            
            {% with messages = get_flashed_messages(with_categories=true) %}
                {% if messages %}
                    <ul class="flash-messages">
                        {% for category, message in messages %}
                            <li class="flash-message {{ category }}">{{ message }}</li>
                        {% endfor %}
                    </ul>
                {% endif %}
            {% endwith %}
            
            <form method="POST" action="{{ url_for('setup_college') }}">
                <div class="form-group">
                    <label for="name">College Name</label>
                    <input type="text" id="name" name="name" value="{{ college.name if college else '' }}" required>
                </div>
                <div class="form-group">
                    <label for="address">Address</label>
                    <textarea id="address" name="address" rows="3" required>{{ college.address if college else '' }}</textarea>
                </div>
                <div class="form-group">
                    <label for="phone">Phone</label>
                    <input type="text" id="phone" name="phone" value="{{ college.phone if college else '' }}" required>
                </div>
                <div class="form-group">
                    <label for="email">Email</label>
                    <input type="email" id="email" name="email" value="{{ college.email if college else '' }}" required>
                </div>
                <div class="form-group">
                    <label for="website">Website</label>
                    <input type="url" id="website" name="website" value="{{ college.website if college else '' }}">
                </div>
                <button type="submit" class="btn">Save College Information</button>
            </form>
        </div>
    </div>
</body>
</html>'''

setup_department_html = '''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Departments - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group input {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .btn {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        table, th, td {
            border: 1px solid #ddd;
        }
        th, td {
            padding: 12px;
            text-align: left;
        }
        th {
            background-color: #4CAF50;
            color: white;
        }
        tr:nth-child(even) {
            background-color: #f2f2f2;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Add New Department</div>
            
            {% with messages = get_flashed_messages(with_categories=true) %}
                {% if messages %}
                    <ul class="flash-messages">
                        {% for category, message in messages %}
                            <li class="flash-message {{ category }}">{{ message }}</li>
                        {% endfor %}
                    </ul>
                {% endif %}
            {% endwith %}
            
            <form method="POST" action="{{ url_for('setup_department') }}">
                <div class="form-group">
                    <label for="name">Department Name</label>
                    <input type="text" id="name" name="name" required>
                </div>
                <div class="form-group">
                    <label for="hod">Head of Department</label>
                    <input type="text" id="hod" name="hod" required>
                </div>
                <button type="submit" class="btn">Add Department</button>
            </form>
        </div>

        <div class="card">
            <div class="card-header">Existing Departments</div>
            <table>
                <thead>
                    <tr>
                        <th>ID</th>
                        <th>Name</th>
                        <th>HOD</th>
                    </tr>
                </thead>
                <tbody>
                    {% for department in departments %}
                    <tr>
                        <td>{{ department.id }}</td>
                        <td>{{ department.name }}</td>
                        <td>{{ department.hod }}</td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
    </div>
</body>
</html>'''

setup_branch_html = '''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Branches - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group input, .form-group select {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .btn {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        table, th, td {
            border: 1px solid #ddd;
        }
        th, td {
            padding: 12px;
            text-align: left;
        }
        th {
            background-color: #4CAF50;
            color: white;
        }
        tr:nth-child(even) {
            background-color: #f2f2f2;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Add New Branch</div>
            
            {% with messages = get_flashed_messages(with_categories=true) %}
                {% if messages %}
                    <ul class="flash-messages">
                        {% for category, message in messages %}
                            <li class="flash-message {{ category }}">{{ message }}</li>
                        {% endfor %}
                    </ul>
                {% endif %}
            {% endwith %}
            
            <form method="POST" action="{{ url_for('setup_branch') }}">
                <div class="form-group">
                    <label for="name">Branch Name</label>
                    <input type="text" id="name" name="name" required>
                </div>
                <div class="form-group">
                    <label for="department_id">Department</label>
                    <select id="department_id" name="department_id" required>
                        {% for department in departments %}
                        <option value="{{ department.id }}">{{ department.name }}</option>
                        {% endfor %}
                    </select>
                </div>
                <button type="submit" class="btn">Add Branch</button>
            </form>
        </div>

        <div class="card">
            <div class="card-header">Existing Branches</div>
            <table>
                <thead>
                    <tr>
                        <th>ID</th>
                        <th>Name</th>
                        <th>Department</th>
                    </tr>
                </thead>
                <tbody>
                    {% for branch in branches %}
                    <tr>
                        <td>{{ branch.id }}</td>
                        <td>{{ branch.name }}</td>
                        <td>{{ branch.department_name }}</td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
    </div>
</body>
</html>'''


students_html = '''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Students - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group input, .form-group select {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .btn {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        .btn-secondary {
            background-color: #6c757d;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        table, th, td {
            border: 1px solid #ddd;
        }
        th, td {
            padding: 12px;
            text-align: left;
        }
        th {
            background-color: #4CAF50;
            color: white;
        }
        tr:nth-child(even) {
            background-color: #f2f2f2;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
        .action-buttons a {
            color: #4CAF50;
            margin-right: 10px;
            text-decoration: none;
        }
        .csv-upload {
            margin-top: 20px;
            padding: 15px;
            border: 1px dashed #ccc;
            border-radius: 5px;
        }
        .tabs {
            display: flex;
            margin-bottom: 20px;
        }
        .tab {
            padding: 10px 20px;
            cursor: pointer;
            background-color: #f1f1f1;
            margin-right: 5px;
            border-radius: 5px 5px 0 0;
        }
        .tab.active {
            background-color: #4CAF50;
            color: white;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Manage Students</div>
            
            {% with messages = get_flashed_messages(with_categories=true) %}
                {% if messages %}
                    <ul class="flash-messages">
                        {% for category, message in messages %}
                            <li class="flash-message {{ category }}">{{ message }}</li>
                        {% endfor %}
                    </ul>
                {% endif %}
            {% endwith %}
            
            <div class="tabs">
                <div class="tab active">Add Student</div>
                <div class="tab">Bulk Upload</div>
            </div>

            <div id="add-student-form">
                <form method="POST" action="{{ url_for('manage_students') }}">
                    <div class="form-group">
                        <label for="roll_number">Roll Number</label>
                        <input type="text" id="roll_number" name="roll_number" required>
                    </div>
                    <div class="form-group">
                        <label for="name">Name</label>
                        <input type="text" id="name" name="name" required>
                    </div>
                    <div class="form-group">
                        <label for="email">Email</label>
                        <input type="email" id="email" name="email">
                    </div>
                    <div class="form-group">
                        <label for="phone">Phone</label>
                        <input type="text" id="phone" name="phone">
                    </div>
                    <div class="form-group">
                        <label for="semester">Semester</label>
                        <select id="semester" name="semester" required>
                            {% for i in range(1, 9) %}
                            <option value="{{ i }}">Semester {{ i }}</option>
                            {% endfor %}
                        </select>
                    </div>
                    <div class="form-group">
                        <label for="branch_id">Branch</label>
                        <select id="branch_id" name="branch_id" required>
                            {% for branch in branches %}
                            <option value="{{ branch.id }}">{{ branch.name }}</option>
                            {% endfor %}
                        </select>
                    </div>
                    <button type="submit" class="btn">Add Student</button>
                </form>
            </div>

            <div id="bulk-upload-form" style="display: none;">
                <div class="csv-upload">
                    <form method="POST" action="{{ url_for('manage_students') }}" enctype="multipart/form-data">
                        <div class="form-group">
                            <label for="csv_file">Upload CSV File</label>
                            <input type="file" id="csv_file" name="csv_file" accept=".csv" required>
                        </div>
                        <button type="submit" class="btn">Upload CSV</button>
                    </form>
                    <div style="margin-top: 15px;">
                        <a href="#" class="btn btn-secondary">Download Sample CSV</a>
                    </div>
                </div>
            </div>
        </div>

        <div class="card">
            <div class="card-header">Student List</div>
            <table>
                <thead>
                    <tr>
                        <th>Roll No.</th>
                        <th>Name</th>
                        <th>Semester</th>
                        <th>Branch</th>
                        <th>Face Registered</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    {% for student in students %}
                    <tr>
                        <td>{{ student.roll_number }}</td>
                        <td>{{ student.name }}</td>
                        <td>Semester {{ student.semester }}</td>
                        <td>{{ student.branch_name }}</td>
                        <td>{{ 'Yes' if student.face_encoding else 'No' }}</td>
                        <td class="action-buttons">
                            <a href="{{ url_for('capture_student_face', student_id=student.id) }}">Capture Face</a>
                            <a href="#">Edit</a>
                            <a href="#">Delete</a>
                        </td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
    </div>

    <script>
        document.querySelectorAll('.tab').forEach(tab => {
            tab.addEventListener('click', () => {
                document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
                tab.classList.add('active');
                
                if (tab.textContent === 'Add Student') {
                    document.getElementById('add-student-form').style.display = 'block';
                    document.getElementById('bulk-upload-form').style.display = 'none';
                } else {
                    document.getElementById('add-student-form').style.display = 'none';
                    document.getElementById('bulk-upload-form').style.display = 'block';
                }
            });
        });
    </script>
</body>
</html>'''

capture_student_face_html = '''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Capture Face - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .student-info {
            display: flex;
            margin-bottom: 20px;
        }
        .student-details {
            margin-left: 20px;
        }
        .student-details h3 {
            margin: 0 0 10px 0;
        }
        .camera-container {
            position: relative;
            width: 640px;
            height: 480px;
            margin: 0 auto;
            border: 2px solid #ddd;
        }
        #video {
            width: 100%;
            height: 100%;
            background-color: #eee;
        }
        #canvas {
            display: none;
        }
        .controls {
            text-align: center;
            margin-top: 20px;
        }
        .btn {
            padding: 10px 20px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            font-size: 16px;
            margin: 0 10px;
        }
        .btn-danger {
            background-color: #f44336;
        }
        .btn-secondary {
            background-color: #6c757d;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
        #result-message {
            text-align: center;
            font-weight: bold;
            margin: 20px 0;
            padding: 10px;
            border-radius: 4px;
            display: none;
        }
        .success-message {
            background-color: #d4edda;
            color: #155724;
        }
        .error-message {
            background-color: #f8d7da;
            color: #721c24;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Capture Student Face</div>
            
            {% with messages = get_flashed_messages(with_categories=true) %}
                {% if messages %}
                    <ul class="flash-messages">
                        {% for category, message in messages %}
                            <li class="flash-message {{ category }}">{{ message }}</li>
                        {% endfor %}
                    </ul>
                {% endif %}
            {% endwith %}
            
            <div class="student-info">
                <div>
                    <div style="width: 120px; height: 120px; background-color: #eee; border-radius: 5px;"></div>
                </div>
                <div class="student-details">
                    <h3>{{ student.name }}</h3>
                    <p>Roll Number: {{ student.roll_number }}</p>
                    <p>Semester: {{ student.semester }}</p>
                    <p>Branch: {{ student.branch_name }}</p>
                </div>
            </div>

            <div id="result-message"></div>

            <div class="camera-container">
                <video id="video" autoplay></video>
                <canvas id="canvas"></canvas>
            </div>

            <div class="controls">
                <button id="capture-btn" class="btn">Capture</button>
                <button id="retry-btn" class="btn btn-secondary" style="display: none;">Retry</button>
                <button id="save-btn" class="btn" style="display: none;">Save</button>
                <a href="{{ url_for('manage_students') }}" class="btn btn-danger">Cancel</a>
            </div>
        </div>
    </div>

    <script>
        const video = document.getElementById('video');
        const canvas = document.getElementById('canvas');
        const captureBtn = document.getElementById('capture-btn');
        const retryBtn = document.getElementById('retry-btn');
        const saveBtn = document.getElementById('save-btn');
        const resultMessage = document.getElementById('result-message');
        
        let stream = null;
        let capturedImage = null;

        // Start camera
        async function startCamera() {
            try {
                stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: false });
                video.srcObject = stream;
            } catch (err) {
                console.error("Error accessing camera:", err);
                resultMessage.textContent = "Error accessing camera. Please make sure you have granted camera permissions.";
                resultMessage.className = "error-message";
                resultMessage.style.display = "block";
                captureBtn.disabled = true;
            }
        }

        // Capture image
        captureBtn.addEventListener('click', () => {
            canvas.width = video.videoWidth;
            canvas.height = video.videoHeight;
            canvas.getContext('2d').drawImage(video, 0, 0, canvas.width, canvas.height);
            
            capturedImage = canvas.toDataURL('image/png');
            video.style.display = 'none';
            canvas.style.display = 'block';
            
            captureBtn.style.display = 'none';
            retryBtn.style.display = 'inline-block';
            saveBtn.style.display = 'inline-block';
        });

        // Retry capture
        retryBtn.addEventListener('click', () => {
            video.style.display = 'block';
            canvas.style.display = 'none';
            
            captureBtn.style.display = 'inline-block';
            retryBtn.style.display = 'none';
            saveBtn.style.display = 'none';
            
            resultMessage.style.display = 'none';
        });

        // Save image
        saveBtn.addEventListener('click', async () => {
            try {
                const response = await fetch('/student/capture_face/{{ student.id }}', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify({ image: capturedImage })
                });

                const data = await response.json();
                
                if (data.success) {
                    resultMessage.textContent = data.message;
                    resultMessage.className = "success-message";
                    saveBtn.style.display = 'none';
                } else {
                    resultMessage.textContent = data.message;
                    resultMessage.className = "error-message";
                }
                
                resultMessage.style.display = 'block';
            } catch (err) {
                console.error("Error saving face:", err);
                resultMessage.textContent = "Error saving face data. Please try again.";
                resultMessage.className = "error-message";
                resultMessage.style.display = 'block';
            }
        });

        // Initialize camera when page loads
        window.addEventListener('DOMContentLoaded', startCamera);

        // Stop camera when leaving page
        window.addEventListener('beforeunload', () => {
            if (stream) {
                stream.getTracks().forEach(track => track.stop());
            }
        });
    </script>
</body>
</html>'''

subjects_html = '''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Subjects - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group input, .form-group select {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .btn {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        table, th, td {
            border: 1px solid #ddd;
        }
        th, td {
            padding: 12px;
            text-align: left;
        }
        th {
            background-color: #4CAF50;
            color: white;
        }
        tr:nth-child(even) {
            background-color: #f2f2f2;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
        .action-buttons a {
            color: #4CAF50;
            margin-right: 10px;
            text-decoration: none;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Add New Subject</div>
            
            {% with messages = get_flashed_messages(with_categories=true) %}
                {% if messages %}
                    <ul class="flash-messages">
                        {% for category, message in messages %}
                            <li class="flash-message {{ category }}">{{ message }}</li>
                        {% endfor %}
                    </ul>
                {% endif %}
            {% endwith %}
            
            <form method="POST" action="{{ url_for('manage_subjects') }}">
                <div class="form-group">
                    <label for="code">Subject Code</label>
                    <input type="text" id="code" name="code" required>
                </div>
                <div class="form-group">
                    <label for="name">Subject Name</label>
                    <input type="text" id="name" name="name" required>
                </div>
                <div class="form-group">
                    <label for="semester">Semester</label>
                    <select id="semester" name="semester" required>
                        {% for i in range(1, 9) %}
                        <option value="{{ i }}">Semester {{ i }}</option>
                        {% endfor %}
                    </select>
                </div>
                <div class="form-group">
                    <label for="branch_id">Branch</label>
                    <select id="branch_id" name="branch_id" required>
                        {% for branch in branches %}
                        <option value="{{ branch.id }}">{{ branch.name }}</option>
                        {% endfor %}
                    </select>
                </div>
                <button type="submit" class="btn">Add Subject</button>
            </form>
        </div>

        <div class="card">
            <div class="card-header">Subject List</div>
            <table>
                <thead>
                    <tr>
                        <th>Code</th>
                        <th>Name</th>
                        <th>Semester</th>
                        <th>Branch</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    {% for subject in subjects %}
                    <tr>
                        <td>{{ subject.code }}</td>
                        <td>{{ subject.name }}</td>
                        <td>Semester {{ subject.semester }}</td>
                        <td>{{ subject.branch_name }}</td>
                        <td class="action-buttons">
                            <a href="#">Edit</a>
                            <a href="#">Delete</a>
                        </td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
    </div>
</body>
</html>'''

timetable_html = '''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Timetable - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group input, .form-group select {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .btn {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        table, th, td {
            border: 1px solid #ddd;
        }
        th, td {
            padding: 12px;
            text-align: left;
        }
        th {
            background-color: #4CAF50;
            color: white;
        }
        tr:nth-child(even) {
            background-color: #f2f2f2;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
        .action-buttons a {
            color: #4CAF50;
            margin-right: 10px;
            text-decoration: none;
        }
        .timetable-view {
            margin-top: 30px;
            overflow-x: auto;
        }
        .timetable {
            width: 100%;
            border-collapse: collapse;
        }
        .timetable th, .timetable td {
            border: 1px solid #ddd;
            padding: 8px;
            text-align: center;
        }
        .timetable th {
            background-color: #4CAF50;
            color: white;
        }
        .timetable td {
            height: 60px;
            vertical-align: top;
        }
        .timetable-class {
            background-color: #e8f5e9;
            border-radius: 3px;
            padding: 5px;
            margin: 2px 0;
            font-size: 12px;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Add Timetable Entry</div>
            
            {% with messages = get_flashed_messages(with_categories=true) %}
                {% if messages %}
                    <ul class="flash-messages">
                        {% for category, message in messages %}
                            <li class="flash-message {{ category }}">{{ message }}</li>
                        {% endfor %}
                    </ul>
                {% endif %}
            {% endwith %}
            
            <form method="POST" action="{{ url_for('manage_timetable') }}">
                <div class="form-group">
                    <label for="day_of_week">Day of Week</label>
                    <select id="day_of_week" name="day_of_week" required>
                        <option value="Monday">Monday</option>
                        <option value="Tuesday">Tuesday</option>
                        <option value="Wednesday">Wednesday</option>
                        <option value="Thursday">Thursday</option>
                        <option value="Friday">Friday</option>
                        <option value="Saturday">Saturday</option>
                    </select>
                </div>
                <div class="form-group">
                    <label for="start_time">Start Time</label>
                    <input type="time" id="start_time" name="start_time" required>
                </div>
                <div class="form-group">
                    <label for="end_time">End Time</label>
                    <input type="time" id="end_time" name="end_time" required>
                </div>
                <div class="form-group">
                    <label for="subject_id">Subject</label>
                    <select id="subject_id" name="subject_id" required>
                        {% for subject in subjects %}
                        <option value="{{ subject.id }}">{{ subject.name }} ({{ subject.code }})</option>
                        {% endfor %}
                    </select>
                </div>
                <div class="form-group">
                    <label for="branch_id">Branch</label>
                    <select id="branch_id" name="branch_id" required>
                        {% for branch in branches %}
                        <option value="{{ branch.id }}">{{ branch.name }}</option>
                        {% endfor %}
                    </select>
                </div>
                <div class="form-group">
                    <label for="semester">Semester</label>
                    <select id="semester" name="semester" required>
                        {% for i in range(1, 9) %}
                        <option value="{{ i }}">Semester {{ i }}</option>
                        {% endfor %}
                    </select>
                </div>
                <button type="submit" class="btn">Add Entry</button>
            </form>
        </div>

        <div class="card">
            <div class="card-header">Timetable Entries</div>
            <table>
                <thead>
                    <tr>
                        <th>Day</th>
                        <th>Time</th>
                        <th>Subject</th>
                        <th>Branch</th>
                        <th>Semester</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    {% for entry in timetable %}
                    <tr>
                        <td>{{ entry.day_of_week }}</td>
                        <td>{{ entry.start_time.strftime('%H:%M') }} - {{ entry.end_time.strftime('%H:%M') }}</td>
                        <td>{{ entry.subject_name }}</td>
                        <td>{{ entry.branch_name }}</td>
                        <td>Semester {{ entry.semester }}</td>
                        <td class="action-buttons">
                            <a href="#">Edit</a>
                            <a href="#">Delete</a>
                        </td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>

        <div class="card">
            <div class="card-header">Weekly Timetable View</div>
            <div class="timetable-view">
                <table class="timetable">
                    <thead>
                        <tr>
                            <th>Time</th>
                            <th>Monday</th>
                            <th>Tuesday</th>
                            <th>Wednesday</th>
                            <th>Thursday</th>
                            <th>Friday</th>
                            <th>Saturday</th>
                        </tr>
                    </thead>
                    <tbody>
                        {% for hour in range(8, 18) %}
                        <tr>
                            <td>{{ hour }}:00 - {{ hour+1 }}:00</td>
                            {% for day in ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'] %}
                            <td>
                                {% for entry in timetable %}
                                    {% if entry.day_of_week == day and hour >= entry.start_time.hour and hour < entry.end_time.hour %}
                                    <div class="timetable-class">
                                        {{ entry.subject_name }}<br>
                                        {{ entry.branch_name }} - Sem {{ entry.semester }}
                                    </div>
                                    {% endif %}
                                {% endfor %}
                            </td>
                            {% endfor %}
                        </tr>
                        {% endfor %}
                    </tbody>
                </table>
            </div>
        </div>
    </div>
</body>
</html>'''


attendance_report_html = '''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Attendance Report - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group input, .form-group select {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .btn {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        table, th, td {
            border: 1px solid #ddd;
        }
        th, td {
            padding: 12px;
            text-align: left;
        }
        th {
            background-color: #4CAF50;
            color: white;
        }
        tr:nth-child(even) {
            background-color: #f2f2f2;
        }
        .low-attendance {
            background-color: #ffebee;
        }
        .filter-form {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
            gap: 15px;
            margin-bottom: 20px;
        }
        .filter-actions {
            display: flex;
            justify-content: flex-end;
            gap: 10px;
        }
        .percentage-cell {
            font-weight: bold;
        }
        .percentage-high {
            color: #2e7d32;
        }
        .percentage-medium {
            color: #ff8f00;
        }
        .percentage-low {
            color: #c62828;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <!-- attendance_report.html continued -->
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Attendance Report</div>
            
            <form method="GET" action="{{ url_for('attendance_report') }}">
                <div class="filter-form">
                    <div class="form-group">
                        <label for="branch_id">Branch</label>
                        <select id="branch_id" name="branch_id">
                            <option value="">All Branches</option>
                            {% for branch in branches %}
                            <option value="{{ branch.id }}" {% if request.args.get('branch_id') == branch.id|string %}selected{% endif %}>{{ branch.name }}</option>
                            {% endfor %}
                        </select>
                    </div>
                    <div class="form-group">
                        <label for="semester">Semester</label>
                        <select id="semester" name="semester">
                            <option value="">All Semesters</option>
                            {% for i in range(1, 9) %}
                            <option value="{{ i }}" {% if request.args.get('semester') == i|string %}selected{% endif %}>Semester {{ i }}</option>
                            {% endfor %}
                        </select>
                    </div>
                    <div class="form-group">
                        <label for="subject_id">Subject</label>
                        <select id="subject_id" name="subject_id">
                            <option value="">All Subjects</option>
                            {% for subject in subjects %}
                            <option value="{{ subject.id }}" {% if request.args.get('subject_id') == subject.id|string %}selected{% endif %}>{{ subject.name }}</option>
                            {% endfor %}
                        </select>
                    </div>
                    <div class="form-group">
                        <label for="start_date">Start Date</label>
                        <input type="date" id="start_date" name="start_date" value="{{ request.args.get('start_date', '') }}">
                    </div>
                    <div class="form-group">
                        <label for="end_date">End Date</label>
                        <input type="date" id="end_date" name="end_date" value="{{ request.args.get('end_date', '') }}">
                    </div>
                </div>
                <div class="filter-actions">
                    <button type="submit" class="btn">Apply Filters</button>
                    <a href="{{ url_for('attendance_report') }}" class="btn" style="background-color: #6c757d;">Reset</a>
                </div>
            </form>

            <table>
                <thead>
                    <tr>
                        <th>Roll No.</th>
                        <th>Student Name</th>
                        <th>Classes Attended</th>
                        <th>Total Classes</th>
                        <th>Attendance %</th>
                    </tr>
                </thead>
                <tbody>
                    {% for student in attendance_data %}
                    <tr {% if (student.attended_classes / student.total_classes * 100) < 30 %}class="low-attendance"{% endif %}>
                        <td>{{ student.roll_number }}</td>
                        <td>{{ student.name }}</td>
                        <td>{{ student.attended_classes }}</td>
                        <td>{{ student.total_classes }}</td>
                        <td class="percentage-cell 
                            {% if (student.attended_classes / student.total_classes * 100) >= 75 %}percentage-high
                            {% elif (student.attended_classes / student.total_classes * 100) >= 30 %}percentage-medium
                            {% else %}percentage-low{% endif %}">
                            {{ "%.1f"|format((student.attended_classes / student.total_classes * 100) }}%
                        </td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
    </div>
</body>
</html>'''


notifications_html = '''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Notifications - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .notification {
            padding: 15px;
            border-left: 4px solid #4CAF50;
            background-color: #f9f9f9;
            margin-bottom: 15px;
            border-radius: 4px;
        }
        .notification.unread {
            border-left-color: #f44336;
            background-color: #ffebee;
        }
        .notification-date {
            font-size: 12px;
            color: #666;
            margin-top: 5px;
        }
        .no-notifications {
            text-align: center;
            padding: 20px;
            color: #666;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">Notifications</div>
            
            {% if notifications %}
                {% for notification in notifications %}
                <div class="notification {% if not notification.is_read %}unread{% endif %}">
                    <div>{{ notification.message }}</div>
                    <div class="notification-date">{{ notification.date.strftime('%Y-%m-%d %H:%M') }}</div>
                </div>
                {% endfor %}
            {% else %}
                <div class="no-notifications">No notifications to display</div>
            {% endif %}
        </div>
    </div>
</body>
</html>'''


settings_html = '''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Settings - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f5f5f5;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .sidebar {
            width: 250px;
            background-color: #333;
            color: white;
            position: fixed;
            height: 100%;
            padding-top: 20px;
        }
        .sidebar a {
            display: block;
            color: white;
            padding: 15px;
            text-decoration: none;
        }
        .sidebar a:hover {
            background-color: #4CAF50;
        }
        .main-content {
            margin-left: 250px;
            padding: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .card-header {
            border-bottom: 1px solid #eee;
            padding-bottom: 10px;
            margin-bottom: 15px;
            font-size: 18px;
            font-weight: bold;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        .form-group input {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        .btn {
            padding: 10px 15px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        .flash-messages {
            list-style-type: none;
            padding: 0;
        }
        .flash-message {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 4px;
            color: white;
        }
        .flash-message.error {
            background-color: #f44336;
        }
        .flash-message.success {
            background-color: #4CAF50;
        }
        .section-title {
            font-size: 16px;
            font-weight: bold;
            margin: 20px 0 10px 0;
            padding-bottom: 5px;
            border-bottom: 1px solid #eee;
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <a href="{{ url_for('dashboard') }}">Dashboard</a>
        <a href="{{ url_for('setup_college') }}">College Setup</a>
        <a href="{{ url_for('setup_department') }}">Departments</a>
        <a href="{{ url_for('setup_branch') }}">Branches</a>
        <a href="{{ url_for('manage_students') }}">Students</a>
        <a href="{{ url_for('manage_subjects') }}">Subjects</a>
        <a href="{{ url_for('manage_timetable') }}">Timetable</a>
        <a href="{{ url_for('attendance_report') }}">Attendance Reports</a>
        <a href="{{ url_for('notifications') }}">Notifications</a>
        <a href="{{ url_for('user_settings') }}">Settings</a>
        <a href="{{ url_for('logout') }}">Logout</a>
    </div>

    <div class="main-content">
        <div class="header">
            <h2>College Attendance System</h2>
            <div>Welcome, {{ session.username }} ({{ session.user_role }})</div>
        </div>

        <div class="card">
            <div class="card-header">User Settings</div>
            
            {% with messages = get_flashed_messages(with_categories=true) %}
                {% if messages %}
                    <ul class="flash-messages">
                        {% for category, message in messages %}
                            <li class="flash-message {{ category }}">{{ message }}</li>
                        {% endfor %}
                    </ul>
                {% endif %}
            {% endwith %}
            
            <form method="POST" action="{{ url_for('user_settings') }}">
                <div class="section-title">Account Information</div>
                <div class="form-group">
                    <label for="email">Email</label>
                    <input type="email" id="email" name="email" value="{{ user.email }}" required>
                </div>

                <div class="section-title">Change Password</div>
                <div class="form-group">
                    <label for="current_password">Current Password</label>
                    <input type="password" id="current_password" name="current_password">
                </div>
                <div class="form-group">
                    <label for="new_password">New Password</label>
                    <input type="password" id="new_password" name="new_password">
                </div>
                <div class="form-group">
                    <label for="confirm_password">Confirm New Password</label>
                    <input type="password" id="confirm_password" name="confirm_password">
                </div>

                <button type="submit" class="btn">Save Changes</button>
            </form>
        </div>
    </div>
</body>
</html>'''


unauthorized_html = '''<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Unauthorized - College Attendance</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f5f5f5;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
            text-align: center;
        }
        .error-container {
            background-color: white;
            padding: 40px;
            border-radius: 8px;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
            max-width: 500px;
        }
        h1 {
            font-size: 72px;
            margin: 0;
            color: #ff9800;
        }
        h2 {
            margin: 10px 0 20px 0;
        }
        p {
            margin-bottom: 30px;
            color: #666;
        }
        a {
            display: inline-block;
            padding: 10px 20px;
            background-color: #4CAF50;
            color: white;
            text-decoration: none;
            border-radius: 4px;
        }
        a:hover {
            background-color: #45a049;
        }
    </style>
</head>
<body>
    <div class="error-container">
        <h1>403</h1>
        <h2>Unauthorized Access</h2>
        <p>You don't have permission to access this page. Please contact your administrator if you believe this is an error.</p>
        <a href="{{ url_for('dashboard') }}">Go to Dashboard</a>
    </div>
</body>
</html>'''


# Main entry point
if __name__ == '__main__':
    # Initialize database
    initialize_database()
    
    # Start Flask application
    app.run(host='0.0.0.0', port=5000, debug=True)

  


