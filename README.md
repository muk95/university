# 임폴트구성
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
# 전역변수
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

# -------------------------------------------------------------------------------
# 기능-배경을 찾음
# -------------------------------------------------------------------------------
def run_avg(image, accumWeight, bg):
    # 배경 초기화
    if bg is None:
        bg = image.copy().astype("float")
        return bg

    # 가중평균 초기화
    cv2.accumulateWeighted(image, bg, accumWeight)
    return bg


# -------------------------------------------------------------------------------
# 기능
# -------------------------------------------------------------------------------
def segment(image, bg, dire, threshold=25):
    # 배경과 차이점 찾기
    diff = cv2.absdiff(bg.astype("uint8"), image)

    # 임계점 설정
    thresholded = cv2.threshold(diff, threshold, 255, cv2.THRESH_BINARY)[1]

    # 윤관선 얻기
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
        if len(cnts) > weight-1   :
            l = True
            if f is False and b is False and r is False:
                print(3)
                # 시리얼
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
                # 시리얼
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
                # 시리얼
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
                # 시리얼
                arduino.write(bytearray(backward_data))
                sleep(0.005)
        else:
            b = False
    count_s = str(len(cnts))
    # 윤곽선 발견 불가시 기본으로
    if len(cnts) == 0:
        if f is False and l is False and r is False and b is False:
            print(0)
            arduino.write(bytearray(stop_data))
            sleep(0.005)
    else:
        # 영역에 기반하여 손 최대 영역
        segmented = max(cnts, key=cv2.contourArea)
        return (thresholded, segmented)


# -------------------------------------------------------------------------------
# 메인기능
# -------------------------------------------------------------------------------

# 시리얼
arduino = serial.Serial('COM4', 9600)

if __name__ == "__main__":
    # 누적무게를 초기화해준다
    accumWeight = 0.5

    # 웹캠을켜준다
    camera = cv2.VideoCapture(0)

    # 관심영역 좌표
    top, right, bottom, left = 250, 500, 350, 610

    # 프레임을 초기화해줌
    num_frames = 0

    # 측정된것을 표시
    calibrated = False

    # 중단 할떄까지 루핑을계속함
    while (True):
        # 현재프레임을 얻음
        (grabbed, frame) = camera.read()

        # 프레임사이즈조정
        frame = imutils.resize(frame, width=700)

        # 프레임뒤집어 좌우반전이 아닌지 확인
        frame = cv2.flip(frame, 1)

        # 프레임 복제
        clone = frame.copy()

        # 프레임의 높이와 폭을 얻음
        (height, width) = frame.shape[:2]

        # 관심 영역얻음
        roi_l = frame[10:110, 400:500]
        roi_r = frame[10:110, 560:690]
        roi_f = frame[10:110, 10:200]
        roi_b = frame[400:500, 10:200]
        roi = frame[0:500, 0:700]

        # 관심영역을 그레이스케일 그리고 흐리게 변환을
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

        # weight 평균 모델이 보정되도록 배경을 얻고, 임계 값에 도달 할 때까지 계속 관찰
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
            # 손을 인식하는 구역을 세분화
            hand_l = segment(gray_l, bg_l, "l")
            hand_b = segment(gray_b, bg_b, "b")
            hand_r = segment(gray_r, bg_r, "r")
            hand_f = segment(gray_f, bg_f, "f")

            # 손을 인식하는 구역이 세분화 되어 있는지 확인
            if hand_l is not None:
                # 그렇다면 임계 값 이미지와 분할 된 구역의 압축을 푼다

                (thresholded, segmented) = hand_l

                # 분할 된 영역을 그려 프레임을 표시
                cv2.drawContours(clone, [segmented + (500, 250)], -1, (0, 0, 255))

                # 임계 값 이미지 표시
                cv2.imshow("Thesholded", thresholded)

            if hand_r is not None:
                # 그렇다면 임계 값 이미지와 분할 된 구역의 압축을 푼다

                (thresholded, segmented) = hand_r

                # 분할 된 영역을 그려 프레임을 표시
                cv2.drawContours(clone, [segmented + (580, 350)], -1, (0, 0, 255))

                # 임계 값 이미지 표시
                cv2.imshow("Thesholded", thresholded)
            if hand_f is not None:
                # 그렇다면 임계 값 이미지와 분할 된 구역의 압축을 푼다

                (thresholded, segmented) = hand_f

                # 분할 된 영역을 그려 프레임을 표시
                cv2.drawContours(clone, [segmented + (10, 10)], -1, (0, 0, 255))

                # 임계 값 이미지 표시
                cv2.imshow("Thesholded", thresholded)

            if hand_b is not None:
                # 그렇다면 임계 값 이미지와 분할 된 구역의 압축을 푼다

                (thresholded, segmented) = hand_b

                # 분할 된 영역을 그려 프레임을 표시
                cv2.drawContours(clone, [segmented + (10, 400)], -1, (0, 0, 255))

                # 임계 값 이미지 표시
                cv2.imshow("Thesholded", thresholded)

        # 손의 부분을 그려줌

            cv2.rectangle(clone, (500, 250), (610, 350), (0, 255, 0), 2)
            cv2.rectangle(clone, (580, 350), (690, 450), (0, 255, 0), 2)
            cv2.rectangle(clone, (10, 10), (200, 110), (0, 255, 0), 2)
            cv2.rectangle(clone, (10, 400), (200, 500), (0, 255, 0), 2)

        # 프레임 수를 증가
        num_frames += 1

        # 분할 된 손 구역으로 프레임을 표시
        cv2.imshow("Video Feed", clone)

        # 사용자의 키누름감지
        keypress = cv2.waitKey(1) & 0xFF

        # q를 누르면 looping이 정지됨
        if keypress == ord("q"):
            break

# 메모리비우기
camera.release()
cv2.destroyAllWindows()

