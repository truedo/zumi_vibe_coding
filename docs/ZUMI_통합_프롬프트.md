# ZUMI AI 로봇 통합 웹앱 만들기 (연결·표현·이동·AI비전 통합)

## 시스템 구조
브라우저(HTML) ↔ WebSocket(ws://{IP}/ws) ↔ 주미 AI(하드웨어)
서버 없음. HTML 파일 하나로만 동작.
AI 처리도 브라우저에서 직접 수행 (CDN 라이브러리 사용)

## WebSocket 연결 방법
ws = new WebSocket('ws://' + ip + '/ws')
ws.binaryType = 'arraybuffer'
연결 성공(onopen) 시 반드시 아래 두 명령을 순서대로 전송:
1. ws.send('stream') → 영상 스트리밍 시작
2. ws.send('sensor') → 센서 데이터 수신 시작

## 영상 및 센서 수신 방법
ws.onmessage = (event) => {
  if (typeof event.data === 'string') return;
  const arr = new Uint8Array(event.data);

  // 센서 데이터 (10바이트, 헤더 $R)
  if (arr.length === 10 && arr[0] === 0x24 && arr[1] === 0x52) {
    parseAndUpdateSensors(arr);
    return;
  }

  // 영상 데이터 (그 외 = JPEG)
  const blob = new Blob([arr], {type: 'image/jpeg'});
  const url = URL.createObjectURL(blob);
  const img = new Image();
  img.onload = () => { ctx.drawImage(img, 0, 0, w, h); URL.revokeObjectURL(url); };
  img.src = url;
}

## 패킷 빌더
function buildPacket(cmd, ...params) {
  const buf = new Uint8Array(4 + params.length);
  buf[0]=0x24; buf[1]=0x52; buf[2]=cmd; buf[3]=0x00;
  params.forEach((v,i) => buf[4+i]=v);
  return buf.buffer;
}

## 센서 데이터 파싱 규칙 (10바이트)
arr[0]: 0x24 ($)
arr[1]: 0x52 (R)
arr[2]: FR (전방 우측, 0~255) — 가까울수록 작아짐, 감지 기준값: 100
arr[3]: FL (전방 좌측, 0~255) — 가까울수록 작아짐, 감지 기준값: 100
arr[4]: BR (바닥 우측, 0~255) — 흰색일수록 작아짐, 감지 기준값: 30
arr[5]: BL (바닥 좌측, 0~255) — 흰색일수록 작아짐, 감지 기준값: 30
arr[6]: BC (바닥 중앙, 0~255) — 흰색일수록 작아짐, 감지 기준값: 30
arr[7]: BTN (버튼 비트마스크: 8=🔴빨강 4=🔵파랑 2=🟢초록 1=🟡노랑 / 안눌리면 '없음')
        버튼 배열 순서: 빨강-파랑-초록-노랑 (수평)
arr[8]: BAT (배터리 잔량, 최대 100 제한 필요)
arr[9]: STAT (명령 수행 상태: 1=수행중, 0=대기중)
        순차 이동 시 STAT이 0인 것을 확인한 후 다음 명령을 전송해야 함
센서값은 주미마다 개체차 있음

---

## [표현 기능] 명령어

// LED (r, g, b 각 0~10)
LED 색상: CMD=10, params=[r, g, b]
LED 패턴: CMD=201, params=[pattern, time_high, time_low]
  패턴: 0=유지 1=깜빡 2=두번깜빡 3=밝아졌다어두워짐 4=점점어두워짐 5=점점밝아짐 6=무지개
  time: ms 단위, 예) 1000ms → time_high=3, time_low=232

// 사운드
사운드: CMD=242, params=[sound]
  0=고양이 1=셔터 2=실패1 3=실패2 4=경적1 5=경적2 6=사이렌 7=성공

// LCD 텍스트
텍스트 출력: CMD=230, 이후 UTF-8 문자열 바이트 + 0x00
텍스트 추가: CMD=232, 이후 UTF-8 문자열 바이트 + 0x00
텍스트 설정: CMD=231, params=[color, size, 0, 0, 0]
  color(기본값 1): 0=변경안함 1=흰색 2=검정 3=남색 4=파랑 5=하늘색 6=청록색 7=틸 8=초록
                  9=연두 10=라임 11=노랑 12=호박색 13=주황 14=진주황 15=갈색 16=청회색 17=회색
  size(기본값 5): 0=변경안함 1~5

## 텍스트 스마트 분할 전송
하드웨어 버퍼 제한: 한 번에 최대 11바이트(payload)
11바이트 초과 시 자동 슬라이싱, 각 조각 사이 100ms 딜레이 필수

