###########Placement Eligibility Streamlit Application################

#Importing Necessary Libraries

import sqlite3 
import random
from faker import Faker
import streamlit as st
import pandas as pd
import plotly.express as px
import os
from dotenv import load_dotenv

# Database connection 
DB_NAME="placement.db"

load_dotenv()

DB_NAME = os.getenv("DB_NAME")


def create_connection():
    return sqlite3.connect("DB_NAME")

def create_tables():
    conn = create_connection()
    cursor = conn.cursor()

    # Students Table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS Students (
            student_id INTEGER PRIMARY KEY,
            name TEXT,
            age INTEGER,
            gender TEXT,
            email TEXT,
            phone TEXT,
            enrollment_year INTEGER,
            course_batch TEXT,
            city TEXT,
            graduation_year INTEGER
        )
    ''')

    # Programming Table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS Programming (
            programming_id INTEGER PRIMARY KEY,
            student_id INTEGER,
            language TEXT,
            problems_solved INTEGER,
            assessments_completed INTEGER,
            mini_projects INTEGER,
            certifications_earned INTEGER,
            latest_project_score INTEGER,
            FOREIGN KEY(student_id) REFERENCES Students(student_id)
        )
    ''')

    # Soft Skills Table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS SoftSkills (
            soft_skill_id INTEGER PRIMARY KEY,
            student_id INTEGER,
            communication INTEGER,
            teamwork INTEGER,
            presentation INTEGER,
            leadership INTEGER,
            critical_thinking INTEGER,
            interpersonal_skills INTEGER,
            FOREIGN KEY(student_id) REFERENCES Students(student_id)
        )
    ''')

    # Placements Table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS Placements (
            placement_id INTEGER PRIMARY KEY,
            student_id INTEGER,
            mock_interview_score INTEGER,
            internships_completed INTEGER,
            placement_status TEXT,
            company_name TEXT,
            placement_package REAL,
            interview_rounds_cleared INTEGER,
            placement_date TEXT,
            FOREIGN KEY(student_id) REFERENCES Students(student_id)
        )
    ''')

    conn.commit()
    conn.close()

# Data Generation with Faker Library
students_data = Faker()

def generate_data(num_students=100):
    conn = create_connection()
    cursor = conn.cursor()

    for _ in range(num_students):
        name = students_data.name()
        age = random.randint(18, 25)
        gender = random.choice(["Male", "Female", "Other"])
        email = students_data.email()
        phone = students_data.phone_number()
        enrollment_year = random.randint(2019, 2024)
        course_batch = f"AIML_Batch_{random.randint(1,13)}"
        city = students_data.city()
        graduation_year = enrollment_year + 1

# Insert data into Students table

        cursor.execute('''
            INSERT INTO Students (name, age, gender, email, phone, enrollment_year, course_batch, city, graduation_year)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
        ''', (name, age, gender, email, phone, enrollment_year, course_batch, city, graduation_year))

        student_id = cursor.lastrowid

# Insert data into Programming table

        cursor.execute('''
            INSERT INTO Programming (student_id, language, problems_solved, assessments_completed, mini_projects, certifications_earned, latest_project_score)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        ''', (student_id, random.choice(["Python", "R", "SQL"]), random.randint(20, 100), random.randint(5, 20), random.randint(1, 5), random.randint(0, 3), random.randint(50, 100)))

# Insert data into SoftSkills table

        cursor.execute('''
            INSERT INTO SoftSkills (student_id, communication, teamwork, presentation, leadership, critical_thinking, interpersonal_skills)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        ''', (student_id, random.randint(50, 100), random.randint(50, 100), random.randint(50, 100), random.randint(50, 100), random.randint(50, 100), random.randint(50, 100)))

# Insert data into Placements table

        placement_status = random.choice(["Ready", "Not Ready", "Placed"])
        company_name = students_data.company() if placement_status == "Placed" else None
        placement_package = round(random.uniform(4, 20), 2) if placement_status == "Placed" else None
        interview_rounds = random.randint(1, 5)
        placement_date = students_data.date_this_decade() if placement_status == "Placed" else None

        cursor.execute('''
            INSERT INTO Placements (student_id, mock_interview_score, internships_completed, placement_status, company_name, placement_package, interview_rounds_cleared, placement_date)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?)
        ''', (student_id, random.randint(50, 100), random.randint(0, 3), placement_status, company_name, placement_package, interview_rounds, placement_date))

    conn.commit()
    conn.close()

# Fetch Query - Students Placement Eligibility

def fetch_data(query):
    conn = create_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Streamlit App
st.title("Placement Eligibility App")

# Sidebar Filters
st.sidebar.header("Placement Eligibility Selection Criteria")
min_problems_solved = st.sidebar.slider("Minimum Problems Solved", 0, 100, 50)
min_communication = st.sidebar.slider("Minimum Communication Score", 0, 100, 75)
placement_status_filter = st.sidebar.selectbox("Placement Status", ["All", "Ready", "Not Ready", "Placed"])

