import sys
import time

from PyQt5.QtWidgets import *
from PyQt5.QtGui import *
from PyQt5.QtCore import *
from PyQt5 import uic
from pytimekr import pytimekr

import csv
import json
import string
import random

import requests

from bs4 import BeautifulSoup

import threading

import urllib3
from urllib3.exceptions import InsecureRequestWarning
urllib3.disable_warnings(InsecureRequestWarning)

# UI 파일 위치
form_class = uic.loadUiType('NewMacroGajwaUI.ui')[0]

login_url = 'https://www.icsports.or.kr/fmcs/4?login_check=skip'

def Reservation(self):
    # 세션 객체 생성
    session = requests.Session()

    login_page = session.get(login_url, verify=False)

    soup = BeautifulSoup(login_page.text, 'html.parser')
    security_token = soup.find('input', {'name': 'SecurityToken'}).get('value')

    # 로그인 폼 파싱
    login_data = {
        'SecurityToken': security_token,
        'user_id': self.selectedLoginUserInfo['id'],
        'user_password': self.selectedLoginUserInfo['pwd']
    }

    # 로그인 요청 보내기
    response = session.post(login_url, params=login_data, verify=False)

    # 로그인 성공 여부 확인
    if response.status_code == 200:
        soup = BeautifulSoup(response.text, 'html.parser')

        res = soup.find('div', {'class': 'box_text'})

        if not res:
            self.setLog(self.cb_Users.currentText() + ' 로그인 성공')
        else:
            self.setLog(self.cb_Users.currentText() + ' 로그인 실패')
            self.stopMacroAction()
            return
    else:
        self.setLog(self.cb_Users.currentText() + ' 로그인 실패')
        self.stopMacroAction()
        return

    self.setLog('매크로 돌리는 중...')

    # 17시 이후 조명 사용 필수네.
    # 만약 비 오거나그러면 거의 필수로 써야함
    # 한번에 최대 3시간 예약 가능
    macroInfoList = self.macroInfoList

    # 대표자
    teamName = self.cb_Users.currentText()
    # 이용 날짜
    usedDate = self.cal_UsedDate.selectedDate().toString('yyyyMMdd')
    # 시작 할 코트
    startCourtCount = self.lw_2.count()
    # 시작 시간
    startTime = int(self.cb_StartTime.currentText().split(':')[0])
    # 종료 시간
    endTime = int(self.cb_EndTime.currentText().split(':')[0])

    courtInfos = {
        '실내 1': [1, 1, '01', '02', 'A'],
        '실내 2': [2, 2, '01', '02', 'A'],
        '실내 3': [3, 3, '01', '02', 'A'],
        '실외 1': [17, 4, '02', '03', 'B'],
        '실외 2': [18, 5, '02', '03', 'B'],
        '실외 3': [19, 6, '02', '03', 'B'],
    }

    if startCourtCount > 1:
        for infinite in range(10000000000):
            for i in range(self.lw_2.count()):
                datas = courtInfos[self.lw_2.item(i).text()]

                timeText = getTimeNo(macroInfoList, int(startTime), int(endTime), datas[1])
                if datas[4] == 'A':
                    baseUrl = 'https://www.icsports.or.kr/fmcs/6?facilities_type=C&center=ICGYM&part=' + str(datas[2])
                    fullUrl = baseUrl + '&base_date=' + usedDate + '&action=write&place=' + str(
                        datas[0]) + '&comcd=ICGYM&part_cd=' + str(datas[3]) + '&place_cd=' + str(
                        datas[0]) + '&time_no=' + timeText + '&rent_date=' + usedDate
                else:
                    baseUrl = 'https://www.icsports.or.kr/fmcs/6?action=write&comcd=ICGYM&part_cd=' + str(datas[3]) +'&place_cd=' + str(datas[0]) + '&time_no='
                    fullUrl = baseUrl + timeText + '&rent_date=' + usedDate

                try:
                    if self.isStart:
                        response = session.get(fullUrl, verify=False)

                        if response.text.__contains__('선택업장 의 일일 최대 장소선택 제한(1장소) 이상 예약 할 수 없습니다.'):
                            self.setLog('예약제한 1회를 초과하였습니다.')
                            self.stopMacroAction()
                            return

                        soup = BeautifulSoup(response.text, 'html.parser')

                        data = setReservationInfo(soup, teamName)

                        reservationUrl = 'https://www.icsports.or.kr/fmcs/6?action=write_proc'

                        headers = setReservationHeader(fullUrl)

                        # session.post(reservationUrl, headers=headers, params=data, verify=False)
                        print(self.lw_2.item(i).text())
                        print(fullUrl)
                    else:
                        session.get('https://www.icsports.or.kr/fmcs/17', verify=False)
                        session.close()
                        self.setLog('로그아웃 성공')
                        return;
                except:
                    continue
    else:
        datas = courtInfos[self.lw_2.item(0).text()]
        timeText = getTimeNo(macroInfoList, int(startTime), int(endTime), datas[1])

        if datas[4] == 'A':
            baseUrl = 'https://www.icsports.or.kr/fmcs/6?facilities_type=C&center=ICGYM&part=' + str(datas[2])
            fullUrl = baseUrl + '&base_date=' + usedDate + '&action=write&place=' + str(
                datas[0]) + '&comcd=ICGYM&part_cd=' + str(datas[3]) + '&place_cd=' + str(
                datas[0]) + '&time_no=' + timeText + '&rent_date=' + usedDate
        else:
            baseUrl = 'https://www.icsports.or.kr/fmcs/6?action=write&comcd=ICGYM&part_cd=' + str(
                datas[3]) + '&place_cd=' + str(datas[0]) + '&time_no='
            fullUrl = baseUrl + timeText + '&rent_date=' + usedDate

        for i in range(10000000000):
            try:
                if self.isStart:
                    response = session.get(fullUrl, verify=False)
                    if response.text.__contains__('선택업장 의 일일 최대 장소선택 제한(1장소) 이상 예약 할 수 없습니다.'):
                        self.setLog('예약제한 1회를 초과하였습니다.')
                        self.stopMacroAction()
                        return

                    soup = BeautifulSoup(response.text, 'html.parser')

                    data = setReservationInfo(soup, teamName)

                    reservationUrl = 'https://www.icsports.or.kr/fmcs/6?action=write_proc'

                    headers = setReservationHeader(fullUrl)

                    res = session.post(reservationUrl, headers=headers, params=data, verify=False)
                else:
                    session.get('https://www.icsports.or.kr/fmcs/17', verify=False)
                    session.close()
                    self.setLog('로그아웃 성공')
                    return;
            except:
                pass

