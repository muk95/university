import cv2
import serial
import imutils
import numpy as np
from time import sleep

forward_data =  [2,1,0,3]
backward_data =  [2,2,0,3]
left_data =  [2,3,0,3]
right_data =  [2,4,0,3]
stop_data =  [2,0,0,3]
forwardleft_data =  [2,5,0,3]
forwardright_data =  [2,6,0,3]

bg = None
bg_l = None
bg_r = None
bg_f = None
bg_b = None
l = False
r = False
f = False
b = False
st = False

def run_avg(image, accumWeight, bg):
   
    if bg is None:
        bg = image.copy().astype("float")
        return bg

    
    cv2.accumulateWeighted(image, bg, accumWeight)
    return bg

def segment(image, bg, dire, threshold=25):
   
    diff = cv2.absdiff(bg.astype("uint8"), image)

  
    thresholded = cv2.threshold(diff, threshold, 255, cv2.THRESH_BINARY)[1]


    (_, cnts, _) = cv2.findContours(thresholded.copy(), cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    weight = 2
    global l
    global r
    global f
    global b

    if f is True and l is True:
        print(5)
        arduino.write(bytearray(forwardleft_data))
        sleep(0.005)
        if l is False and f is True:
            print(00000)
            arduino.write(bytearray(forward_data))
            sleep(0.005)
    if f is True and r is True:
        print(6)
        arduino.write(bytearray(forwardright_data))
        sleep(0.005)
        if r is False and f is True:
            print(00000)
            arduino.write(bytearray(forward_data))
            sleep(0.005)

    if dire is "l":

        cv2.putText(clone, str(len(cnts)), (70, 45), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 0, 255), 2)
        if len(cnts) > weight-1:
            l = True
            if f is False and b is False and r is False:
                print(3)
                arduino.write(bytearray(left_data))
                sleep(0.005)
        else:
            l = False
    elif dire is "r":
        cv2.putText(clone, str(len(cnts)), (110, 45), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 0, 255), 2)
        if len(cnts) > weight:
            r = True
            if f is False and b is False and l is False:
                print(4)
                arduino.write(bytearray(right_data))
                sleep(0.005)
        else:
            r = False
    elif dire is "f":
        cv2.putText(clone, str(len(cnts)), (150, 45), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 0, 255), 2)
        if len(cnts) > weight-1:
            f = True
            if b is False and l is False and r is False:
                print(1)
                arduino.write(bytearray(forward_data))
                sleep(0.005)
        else:
            f = False
    elif dire is "b":
        cv2.putText(clone, str(len(cnts)), (190, 45), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 0, 255), 2)
        if len(cnts) > weight:
            b = True
            if f is False and l is False and r is False:
                print(2)
                arduino.write(bytearray(backward_data))
                sleep(0.005)
        else:
            b = False
    count_s = str(len(cnts))
    if len(cnts) == 0:
        if f is False and l is False and r is False and b is False:
            print(0)
            arduino.write(bytearray(stop_data))
            sleep(0.005)
    else:
        segmented = max(cnts, key=cv2.contourArea)
        return (thresholded, segmented)


arduino = serial.Serial('COM4', 9600)

if __name__ == "__main__":
    accumWeight = 0.5
    camera = cv2.VideoCapture(0)

    top, right, bottom, left = 250, 500, 350, 610

    num_frames = 0
    calibrated = False
    while (True): 
        (grabbed, frame) = camera.read()


        frame = imutils.resize(frame, width=700)

       
        frame = cv2.flip(frame, 1)

      
        clone = frame.copy()

       
        (height, width) = frame.shape[:2]

        
        roi_l = frame[10:130, 380:530]
        roi_r = frame[10:130, 540:690]
        roi_f = frame[10:120, 10:200]
        roi_b = frame[390:500, 10:200]
        roi = frame[0:500, 0:700]

        
        gray_l = cv2.cvtColor(roi_l, cv2.COLOR_BGR2GRAY)
        gray_l = cv2.GaussianBlur(gray_l, (7, 7), 0)
        gray_r = cv2.cvtColor(roi_r, cv2.COLOR_BGR2GRAY)
        gray_r = cv2.GaussianBlur(gray_r, (7, 7), 0)
        gray_f = cv2.cvtColor(roi_f, cv2.COLOR_BGR2GRAY)
        gray_f = cv2.GaussianBlur(gray_f, (7, 7), 0)
        gray_b = cv2.cvtColor(roi_b, cv2.COLOR_BGR2GRAY)
        gray_b = cv2.GaussianBlur(gray_b, (7, 7), 0)
        gray = cv2.cvtColor(roi, cv2.COLOR_BGR2GRAY)
        gray = cv2.GaussianBlur(gray, (7, 7), 0)

      
        if num_frames < 30:
            bg_l = run_avg(gray_l, accumWeight, bg_l)
            bg_r = run_avg(gray_r, accumWeight, bg_r)
            bg_f = run_avg(gray_f, accumWeight, bg_f)
            bg_b = run_avg(gray_b, accumWeight, bg_b)
            if num_frames == 1:
                print("[STATUS] please wait! calibrating...")
            elif num_frames == 29:
                print("[STATUS] calibration successfull...")
        else:
           
            hand_l = segment(gray_l, bg_l, "l")
            hand_b = segment(gray_b, bg_b, "b")
            hand_r = segment(gray_r, bg_r, "r")
            hand_f = segment(gray_f, bg_f, "f")

          
            if hand_l is not None:
                
                (thresholded, segmented) = hand_l

              
                cv2.drawContours(clone, [segmented + (380, 10)], -1, (0, 0, 255))

                
                cv2.imshow("Thesholded", thresholded)

            if hand_r is not None:
                

                (thresholded, segmented) = hand_r

              
                cv2.drawContours(clone, [segmented + (540, 10)], -1, (0, 0, 255))

               
                cv2.imshow("Thesholded", thresholded)
            if hand_f is not None:
                

                (thresholded, segmented) = hand_f

              
                cv2.drawContours(clone, [segmented + (10, 10)], -1, (0, 0, 255))

                
                cv2.imshow("Thesholded", thresholded)

            if hand_b is not None:
               

                (thresholded, segmented) = hand_b

               
                cv2.drawContours(clone, [segmented + (10, 400)], -1, (0, 0, 255))

                cv2.imshow("Thesholded", thresholded)

      

            cv2.rectangle(clone, (380, 10), (530, 130), (255, 0, 0), 2)
            cv2.rectangle(clone, (540, 10), (690, 130), (255, 0, 0), 2)
            cv2.rectangle(clone, (10, 10), (200, 120), (255, 0, 0), 2)
            cv2.rectangle(clone, (10, 390), (200, 500), (255, 0, 0), 2)

        
        num_frames += 1

       
        cv2.imshow("Video Feed", clone)

      
        keypress = cv2.waitKey(1) & 0xFF

    
        if keypress == ord("q"):
            break


camera.release()
cv2.destroyAllWindows()