규칙:
- 첫 번째 조각: CMD=230 (새로 출력)
- 이후 조각들: CMD=232 (추가 출력)
- 조각 간 딜레이: 100ms (setTimeout 재귀 루프)

## 기본 sendText 참고
function sendText(text) {
  const encoded = new TextEncoder().encode(text);
  const buf = new Uint8Array(4 + encoded.length + 1);
  buf[0]=0x24; buf[1]=0x52; buf[2]=230; buf[3]=0x00;
  buf.set(encoded, 4);
  buf[4 + encoded.length] = 0x00;
  ws.send(buf.buffer);
}
// 위 함수를 베이스로 11바이트 분할 + 100ms 딜레이 방식으로 고도화할 것

---

## [이동 기능] 명령어

## STAT 적용 규칙
STAT 체크 필요 (완료 후 다음 명령 전송):
- 정밀 이동: CMD=26, 70, 50, 51, 52, 53
- 특수 이동: CMD=101, 28, 100

STAT 체크 불필요 (즉시 전송):
- 버튼/키보드 조종: CMD=32 (CMD_MOTOR_INFINITE)
- 라인트레이서 무한: CMD=30
- 정지: CMD=25

## 전송 딜레이
범위: 10~200ms
sendRaw() 내부에서 if (Date.now() - lastCmdTime < cmdDelay) return;
CMD_STOP(25)은 딜레이 무시하고 즉시 전송

## 모터 제어 (CMD_MOTOR_INFINITE)
버튼/키보드 조종에 적합
CMD_STOP = 25 → buildPacket(25)
CMD_MOTOR_INFINITE = 32
## 모터 패킷 생성 함수
function makeMotorPacket(dirL, speedL, dirR, speedR) {
  speedL = Math.max(0, Math.min(250, speedL));
  speedR = Math.max(0, Math.min(250, speedR));
  dirL = Math.max(0, Math.min(2, dirL));
  dirR = Math.max(0, Math.min(2, dirR));
  const dir = 0x40 | dirL | (dirR << 4);
  return buildPacket(32, speedL, speedR, dir);
}

## 모터 제어(CMD_MOTOR_INFINITE) 캘리브레이션
버튼이나 키보드 조종을 원하는 경우에 함께 적용
변수 4개 (기본값 40, 범위 0~250): calGoL, calGoR, calTurnL, calTurnR
전진: makeMotorPacket(2, calGoL, 1, calGoR)
후진: makeMotorPacket(1, calGoL, 2, calGoR)
좌회전: makeMotorPacket(1, calTurnL, 1, calTurnR)
우회전: makeMotorPacket(2, calTurnL, 2, calTurnR)

## 입력 이벤트
버튼: mousedown/touchstart → 이동, mouseup/mouseleave/touchend → 정지
키보드: keydown(repeat 무시, input 포커스 시 무시) → 이동, keyup → 정지
키 매핑: w=전진 s=후진 a=좌회전 d=우회전 e=정지

## 정밀 이동
전진 거리: CMD=26 params=[speed, dist, 0]
후진 거리: CMD=26 params=[speed, dist, 1]
좌회전 각도: CMD=70 params=[speed, deg%256, Math.floor(deg/256), 0]
우회전 각도: CMD=70 params=[speed, deg%256, Math.floor(deg/256), 1]
빠른 전진: CMD=50 params=[dist] (단위: cm, 0~300)
빠른 후진: CMD=51 params=[dist] (단위: cm, 0~300)
빠른 좌회전: CMD=52 params=[Math.round(deg/5)] (5도 단위, 최대 360도)
빠른 우회전: CMD=53 params=[Math.round(deg/5)] (5도 단위, 최대 360도)

## 특수 이동
라인트레이서 무한: CMD=30 params=[speed]
라인트레이서 시간: CMD=101 params=[speed, senBL, senBR, senBC, time×10] 기본 센서값=100
라인트레이서 거리: CMD=28 params=[speed, dist]
장애물 감지 정지: CMD=100 params=[speed, senL, senR] 기본 센서값=150

---

## [AI 비전 기능]