def possibleReservationCheck(courtNum, usedDate, startTime, endTime, isWeek):
    checkStartTimeIArr = []
    if isWeek:
        if int(endTime) == 22:
            checkStartTimeIArr = [19, 21]
        else:
            if int(startTime) == int(endTime) - 2:
                checkStartTimeIArr = [int(startTime)]
            else:
                checkStartTimeIArr = [int(startTime), int(endTime) - 2]
    else:
        for i in range(int(startTime), int(endTime)):
            checkStartTimeIArr.append(i)

    checkStartTimeSArr = []

    for checkStartTimeI in checkStartTimeIArr:
        if int(checkStartTimeI) == 9:
            timeText = '0' + str(checkStartTimeI) + ':00'
        else:
            timeText = str(checkStartTimeI) + ':00'

        checkStartTimeSArr.append(timeText)

    placeCode = str(courtNum)
    url = 'https://www.ycs.or.kr/rest/facilities/place_time_state_list?company_code=YCS04&part_code=02&place_code=' + \
          placeCode + '&base_date=' + usedDate + '&rent_type=1001'

    result = requests.get(url, verify=False)
    soup = BeautifulSoup(result.text, 'html.parser')
    datas = json.loads(soup.text)

    isCheckRes = True
    for data in datas:
        for checkStartTimeS in checkStartTimeSArr:
            if data['start_time'] == checkStartTimeS and data['use_yn'] == 'Y':
                isCheckRes = False

    return isCheckRes

