import cv2
import face_recognition
import os
import pandas as pd
from datetime import datetime
import time
import pyttsx3  # Import the pyttsx3 library for TTS

    # Initialize TTS engine
engine = pyttsx3.init()
engine.setProperty('rate', 150)  # Adjust speech rate (default is 200)
voices = engine.getProperty('voices')
engine.setProperty('voice', voices[0].id)  # Set to male voice (typically index 0)


    # Function to speak a message
def speak(message):
    engine.setProperty('volume', 1.0)
    engine.say(message)
    engine.runAndWait()


        # Define paths
students_folder = r"C:\Users\HP\Desktop\students"
attendance_folder = r"C:\Users\HP\Desktop\attendence"  # Folder for attendance file
attendance_file = os.path.join(attendance_folder, "attendance.csv")  # Full file path

    # Ensure attendance directory exists
if not os.path.exists(attendance_folder):
    os.makedirs(attendance_folder)
    print(f"Directory '{attendance_folder}' created.")
else:
    print(f"Directory '{attendance_folder}' already exists.")


    # Load student images and encode them (cache face encodings)
def load_student_images(folder_path):
    student_images = []
    student_names = []
    for filename in os.listdir(folder_path):
        if filename.endswith((".jpg", ".jpeg", ".png")):
            img = cv2.imread(os.path.join(folder_path, filename))
            rgb_img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
            face_encodings = face_recognition.face_encodings(rgb_img)

            if face_encodings:
                student_images.append(face_encodings[0])
                student_names.append(filename.split('.')[0])
            else:
                print(f"⚠️ Warning: No face found in image {filename}. Skipping.")
                speak(f"No face found in image {filename} file. Skipping.")
    return student_images, student_names


    # Mark attendance in the CSV file with morning/evening distinction
def mark_attendance(name, df):
    columns = ["Name", "Date", "Morning Time", "Evening Time"]
    now = datetime.now()
    attendance_time = now.strftime("%H:%M:%S")
    attendance_date = now.strftime("%Y-%m-%d")
    is_morning = now.hour < 12  # True for morning, False for evening

    # Check if student already has a record for the day
    if name in df["Name"].values and attendance_date in df["Date"].values:
        student_index = df[(df["Name"] == name) & (df["Date"] == attendance_date)].index[0]

        if is_morning and pd.isna(df.at[student_index, "Morning Time"]):
            df.at[student_index, "Morning Time"] = attendance_time
            print(f"Morning attendance marked for {name} at {attendance_time}.")
            speak(f"Morning attendance marked for {name} morning session.")
        elif not is_morning and pd.isna(df.at[student_index, "Evening Time"]):
            df.at[student_index, "Evening Time"] = attendance_time
            print(f"Evening attendance marked for {name} at {attendance_time}.")
            speak(f"Evening attendance marked for {name} in Evening session.")
        else:
            print(f"Attendance for {name} is already recorded in this session.")
            speak(f"Attendance for {name} is already recorded in this session.")
    else:
        # Create a new entry
        new_entry = {"Name": name, "Date": attendance_date, "Morning Time": None, "Evening Time": None}
        if is_morning:
            new_entry["Morning Time"] = attendance_time
        else:
            new_entry["Evening Time"] = attendance_time

        df = pd.concat([df, pd.DataFrame([new_entry])], ignore_index=True)
        print(f"New entry created. Attendance marked for {name} at {attendance_time}.")
        speak(f"New entry created. Attendance marked for {name} successfully.")

    # Save the DataFrame to CSV immediately after modification
    df.to_csv(attendance_file, index=False, encoding="utf-8")
    return df


    # View today's attendance
def view_todays_attendance(df):
    now = datetime.now()
    attendance_date = now.strftime("%Y-%m-%d")

    todays_attendance = df[df["Date"] == attendance_date]
    print(f"\nToday's Attendance ({attendance_date}):")
    if not todays_attendance.empty:
        print(todays_attendance[["Name", "Morning Time", "Evening Time"]])
        speak(f"Here is today's attendance for {attendance_date}.")
    else:
        print("No attendance recorded today.")
        speak("No attendance recorded today.")


    # View attendance for a specific date
