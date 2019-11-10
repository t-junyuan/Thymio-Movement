# Thymio-Movement
Thymio movement, connect to Firebase, and RFID sensor code

import RPi.GPIO as GPIO
from libdw import pyrebase
from libdw import sm
from pythymiodw import *
from pythymiodw import io
from pythymiodw.sm import *
from boxworld import thymio_world
from mfrc522 import SimpleMFRC522


### SET UP FIREBASE ###

url = "https://tim-by-design.firebaseio.com/"
apikey = "AIzaSyAlT5K9Dq9DEmDyKG4A7BV_d5AEMyPnJtU"

config = {
    "apiKey": apikey,
    "databaseURL": url,
}

firebase = pyrebase.initialize_app(config)
db = firebase.database() 


### SET UP RFID SCANNER ###
reader = SimpleMFRC522()


### PROMPT USER ###
cohort_class = input("Enter class: ")

valid_classes = db.child("Seating Arrangement").get().val()

while True:
    if cohort_class not in valid_classes: # Checks for accurate input
        print("Error: Please input a valid class.")
        cohort_class = input("Enter class: ")
    else:
        break


### THYMIO MOVEMENT ###
def instructions():
    
    class MySMClass(sm.SM):
        
        start_state = "free"
        
        def get_next_values(self, state, inp):
            
            # Get values from the bottom sensors of the Thymio
            ground_reflected = inp.prox_ground.reflected
            left_reflected = ground_reflected[0]
            right_reflected = ground_reflected[1]
            
            # After measuring, we found that value below 450 means that the ground is black
            black = 450
            
            # First state: "free"; Thymio has not found the line that it has to follow
            if state == "free":
                # Obtain instructions for whether the Thymio should be moving or not
                # These instructions are updated on Firebase using the Kivy application
                movement = db.child("Movement").child(cohort_class).get().val()
                
                if movement == "move": # Thymio should be moving
                    # Thymio turns anti-clockwise if the right or left sensor detects the black line
                    # This positions the Thymio to follow the line properly
                    if right_reflected <= black or left_reflected <= black:
                        output = io.Action(0, 0)
                        next_state = "turning_acw"
                        return next_state, output
                    else: # If Thymio still cannot find the line, it continues searching for the line
                        output = io.Action(0.03, 0)
                        next_state = "free"
                        return next_state, output
                
                if movement == "stop": # Thymio should not be moving
                    output = io.Action(0, 0)
                    next_state = state # State remains the same so that the Thymio continues where it left off when it is set to move again
                    return next_state, output
            
            # Second state: "turning_cw"; Thymio turns clockwise to reposition itself properly on the line
            if state == "turning_cw":
                # Obtain instructions for whether the Thymio should be moving or not
                # These instructions are updated on Firebase using the Kivy application
                movement = db.child("Movement").child(cohort_class).get().val()
                
                if movement == "move": # Thymio should be moving
                    # Thymio turns clockwise until the right sensor detects white and left sensor detects black
                    # This is when the Thymio is properly back on the line
                    if right_reflected >= black and left_reflected <= black:
                        output = io.Action(0, 0)
                        next_state = "follow"
                        return next_state, output
                    else: # Thymio continues to turn clockwise until it repositions itself properly
                        output = io.Action(0, -0.1)
                        next_state = "turning_cw"
                        return next_state, output
                
                if movement == "stop": # Thymio should not be moving
                    output = io.Action(0, 0)
                    next_state = state # State remains the same so that the Thymio continues where it left off when it is set to move again
                    return next_state, output
                
            # Third state: "turning_acw"; Thymio turns anti-clockwise to reposition itself properly on the line
            if state == "turning_acw":
                # Obtain instructions for whether the Thymio should be moving or not
                # These instructions are updated on Firebase using the Kivy application
                movement = db.child("Movement").child(cohort_class).get().val()
                
                if movement == "move": # Thymio should be moving
                    # Thymio turns anti-clockwise until the right sensor detects white and left sensor detects black
                    # This is when the Thymio is properly back on the line
                    if right_reflected >= black and left_reflected <= black:
                        output = io.Action(0, 0)
                        next_state = "follow"
                        return next_state, output
                    else: # Thymio continues to turn anti-clockwise until it repositions itself properly
                        output = io.Action(0, 0.1)
                        next_state = "turning_acw"
                        return next_state, output
                
                if movement == "stop": # Thymio should not be moving
                    output = io.Action(0, 0)
                    next_state = state # State remains the same so that the Thymio continues where it left off when it is set to move again
                    return next_state, output
                
            # Fourth state: "follow"; Thymio is on the line
            if state == "follow":
                # Obtain instructions for whether the Thymio should be moving or not
                # These instructions are updated on Firebase using the Kivy application
                movement = db.child("Movement").child(cohort_class).get().val()
                
                if movement == "move": # Thymio should be moving
                    # If the right sensor detects white and left sensor detects black, the Thymio knows that it is properly on the line
                    # Thymio continues following the line
                    if right_reflected >= black and left_reflected <= black:
                        output = io.Action(0.05, 0)
                        next_state = "follow"
                        return next_state, output
                    # If the right and left sensors detect black, the Thymio knows that it is no longer properly on the line
                    # Thymio turns clockwise until it is repositioned back on the line
                    elif right_reflected <= black and left_reflected <= black:
                        output = io.Action(0, 0)
                        next_state = "turning_cw"
                        return next_state, output
                    # If the right sensor detects black and left sensor detects white, the Thymio knows that it is no longer properly on the line
                    # Thymio turns anti-clockwise until it is repositioned back on the line
                    elif right_reflected <= black and left_reflected >= black:
                        output = io.Action(0, 0)
                        next_state = "turning_acw"
                        return next_state, output
                    # If the right and left sensors detect white, the Thymio knows that it is no longer properly on the line
                    # Thymio turns anti-clockwise until it is repositioned back on the line
                    elif right_reflected >= black and left_reflected >= black:
                        output = io.Action(0, 0)
                        next_state = "turning_acw"
                        return next_state, output
                
                if movement == "stop": # Thymio should not be moving
                    output = io.Action(0, 0)
                    next_state = state # State remains the same so that the Thymio continues where it left off when it is set to move again
                    return next_state, output
                
                if movement == "done": # Thymio has finished taking the attendance of all the students in the class
                    output = io.Action(0, 0)
                    next_state = "done"
                    return next_state, output
        
        # Stops the state machine from using inputs from the Thymio
        def done(self, state):
            if state == "done":
                return True
            else:
                return False
    
    MySM = MySMClass()
    m = ThymioSMReal(MySM)
    try:
        m.start()
    except KeyboardInterrupt:
        m.stop()