def setReservationInfo(soup, teamName):
    securityToken = getInputValue(soup, 'input', 'SecurityToken')

    placeCode = getInputValue(soup, 'input', 'place_code')

    memNo = getInputValue(soup, 'input', 'mem_no')

    startDate = getInputValue(soup, 'input', 'start_date')

    endDate = getInputValue(soup, 'input', 'end_date')

    timeDatas = getInputValue(soup, 'input', 'time_datas')

    totalAmount = getInputValue(soup, 'input', 'total_amount')

    memNm = getInputValue(soup, 'input', 'mem_nm')

    mobileTel = getInputValue(soup, 'input', 'mobile_tel')

    return setReservationData2(securityToken, placeCode, memNo, startDate, endDate, timeDatas, totalAmount, memNm,
                               teamName, mobileTel)

def getInputValue(soup, tagName, propName):
    inputTag = soup.find(tagName, {'name': propName})
    value = inputTag['value']

    return value

def getTimeNo(macroInfoList, rStartTime, rEndTime, rCourtNum):
    # 시간 [9,10,11,12]
    startTimeArr = []
    for i in range(rStartTime, rEndTime):
        startTimeArr.append(i)

    res = ''
    # 658;평일 8회;1600;1700;1|659;평일 9회;1700;1800;1
    # 658;평일 7회;1600;1700;1|659;평일 8회;1700;1800;1
    for startTime in startTimeArr:
        timeText = ''
        if int(startTime) == 9:
            timeText = '0' + str(startTime) + '00%3B' + str(startTime + 1) + '00%3B'
        else:
            timeText = str(startTime) + '00%3B' + str(startTime + 1) + '00%3B'

        res += macroInfoList[int(rCourtNum)][startTime - 8] + '%3B' + str(startTime - 5) + '%ED%9A%8C%3B' + timeText + '1%7C'

    return res[:-3]

def setReservationData2(security_token, place_code, mem_no, start_date, end_date, time_datas, total_amount, mem_nm, users, mobile_tel):
    _LENGTH = 6
    string_pool = string.digits
    captcha = ""
    for i in range(_LENGTH):
        captcha += random.choice(string_pool)

    data = {
        'SecurityToken': security_token,
        'type': '',
        'company_code': 'ICGYM',
        'part_code': '02',
        'place_code': place_code,
        'guest_yn': 'N',
        'mem_no': mem_no,
        'team_no': '0',
        'start_date': start_date,
        'start_time': '',
        'end_date': end_date,
        'end_time': '',
        'time_datas': time_datas,
        'total_amount': total_amount,
        'mem_nm': mem_nm,
        'team_nm': users,
        'team_yn': 'N',
        'users': '4',
        'mobile_tel': mobile_tel,
        'tel': '',
        'captcha': captcha,
        'type_cd': '1001',
        'title': '경기',
        'purpose': '실력향상',
       'attachfile_count': '1',
        'attachfile': '(바이너리)',
        'place_accessory_1_cd': 'I000025',
        'place_accessory_1_count': '2',
        'agree_use': 'Y'
    }

    return data


def setReservationHeader(referer):
    data = {
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3; q=0.7",
        "Accept-Encoding": "gzip, deflate, br",
        "Accept-Language": "en-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7",
        "Content-Type": "multipart/form-data; boundary=----WebKitFormBoundaryuA9l1e4hBLdeUaWn",
        "sec-ch-ua": "\"Chromium\";v=\"114\", \"Google Chrome\";v=\"114\", \"Not:A-Brand\";v=\"8\"",
        "sec-ch-ua-mobile": "?0",
        "sec-ch-ua-platform": "\"Windows\"",
        "Sec-Fetch-Dest": "document",
        "Sec-Fetch-Mode": "navigate",
        "Sec-Fetch-Site": "same-origin",
        "Sec-Fetch-User": "?1",
        "Upgrade-Insecure-Requests": "1",
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36"
    }

    return data