## AI 오버레이 구조
videoCanvas (영상) + aiCanvas (AI 결과 오버레이, position:absolute)
aiCanvas는 클릭 이벤트 투과: pointer-events: none

        <span class="pk">## 사용 가능한 브라우저 AI 라이브러리 (CDN)</span>
        <span class="pc">// MediaPipe (얼굴 / 손)</span>
        &lt;script src="https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/face_mesh.js"&gt;&lt;/script&gt;
        &lt;script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"&gt;&lt;/script&gt;
        &lt;script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js"&gt;&lt;/script&gt;

        <span class="pc">// TensorFlow.js + COCO-SSD (물체/표지판)</span>
        &lt;script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.17.0/dist/tf.min.js"&gt;&lt;/script&gt;
        &lt;script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/coco-ssd@2.2.3/dist/coco-ssd.min.js"&gt;&lt;/script&gt;

        <span class="pc">// js-aruco (마커 인식)</span>
        &lt;script src="https://cdn.jsdelivr.net/npm/js-aruco2/src/cv.js"&gt;&lt;/script&gt;
        &lt;script src="https://cdn.jsdelivr.net/npm/js-aruco2/src/aruco.js"&gt;&lt;/script&gt;

        <span class="pc">// qr 인식</span>
        &lt;script src="https://cdn.jsdelivr.net/npm/jsqr@1.4.0/dist/jsQR.js"&gt;&lt;/script&gt;

        <span class="pc">// 글자 인식</span>
        &lt;script src="https://cdn.jsdelivr.net/npm/tesseract.js@5/dist/tesseract.min.js"&gt;&lt;/script&gt;

        <span class="pc">// Teachable Machine(이미지 분류)</span>
        &lt;script
        src="https://cdn.jsdelivr.net/npm/@teachablemachine/image@0.8/dist/teachablemachine-image.min.js"&gt;&lt;/script&gt;
        // 모델 URL은 사용자가 직접 입력하게 UI 제공

## AI 루프 패턴 (120~200ms 간격)
// 루프 밖에서 한 번만 생성
const offscreen = document.createElement('canvas');
offscreen.width = 640; offscreen.height = 480;

async function aiLoop() {
  aiCtx.clearRect(0, 0, aiCanvas.width, aiCanvas.height);
  offscreen.getContext('2d').drawImage(videoCanvas, 0, 0);
  if (faceOn) await faceMesh.send({image: offscreen});
  if (handsOn) await hands.send({image: offscreen});
  if (cocoOn) await runCoco(offscreen);
  setTimeout(aiLoop, 150);
}

## 손 랜드마크(21점) 기반 제스처 판별
// 손가락 끝(4,8,12,16,20)과 관절(3,6,10,14,18) y좌표 비교 방식 사용
function recognizeGesture(landmarks) { ... }

## AI 토글 버튼 구조
각 AI 기능(얼굴/손/물체 등)마다 토글 버튼 제공
버튼 ON → 해당 AI 초기화 및 루프 시작
버튼 OFF → 오버레이 지우기, 루프에서 제외

---

## 기본 UI 포함 요소
- IP 입력창 + 연결 버튼 + 해제 버튼
- 연결 상태 표시 (연결됨 / 연결 안 됨)
- 영상 표시 canvas (width=640, height=480)
- 방향 제어 버튼 또는 키보드
- 캘리브레이션 입력 4개
- 전송 딜레이 조절
- AI 라이브러리 로딩 상태 표시
  (용량이 큰 라이브러리 초기화 중에는 "로딩 중..." 등의 메시지 또는 스피너를 표시)

## 결과물 조건
- HTML 파일 하나로 동작
- CSS/JavaScript 내부 포함
- Chrome에서 바로 실행 가능
- 별도 서버 불필요

---

## 추가로 만들어줄 기능:
  (여기에 원하는 것을 직접 적어주세요)

  예시 (연결/보기):
  - 전체화면 버튼 추가 / 사진 저장 기능 / 흑백 필터 버튼 / 화면에 현재 시간 표시 / 밝기·대비 슬라이더

  예시 (표현):
  - 센서 값을 시각적 요소와 숫자로 표현 / 눌린 버튼 값 표현
  - 감정 버튼 (행복/슬픔/화남) — 각각 LED + 사운드 동시 제어
  - LED RGB 슬라이더 / 경고 모드 버튼(빨간 LED 깜빡 + 사이렌) / LCD 텍스트 입력창 / 사운드 버튼 모음

  예시 (이동):
  - WASD 키보드 + D-PAD 버튼 조종 / 속도 슬라이더(1~3) / 거리·각도 슬라이더로 정밀 이동
  - 라인트레이서 시작/정지 버튼 / 정사각형 자동 이동 버튼

  예시 (AI 비전):
  - 얼굴 감지 토글 버튼 + 랜드마크 오버레이
  - 손 제스처로 전진/정지/좌우회전 제어
  - 물체 인식 + 정지 표지판 감지 시 자동 정지
  - 얼굴이 왼쪽이면 좌회전, 오른쪽이면 우회전 (얼굴 추적)