def display_student_data():
    query = f'''
    SELECT S.student_id, S.name, P.language, P.problems_solved, SS.communication, PL.placement_status
    FROM Students S
    JOIN Programming P ON S.student_id = P.student_id
    JOIN SoftSkills SS ON S.student_id = SS.student_id
    JOIN Placements PL ON S.student_id = PL.student_id
    WHERE P.problems_solved >= {min_problems_solved}
    AND SS.communication >= {min_communication}
    '''

    if placement_status_filter != "All":
        query += f" AND PL.placement_status = '{placement_status_filter}'"

    df = fetch_data(query)
    st.dataframe(df)

    # Insights
    #1. Percentage of Students Ready for Placement
    placement_ready_pct_query = """
    SELECT ROUND(
        (COUNT(CASE WHEN placement_status = 'Ready' THEN 1 ELSE NULL END) * 100.0) / COUNT(*), 2
    ) AS percentage_ready
    FROM Placements;
    """
    placement_ready_pct_df = fetch_data(placement_ready_pct_query)
    st.subheader("Percentage of Students Ready for Placement")
    st.metric(label="% of Students Ready for Placement", value=f"{placement_ready_pct_df['percentage_ready'][0]}%")

    #2. Average Programming Performance Per Batch
    avg_perf_query = "SELECT course_batch, AVG(problems_solved) AS avg_problems FROM Programming JOIN Students ON Students.student_id = Programming.student_id GROUP BY course_batch"
    avg_perf_df = fetch_data(avg_perf_query)
    st.subheader("Average Programming Performance Per Batch")
    st.bar_chart(avg_perf_df.set_index('course_batch'))

    #3. Top 5 Students Ready for Placement
    top_students_query = "SELECT name, problems_solved FROM Students JOIN Programming ON Students.student_id = Programming.student_id ORDER BY problems_solved DESC LIMIT 5"
    top_students_df = fetch_data(top_students_query)
    st.subheader("Top 5 Students Ready for Placement")
    st.bar_chart(top_students_df.set_index('name'))
    
    #4. Distribution of Soft Skills Scores
    soft_skills_disb_query = """
    SELECT 'Communication' AS Skill, AVG(communication) AS avg_score FROM SoftSkills
    UNION ALL
    SELECT 'Teamwork', AVG(teamwork) FROM SoftSkills
    UNION ALL
    SELECT 'Presentation', AVG(presentation) FROM SoftSkills    
    UNION ALL
    SELECT 'Leadership', AVG(leadership) FROM SoftSkills
    UNION ALL
    SELECT 'Critical Thinking', AVG(critical_thinking) FROM SoftSkills
    UNION ALL
    SELECT 'Interpersonal Skills', AVG(interpersonal_skills) FROM SoftSkills;
    """
    soft_skills_disb_df = fetch_data(soft_skills_disb_query)
    st.subheader("Distribution of Soft Skills Scores")
    st.bar_chart(soft_skills_disb_df.set_index('Skill'))

#5. Top 5 Students by Placement Package
    top_package_query = """
    SELECT S.name, PL.placement_package
    FROM Students S
    JOIN Placements PL ON S.student_id = PL.student_id
    WHERE PL.placement_status = 'Placed'
    ORDER BY PL.placement_package DESC
    LIMIT 5;
    """
    top_package_df = fetch_data(top_package_query)
    st.subheader("Top 5 Students with Highest Placement Packages")
    st.bar_chart(top_package_df.set_index('name'))

# 6. Students with Highest Certifications
    certifications_query = """
    SELECT S.name, P.certifications_earned
    FROM Students S
    JOIN Programming P ON S.student_id = P.student_id
    ORDER BY P.certifications_earned DESC LIMIT 5;
    """
    certifications_df = fetch_data(certifications_query)
    st.subheader("Top 5 Students with Most Certifications")
    st.bar_chart(certifications_df.set_index('name'))

# 7. Top 3 Cities with Most Placed Students
    cities_most_placed_query = """
    SELECT S.city, COUNT(*) AS placed_students
    FROM Students S
    JOIN Placements PL ON S.student_id = PL.student_id
    WHERE PL.placement_status = 'Placed'
    GROUP BY S.city
    ORDER BY placed_students DESC LIMIT 3;
    """
    cities_most_placed_df = fetch_data(cities_most_placed_query)
    st.subheader("Top 3 Cities with Most Placed Students")
    st.bar_chart(cities_most_placed_df.set_index('city'))

# 8. Average Placement Package per Course Batch
    avg_pkg_query = """
    SELECT S.course_batch, AVG(PL.placement_package) AS avg_package
    FROM Students S
    JOIN Placements PL ON S.student_id = PL.student_id
    GROUP BY S.course_batch
    ORDER BY avg_package DESC;
    """
    avg_pkg_df = fetch_data(avg_pkg_query)
    st.subheader("Average Placement Package per Course Batch")
    st.bar_chart(avg_pkg_df.set_index('course_batch'))

# 9. Number of Students by Programming Language
    students_prog_query = """
    SELECT P.language, COUNT(*) AS total_students
    FROM Programming P
    GROUP BY P.language
    ORDER BY total_students DESC;
    """
    students_prog_df = fetch_data(students_prog_query)
    st.subheader("Number of Students by Programming Language")
    st.bar_chart(students_prog_df.set_index('language'))

#10. Top 5 Students with Most Internship Experience
    internship_exp_query = """
    SELECT S.name, PL.internships_completed
    FROM Students S
    JOIN Placements PL ON S.student_id = PL.student_id
    ORDER BY PL.internships_completed DESC LIMIT 5;
    """
    internship_exp_df = fetch_data(internship_exp_query)
    st.subheader("Top 5 Students with Most Internship Experience")
    st.bar_chart(internship_exp_df.set_index('name'))


if __name__ == "__main__":
    create_tables()
    generate_data(100)
    display_student_data()