def setReservationHeader2(referer):
    data = {
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3; q=0.7",
        "Accept-Encoding": "gzip, deflate, br",
        "Accept-Language": "en-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7",
        "Cache-Control": "no-cache",
        "Connection": "keep-alive",
        "Content-Length": "3048",
        "Content-Type": "multipart/form-data; boundary=----WebKitFormBoundaryuA9l1e4hBLdeUaWn",
        "Host": "www.icsports.or.kr",
        "Origin": "https://www.icsports.or.kr",
        "Pragma": "no-cache",
        "Referer": referer,
        "sec-ch-ua": "\"Chromium\";v=\"114\", \"Google Chrome\";v=\"114\", \"Not:A-Brand\";v=\"8\"",
        "sec-ch-ua-mobile": "?0",
        "sec-ch-ua-platform": "\"Windows\"",
        "Sec-Fetch-Dest": "document",
        "Sec-Fetch-Mode": "navigate",
        "Sec-Fetch-Site": "same-origin",
        "Sec-Fetch-User": "?1",
        "Upgrade-Insecure-Requests": "1",
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36"
    }

    return data

def gzipJson(payload):
    return payload


class WindowClass(QMainWindow, form_class):
    def __init__(self):
        super().__init__()

        self.isWeek = False
        self.isStart = False
        self.macroInfoList = None
        self.selectedLoginUserInfo = None

        fHolidays = pytimekr.holidays()  # holidays메소드는 리스트 형태로 관련값 반환
        holidays = []
        for fHoliday in fHolidays:
            holidays.append(fHoliday.strftime('%Y%m%d'))

        holidays.append('20220912')
        self.holidays = holidays

        self.setWindowIcon(QIcon('logo.png'))

        self.setupUi(self)

        # User객체에 User 추가
        with open('gajwaLoginInfos.txt', 'r', encoding='UTF8') as file:
            file_content = file.read()

        if file_content:
            userInfos = json.loads(file_content)
            self.cb_Users.addItem('')
            for userInfo in userInfos:
                self.cb_Users.addItem(userInfo['name'])
        else:
            self.cb_Users.addItem('등록 된 정보가 없습니다.')

        self.cb_Users.currentIndexChanged.connect(self.selectedLoginUser)

        self.cal_UsedDate.showToday()

        # 공휴일 표시
        fm = QTextCharFormat()
        fm.setForeground(Qt.red)
        fm.setBackground(Qt.yellow)

        # 날짜 선택
        self.cal_UsedDate.setGridVisible(True)
        for holiday in self.holidays:
            dHoliday = QDate.fromString(str(holiday), "yyyyMMdd")
            self.cal_UsedDate.setDateTextFormat(dHoliday, fm)

        self.cal_UsedDate.clicked.connect(self.calendarClicked)

        self.setReservationTime()
        self.setReservationEndTime()

        self.tb_Log.verticalScrollBar().setValue(self.tb_Log.verticalScrollBar().maximum())

        self.cb_StartTime.currentIndexChanged.connect(self.changeEndTimeComboBoxAction)

        self.lw_1.insertItem(0, '실내 1')
        self.lw_1.insertItem(1, '실내 2')
        self.lw_1.insertItem(2, '실내 3')
        self.lw_1.insertItem(3, '실외 1')
        self.lw_1.insertItem(4, '실외 2')
        self.lw_1.insertItem(5, '실외 3')

        self.btn_Add.clicked.connect(self.addCourt)
        self.btn_Remove.clicked.connect(self.removeCourt)

        self.le_UserPwd.setEchoMode(QLineEdit.Password)

        # 시작
        self.btn_Start.clicked.connect(self.startMacroAction)

        # 정지
        self.btn_Stop.clicked.connect(self.stopMacroAction)
        self.btn_Stop.setEnabled(False)

        self.btn_CheckNAdd.clicked.connect(self.checkUserValidation)

        self.btn_UserAdd.clicked.connect(self.addedNewUser)
        self.btn_UserAdd.setEnabled(False)

        # 종료 버튼 - btn_Exit
        self.btn_Exit.clicked.connect(QCoreApplication.instance().quit)

    # Function
    # ==========================================================================
    # addCourt
    def addCourt(self):
        lw1lIndex = self.lw_1.selectedIndexes()

        if len(lw1lIndex) > 0:
            self.lw_2.addItem(self.lw_1.selectedItems()[0].text())
            for modelIndex in lw1lIndex:
                self.lw_1.model().removeRow(modelIndex.row())

    # removeCourt
    def removeCourt(self):
        lw12Index = self.lw_2.selectedIndexes()

        if len(lw12Index) > 0:
            self.lw_1.addItem(self.lw_2.selectedItems()[0].text())
            for modelIndex in lw12Index:
                self.lw_2.model().removeRow(modelIndex.row())

    # setReservationTime
    def setReservationTime(self):
        self.cb_StartTime.clear()
        self.cb_EndTime.clear()

        self.isWeek = False
        for i in range(13):
            iStartTime = 9 + i

            sStartTime = str(iStartTime)

            if iStartTime < 10:
                sStartTime = '0' + sStartTime

            self.cb_StartTime.addItem(sStartTime + ":00")

    def setReservationEndTime(self):
        self.cb_EndTime.clear()

        startTimeIndex = self.cb_StartTime.currentIndex()

        for i in range(startTimeIndex, 13):
            iEndtime = 10 + i
            sEndTime = str(iEndtime)

            if iEndtime == 23:
                sEndTime = str(iEndtime - 1)

            self.cb_EndTime.addItem(sEndTime + ":00")

    # changeEndTimeComboBoxAction
    def changeEndTimeComboBoxAction(self):
        self.setReservationEndTime()

    # checkUserValidation
    def checkUserValidation(self):
        inputName = self.le_UserName.text()
        inputId = self.le_UserId.text()
        inputPwd = self.le_UserPwd.text()

        if not inputName:
            QMessageBox.warning(self, 'Warning', '이름을 입력하세요')
            self.le_UserName.setFocus()
            return

        if not inputId:
            QMessageBox.warning(self, 'Warning', '아이디를 입력하세요')
            self.le_UserId.setFocus()
            return

        if not inputPwd:
            QMessageBox.warning(self, 'Warning', '비밀번호를 입력하세요')
            self.le_UserPwd.setFocus()
            return

        # 세션 객체 생성
        session = requests.Session()

        login_page = session.get(login_url, verify=False)

        soup = BeautifulSoup(login_page.text, 'html.parser')
        security_token = soup.find('input', {'name': 'SecurityToken'}).get('value')

        # 로그인 폼 파싱
        login_data = {
            'SecurityToken': security_token,
            'user_id': inputId,
            'user_password': inputPwd
        }

        # 로그인 요청 보내기
        response = session.post(login_url, params=login_data, verify=False)

        # 로그인 성공 여부 확인
        if response.status_code == 200:
            soup = BeautifulSoup(response.text, 'html.parser')

            res = soup.find('div', {'class': 'box_text'})

            if not res:
                QMessageBox.information(self, 'Success', '로그인 정보 확인 완료\nADD 버튼을 통해 추가 해주세요.')

                self.setBtnCss(True)
            else:
                if res.find('p').text.find('회원 아이디가 존재하지 않습니다.') == -1:
                    QMessageBox.critical(self, 'Error', '회원 비밀번호가 일치하지 않습니다.')
                else:
                    QMessageBox.critical(self, 'Error', '회원 아이디가 존재하지 않습니다.')

                self.setBtnCss(False)
        else:
            QMessageBox.critical(self, 'Error', '입력 한 정보가 올바르지 않습니다.')
            self.setBtnCss(False)

    def setBtnCss(self, res):
        if res:
            self.btn_CheckNAdd.setStyleSheet("color: red;"
                                             "border-style: solid;"
                                             "border-width: 1px;"
                                             "border-color: red;"
                                             "border-radius: 3px;"
                                             "font: 16pt HY엽서L;"
                                             "text-decoration: line-through;")

            self.btn_UserAdd.setStyleSheet("color: green;"
                                           "border-style: solid;"
                                           "border-width: 1px;"
                                           "border-color: green;"
                                           "border-radius: 3px;"
                                           "font: 16pt HY엽서L;")

            self.le_UserName.setEnabled(False)
            self.le_UserId.setEnabled(False)
            self.le_UserPwd.setEnabled(False)
            self.btn_UserAdd.setEnabled(True)
        else:
            self.btn_UserAdd.setStyleSheet("color: red;"
                                             "border-style: solid;"
                                             "border-width: 1px;"
                                             "border-color: red;"
                                             "border-radius: 3px;"
                                             "font: 16pt HY엽서L;"
                                             "text-decoration: line-through;")

            self.btn_CheckNAdd.setStyleSheet("color: green;"
                                           "border-style: solid;"
                                           "border-width: 1px;"
                                           "border-color: green;"
                                           "border-radius: 3px;"
                                           "font: 16pt HY엽서L;")

            self.le_UserName.setEnabled(True)
            self.le_UserId.setEnabled(True)
            self.le_UserPwd.setEnabled(True)
            self.btn_UserAdd.setEnabled(False)

    # addedNewUser
    def addedNewUser(self):
        res = True

        # User 객체에 User 추가
        with open('gajwaLoginInfos.txt', 'r', encoding='UTF8') as file:
            file_content = file.read()

        if file_content:
            userInfos = json.loads(file_content)
            for userInfo in userInfos:
                if self.le_UserId.text() == userInfo['id']:
                    res = False

        if not res:
            QMessageBox.warning(self, 'Warning', '입력 한 정보가 이미 존재합니다.')

            self.le_UserName.clear()
            self.le_UserId.clear()
            self.le_UserPwd.clear()
            self.setBtnCss(False)

        else:
            inputName = self.le_UserName.text()
            inputId = self.le_UserId.text()
            inputPwd = self.le_UserPwd.text()

            if file_content:
                userInfos = json.loads(file_content)
                # Define the object to add to the file
                new_user = {"name": inputName, "id": inputId, "pwd": inputPwd}

                # Append the new object to the data
                userInfos.append(new_user)

                # Write the updated data to the file
                with open('gajwaLoginInfos.txt', 'w') as f:
                    json.dump(userInfos, f)
            else:
                new_user = [{"name": inputName, "id": inputId, "pwd": inputPwd}]

                with open('gajwaLoginInfos.txt', 'w') as f:
                    json.dump(new_user, f)

            QMessageBox.information(self, 'Success', '로그인 정보가 추가 되었습니다.')

            self.le_UserName.clear()
            self.le_UserId.clear()
            self.le_UserPwd.clear()
            self.setBtnCss(False)

            # User객체에 User 추가
            self.cb_Users.clear()

            with open('gajwaLoginInfos.txt', 'r', encoding='UTF8') as file:
                file_content = file.read()

            if file_content:
                userInfos = json.loads(file_content)
                self.cb_Users.addItem('')
                for userInfo in userInfos:
                    self.cb_Users.addItem(userInfo['name'])

    # selectedUserDate
    def calendarClicked(self):
        self.setReservationTime()
        self.setLog("선택 한 날짜 - [" + self.cal_UsedDate.selectedDate().toString('yyyy.MM.dd') + "]")

    # selectedLoginUser
    def selectedLoginUser(self):
        if self.cb_Users.currentText() != '':
            self.setLog("선택 한 사용자 - [" + self.cb_Users.currentText() + "]")

            # User객체에 User 추가
            with open('gajwaLoginInfos.txt', 'r', encoding='UTF8') as file:
                file_content = file.read()

            userInfoList = json.loads(file_content)
            for userInfo in userInfoList:
                if self.cb_Users.currentText() == userInfo['name']:
                    self.selectedLoginUserInfo = userInfo

    # startMacroValidation
    def startMacroValidation(self):
        # 대표자
        teamName = self.cb_Users.currentText()
        # 이용 날짜
        usedData = self.cal_UsedDate.selectedDate().toString('yyyyMMdd')
        # 시작 할 코트
        courtCount = self.lw_2.count()
        # 시작 시간
        startTime = self.cb_StartTime.currentText()
        # 종료 시간
        endTime = self.cb_EndTime.currentText()

        res = True
        if teamName == '':
            self.setLog('USER 미선택')
            res = False

        if usedData == '':
            self.setLog('날짜 미선택')
            res = False

        if courtCount == 0:
            self.setLog('시작 할 코트 미선택')
            res = False

        if startTime == '':
            self.setLog('시작 시간 미선택')
            res = False

        if endTime == '':
            self.setLog('종료 시간 미선택')
            res = False

        if startTime != '' and endTime != '':
            iStartTime = int(startTime.split(':')[0])
            iEndTime = int(endTime.split(':')[0])

            if iEndTime - iStartTime > 4:
                self.setLog('해당 시설을 예약 하기 위한 최대 시간을 초과하였습니다.')
                res = False

        if not res:
            QMessageBox.critical(self, 'Error', '로그를 확인 하세요.')

        return res

    # 매크로 시작
    @pyqtSlot()
    def startMacroAction(self):
        if self.startMacroValidation():
            self.btn_Start.setEnabled(False)
            self.btn_Stop.setEnabled(True)

            self.btn_Add.setEnabled(False)
            self.btn_Remove.setEnabled(False)

            self.cb_StartTime.setEnabled(False)
            self.cb_EndTime.setEnabled(False)

            self.cal_UsedDate.setEnabled(False)

            self.groupBox.setVisible(False)

            self.setLog('매크로 시작')
            self.isStart = True

            self.startReservation()

    @pyqtSlot()
    def startReservation(self):

        with open('macroDatasGajwa.csv', "r", encoding='utf-8-sig') as f:
            reader = csv.reader(f)
            macroInfoList = list(reader)
            self.macroInfoList = macroInfoList

        try:
            t1 = threading.Thread(target=Reservation, args=(self,))
            t1.daemon = True
            t1.start()
        except Exception as e:
            pass


    # 매크로 중단
    @pyqtSlot()
    def stopMacroAction(self):
        self.btn_Stop.setEnabled(False)
        self.btn_Start.setEnabled(True)

        self.btn_Add.setEnabled(True)
        self.btn_Remove.setEnabled(True)

        self.cb_StartTime.setEnabled(True)
        self.cb_EndTime.setEnabled(True)

        self.cal_UsedDate.setEnabled(True)

        self.groupBox.setVisible(True)

        self.setLog('매크로 중단')
        self.isStart = False

    # set Log
    @pyqtSlot()
    def setLog(self, msg):
        currentTime = QTime.currentTime().toString()
        self.tb_Log.append(currentTime + ' : ' + msg)

        verScrollBar = self.tb_Log.verticalScrollBar()
        scrollIsAtEnd = verScrollBar.maximum() - verScrollBar.value() <= 10

        if not scrollIsAtEnd:
            verScrollBar.setValue(verScrollBar.maximum())  # Scrolls to the bottom
            # horScrollBar.setValue(0)  # scroll to the left


    # ==========================================================================


if __name__ == "__main__":
    app = QApplication(sys.argv)
    myWindow = WindowClass()
    myWindow.show()
    app.exec_()