def view_attendance_for_date(df):
    date_input = input("Enter the date to view attendance (YYYY-MM-DD): ")
    attendance_data = df[df["Date"] == date_input]

    if not attendance_data.empty:
        print(f"Attendance for {date_input}:")
        print(attendance_data[["Name", "Morning Time", "Evening Time"]])
        speak(f"Here is the attendance for {date_input}.")
    else:
        print(f"No attendance found for {date_input}.")
        speak(f"No attendance found for {date_input}.")


    # Count the number of classes attended by each student
def count_classes_attended(df):
    student_attendance = df.groupby("Name").size()

    if student_attendance.empty:
        print("\nNo attendance data available yet.")
        speak("No attendance data available yet.")
    else:
        print("\nThe number of classes attended by each student:")
        speak(f"Here are the number of classes attended by each student.")
        for student, classes_attended in student_attendance.items():
            print(f"{student}: {classes_attended} classes attended")


    # Enroll a new student
def enroll_new_student(student_images, student_names):
    print("📷 Please position the student in front of the camera.")
    # Replace with the URL of your mobile camera stream (IP Webcam)
    mobile_camera_url = "http://192.168.71.164:8080/video"  # Example: http://192.168.1.5:8080/video

    # Attempt to connect to the IP Webcam with a short timeout
    cap = None
    start_time = time.time()
    timeout = 2  # Reduced timeout to 2 second for quicker fallback to system camera

    # Try IP Webcam first, with a shorter timeout for connection check
    while cap is None and time.time() - start_time < timeout:
        cap = cv2.VideoCapture(mobile_camera_url)  # Use IP Webcam stream
        if cap.isOpened():
            print("Connected to IP Webcam.")
            break
        time.sleep(0.1)  # Reduced wait time to 100ms between retries
        print("⏳ Trying to connect to IP Webcam...")

    if cap is None or not cap.isOpened():
        print("⚠️ IP Webcam not found or connection failed, falling back to system camera.")
        cap = cv2.VideoCapture(0)  # Fallback to system camera if IP Webcam is not available

    try:
        while True:
            ret, frame = cap.read()
            if not ret:
                print("⚠️ Failed to capture image.")
                break

            # Resize the frame to half the original width (adjust the height proportionally)
            height, width = frame.shape[:2]
            resized_frame = cv2.resize(frame, (width // 2, height // 2))  # Half the width and height

            # Display the resized frame
            cv2.imshow("Press 's' to capture the student's image", resized_frame)

            key = cv2.waitKey(1) & 0xFF

            if key == ord('s'):  # If the 's' key is pressed, capture the image
                # Resize image to optimize face recognition speed
                small_frame = cv2.resize(resized_frame, (640, 480))
                rgb_frame = cv2.cvtColor(small_frame, cv2.COLOR_BGR2RGB)
                face_locations = face_recognition.face_locations(rgb_frame)
                face_encodings = face_recognition.face_encodings(rgb_frame, face_locations)

                if face_encodings:
                    new_face_encoding = face_encodings[0]

                    # Compare the captured face encoding with already enrolled students
                    matches = face_recognition.compare_faces(student_images, new_face_encoding)

                    if True in matches:
                        # Find the index of the matched student
                        first_match_index = matches.index(True)
                        existing_student_name = student_names[first_match_index]
                        print(f"⚠️ Student {existing_student_name} is already enrolled.")
                        speak(f"Student {existing_student_name} is already enrolled.")
                        break  # Break out of the loop after showing the message
                    else:
                        # If the student is not already enrolled, proceed with enrolling them
                        # Close the camera window after capture
                        cv2.destroyAllWindows()  # Close the camera window

                        # Ask for student name after closing the camera window
                        student_name = input("Enter student's name: ")

                        # Save the image in the students folder
                        img_filename = f"{student_name}.jpg"
                        cv2.imwrite(os.path.join(students_folder, img_filename), frame)

                        # Add the face encoding to the list of enrolled students
                        student_images.append(new_face_encoding)
                        student_names.append(student_name)
                        print(f"Student {student_name} enrolled successfully.")
                        speak(f"Student {student_name} enrolled successfully.")
                        break  # Break after successful enrollment

                else:
                    print("⚠️ No face detected. Please try again.")
                    speak("No face detected. Please try again.")

            elif key == ord('q'):
                print("❌ Enroll process canceled.")
                break  # Break the loop if 'q' is pressed

    finally:
        # Ensure that the camera and windows are properly closed
        cap.release()
        cv2.destroyAllWindows()


    # Capture and process attendance (with resized camera display)
    # Capture and process attendance (with resized camera display)
def capture_and_process_attendance():
    student_images, student_names = load_student_images(students_folder)
    # Ensure the CSV contains the "Date" column and load or create it
    if not os.path.exists(attendance_file):
        df = pd.DataFrame(columns=["Name", "Date", "Morning Time", "Evening Time"])
        df.to_csv(attendance_file, index=False, encoding="utf-8")  # Create the file if it doesn't exist
    else:
        df = pd.read_csv(attendance_file, encoding="utf-8", on_bad_lines="skip")

    # Replace with the URL of your mobile camera stream (IP Webcam)
    mobile_camera_url = "http://192.168.71.164:8080/video"  # Example: http://192.168.1.5:8080/video

    while True:
        choice = input(
            " 1.Capture Attendance \n 2.Enroll New Student \n 3.View Today's Attendance \n 4.View Attendance for selected Date \n 5.Count Classes Attended \n 6.Quit \nEnter your choice: ").strip()

        if choice == "1":
            cap = None
            start_time = time.time()
            timeout = 2  # Reduced timeout to 2 second for quicker fallback to system camera

            # Attempt to connect to IP Webcam with a short timeout
            while cap is None and time.time() - start_time < timeout:
                cap = cv2.VideoCapture(mobile_camera_url)  # Use IP Webcam stream
                if cap.isOpened():
                    print("Connected to IP Webcam.")
                    break
                time.sleep(0.1)  # Reduced wait time to 100ms between retries
                print("⏳ Trying to connect to IP Webcam...")

            # If IP Webcam is not available, fallback to system camera immediately
            if cap is None or not cap.isOpened():
                print("⚠️ IP Webcam not found or connection failed, falling back to system camera.")
                cap = cv2.VideoCapture(0)  # Use system camera if IP Webcam is not available

            print("📷 Press 's' to capture an image and mark attendance. Press 'q' to quit.")

            while True:
                ret, frame = cap.read()
                if not ret:
                    break

                # Resize the frame to half the original width (adjust the height proportionally)
                height, width = frame.shape[:2]
                resized_frame = cv2.resize(frame, (width // 2, height // 2))  # Half the width and height

                # Display the resized frame
                cv2.imshow("Press 's' to capture", resized_frame)

                key = cv2.waitKey(1) & 0xFF

                if key == ord('s'):
                    rgb_frame = cv2.cvtColor(resized_frame, cv2.COLOR_BGR2RGB)
                    face_locations = face_recognition.face_locations(rgb_frame)
                    face_encodings = face_recognition.face_encodings(rgb_frame, face_locations)

                    if face_encodings:
                        face_encoding = face_encodings[0]
                        matches = face_recognition.compare_faces(student_images, face_encoding)

                        if True in matches:
                            first_match_index = matches.index(True)
                            student_name = student_names[first_match_index]
                            print(f"👤 Student {student_name} detected. ")

                            # Mark attendance
                            df = mark_attendance(student_name, df)
                        else:
                            print("❌ No matching face found. Try again.")
                            speak("No matching face found. Try again.")
                    else:
                        print("⚠️ No face detected. Try again.")
                        speak("No face detected. Try again.")

                    cap.release()
                    cv2.destroyAllWindows()
                    break

                elif key == ord('q'):
                    cap.release()
                    cv2.destroyAllWindows()
                    break

        elif choice == "2":
            enroll_new_student(student_images, student_names)

        elif choice == "3":
            view_todays_attendance(df)

        elif choice == "4":
            view_attendance_for_date(df)

        elif choice == "5":
            count_classes_attended(df)

        elif choice == "6":
            print("👋 Thank you for using and have a nice day!!. ")
            speak("Thank you for using and have a nice day!!.")
            df.to_csv(attendance_file, index=False, encoding="utf-8")  # Save the updated attendance before quitting
            break

        else:
            print("⚠️ Invalid option, please try again.")
            speak("Invalid option, please try again.")


if __name__ == "__main__":
    capture_and_process_attendance()
