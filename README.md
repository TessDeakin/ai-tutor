# ai-tutor
import streamlit as st
import pandas as pd
import plotly.express as px
from datetime import datetime, timedelta
import random

# Initialize session state variables if they don't exist
if 'authenticated' not in st.session_state:
    st.session_state.authenticated = False
if 'current_user' not in st.session_state:
    st.session_state.current_user = None
if 'user_role' not in st.session_state:
    st.session_state.user_role = None

# Sample data structures (in a real app, this would be a database)
class DataStore:
    def __init__(self):
        self.users = {
            'teacher1': {'password': 'pass123', 'role': 'teacher'},
            'student1': {'password': 'pass123', 'role': 'student', 'grades': []},
            'student2': {'password': 'pass123', 'role': 'student', 'grades': []}
        }
        
        self.skills = [
            'Creative Writing',
            'Research Skills',
            'Critical Thinking',
            'Analysis',
            'Communication',
            'Problem Solving'
        ]
        
        self.assignments = []
        self.student_projects = {}
        
        # Initialize some sample grades
        for student in ['student1', 'student2']:
            dates = pd.date_range(start='2024-01-01', end='2024-11-09', periods=5)
            self.users[student]['grades'] = [
                {
                    'date': date.strftime('%Y-%m-%d'),
                    'grade': random.randint(70, 95),
                    'skills': random.sample(self.skills, 2),
                    'topic': f'Sample Project {i+1}'
                }
                for i, date in enumerate(dates)
            ]

db = DataStore()

def generate_project_prompts(topic, skills):
    """Generate three different project prompts based on topic and skills."""
    prompts = [
        f"Create an engaging {random.choice(skills)} project about {topic} that focuses on {random.choice(['past', 'present', 'future'])} developments.",
        f"Develop a comparative analysis of different aspects of {topic} using your {random.choice(skills)} skills.",
        f"Design an innovative exploration of {topic} that demonstrates your mastery of {random.choice(skills)}."
    ]
    return prompts

def login_page():
    st.title("AI Learning Platform")
    
    username = st.text_input("Username")
    password = st.text_input("Password", type="password")
    
    if st.button("Login"):
        if username in db.users and db.users[username]['password'] == password:
            st.session_state.authenticated = True
            st.session_state.current_user = username
            st.session_state.user_role = db.users[username]['role']
            st.rerun()
        else:
            st.error("Invalid credentials")

def logout():
    if st.sidebar.button("Logout"):
        st.session_state.authenticated = False
        st.session_state.current_user = None
        st.session_state.user_role = None
        st.rerun()

def teacher_dashboard():
    st.title("Teacher Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.radio("Navigation", ["Create Assignment", "View Student Progress"])
    logout()
    
    if page == "Create Assignment":
        st.header("Create New Assignment")
        
        # Multi-select for skills
        selected_skills = st.multiselect("Select Skills", db.skills)
        
        # Due date selection
        due_date = st.date_input("Due Date")
        
        if st.button("Create Assignment"):
            if selected_skills and due_date:
                new_assignment = {
                    'skills': selected_skills,
                    'due_date': due_date.strftime('%Y-%m-%d')
                }
                db.assignments.append(new_assignment)
                st.success("Assignment created successfully!")
            else:
                st.error("Please select skills and due date")
    
    else:  # View Student Progress
        st.header("Student Progress")
        
        # Student selection
        student = st.selectbox("Select Student", 
                             [user for user, data in db.users.items() if data['role'] == 'student'])
        
        if student:
            # Display student's grades
            grades_data = db.users[student]['grades']
            if grades_data:
                df = pd.DataFrame(grades_data)
                
                # Create scatter plot
                fig = px.scatter(df, x='date', y='grade',
                               title=f'{student} Progress Over Time',
                               labels={'date': 'Date', 'grade': 'Grade'},
                               template='simple_white')
                st.plotly_chart(fig)
                
                # Display detailed project information
                st.subheader("Project Details")
                for project in grades_data:
                    with st.expander(f"Project: {project['topic']} ({project['date']})"):
                        st.write(f"Grade: {project['grade']}")
                        st.write(f"Skills: {', '.join(project['skills'])}")

def student_dashboard():
    st.title("Student Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.radio("Navigation", ["Start New Project", "View Progress"])
    logout()
    
    if page == "Start New Project":
        st.header("Start New Project")
        
        # Get available assignments
        available_assignments = [assignment for assignment in db.assignments 
                               if datetime.strptime(assignment['due_date'], '%Y-%m-%d').date() >= datetime.now().date()]
        
        if available_assignments:
            selected_assignment = st.selectbox(
                "Select Assignment",
                range(len(available_assignments)),
                format_func=lambda i: f"Assignment {i+1} (Due: {available_assignments[i]['due_date']})"
            )
            
            # Topic input
            topic = st.text_input("Enter your topic of interest")
            
            if topic:
                st.subheader("Choose your project prompt:")
                prompts = generate_project_prompts(topic, available_assignments[selected_assignment]['skills'])
                selected_prompt = st.radio("Available Projects:", prompts)
                
                if st.button("Start Project"):
                    st.success("Project started! You can now begin working on your submission.")
                    # In a real application, this would create a new project in the database
        else:
            st.info("No assignments currently available")
    
    else:  # View Progress
        st.header("Your Progress")
        
        # Get student's grades
        grades_data = db.users[st.session_state.current_user]['grades']
        if grades_data:
            df = pd.DataFrame(grades_data)
            
            # Create scatter plot
            fig = px.scatter(df, x='date', y='grade',
                           title='Your Progress Over Time',
                           labels={'date': 'Date', 'grade': 'Grade'},
                           template='simple_white')
            st.plotly_chart(fig)
            
            # Display skill breakdown
            st.subheader("Skill Progress")
            for skill in db.skills:
                skill_grades = [g['grade'] for g in grades_data if skill in g['skills']]
                if skill_grades:
                    avg_grade = sum(skill_grades) / len(skill_grades)
                    st.write(f"{skill}: {avg_grade:.1f}%")

def main():
    if not st.session_state.authenticated:
        login_page()
    else:
        if st.session_state.user_role == 'teacher':
            teacher_dashboard()
        else:
            student_dashboard()

if __name__ == "__main__":
    main()