### READING ATTENDANCE USING RFID TAGS ###
# Obtain the list of the student IDs of the students in the class in order of seating arrangement
students = list(db.child("Seating Arrangement").child(cohort_class).get().val())

while True:
    instructions() # Function that tells Thymio how to move
    
    # The loop continues only until all students in the class have already had their attendance taken
    i = 0
    read = []
    while i < (len(students)):
        # The reader reads two values: a default RFID number and the student ID that we wrote onto the tag
        rfid_num, student_id = reader.read()
        
        student_id = student_id.rstrip() # Strip the extra spaces that is read by the reader; cleans up the student ID for comparison with data on Firebase
        
        if student_id not in read: # This ensures that the same tag is not read multiple times

            current_seat = i + 1
        
            # At each seat, compare if student ID read matches the student ID on Firebase
            if len(student_id) != 7: # Student ID for SUTD students are 7 characters long
                # Default tag is read; student ID written onto default tag starts from "0", and will never reach 7 characters
                # Attendance status on Firebase remains as "Absent"
                print("Seat number {} appears to be empty. Update attendance manually if necessary.".format(current_seat))
                read.append(student_id)
                instructions()
                i += 1
            
            elif student_id == db.child("Seating Arrangement").child(cohort_class).child(str(i)).get().val():
                # If student IDs match, set attendance status on Firebase to "Present"
                db.child("Students' Information").child(cohort_class).child(student_id).child("Attendance").set("Present")
                print("Attendance taken for seat number {} with student ID {}.".format(current_seat, student_id))
                read.append(student_id)
                instructions()
                i += 1
            
            else:
                # If student IDs do not match, set attendance status on Firebase to "Error"
                db.child("Students' Information").child(cohort_class).child(student_id).child("Attendance").set("Error")
                print("Please check student in seat {}. Update attendance manually if necessary.".format(current_seat))
                read.append(student_id)
                instructions()
                i += 1
    
    break # After attendance has been taken for all the students in the class, break out of the while loop
    
db.child("Movement").child(cohort_class).set("done") # Update Firebase to inform Thymio that it has finished taking attendance
instructions() # Thymio reacts accordingly to the end of the attendance-taking process

GPIO.cleanup()
