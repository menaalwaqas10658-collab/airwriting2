
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>AirWriting AI Studio</title>

<style>
* {
    box-sizing: border-box;
}

body {
    margin: 0;
    background: #080812;
    color: white;
    font-family: Arial, sans-serif;
    text-align: center;
}

header {
    padding: 24px 10px;
    background: linear-gradient(135deg, #190c35, #30105c);
    border-bottom: 2px solid #9333ea;
    box-shadow: 0 5px 25px #9333ea44;
}

h1 {
    color: #c084fc;
    font-size: 34px;
    text-shadow: 0 0 15px #a855f7;
    margin: 5px;
}

.subtitle {
    color: #d8b4fe;
}

.panel {
    background: #151326;
    border: 1px solid #663399;
    box-shadow: 0 0 20px #9333ea22;
    padding: 20px;
    margin: 18px auto;
    border-radius: 18px;
    max-width: 1050px;
    width: 95%;
}

h2 {
    color: #c084fc;
}

button {
    background: linear-gradient(135deg, #7e22ce, #a855f7);
    color: white;
    padding: 12px 18px;
    border: 1px solid #c084fc;
    border-radius: 9px;
    margin: 6px;
    font-size: 14px;
    font-weight: bold;
    cursor: pointer;
}

button:hover {
    background: #c084fc;
    color: #160d29;
    box-shadow: 0 0 15px #a855f7;
}

input, select {
    background: #211a38;
    color: white;
    border: 1px solid #9333ea;
    border-radius: 8px;
    padding: 12px;
    margin: 6px;
}

#canvas {
    background: white;
    width: 100%;
    max-width: 1000px;
    height: 500px;
    border: 2px solid #a855f7;
    border-radius: 12px;
}

#video {
    width: 100%;
    max-width: 360px;
    border: 3px solid #a855f7;
    border-radius: 15px;
    transform: scaleX(-1);
    background: black;
}

.status {
    color: #d8b4fe;
    font-weight: bold;
    margin: 15px;
}

#clear { background: #be185d; }
#save { background: #15803d; }

#phonePanel, #desktopPanel {
    display: none;
}

small {
    color: #a78bfa;
}

.room {
    font-size: 25px;
    font-weight: bold;
    color: #e9d5ff;
}

</style>
</head>

<body>

<header>
    <h1>✦ AIRWRITING AI</h1>
    <p class="subtitle">AI HAND TRACKING STUDIO</p>
</header>

<!-- HOME -->
<div class="panel" id="menu">

    <h2>Welcome to AirWriting</h2>

    <p>Choose your device mode</p>

    <button onclick="showDesktop()">
        💻 DESKTOP MODE
    </button>

    <button onclick="showPhone()">
        📱 PHONE CAMERA MODE
    </button>

    <p><small>
        Connect your Android camera to your desktop
    </small></p>

</div>

<!-- DESKTOP MODE -->
<div class="panel" id="desktopPanel">

    <h2>💻 Desktop Writing Studio</h2>

    <p>Enter your room code</p>

    <input id="desktopRoom"
           maxlength="20"
           value="air1234">

    <br>

    <button onclick="startDesktop()">
        CREATE ROOM
    </button>

    <div id="desktopStatus" class="status">
        Waiting for connection...
    </div>

    <p>Room Code:</p>

    <div class="room" id="roomDisplay">
        air1234
    </div>

    <hr>

    <label>Pen Color</label>
    <input type="color" id="penColor" value="#0000ff">

    <label>Pen Size</label>

    <select id="penSize">
        <option value="3">Small</option>
        <option value="6" selected>Medium</option>
        <option value="12">Large</option>
        <option value="20">Extra Large</option>
    </select>

    <br>

    <button onclick="setEraser(false)">PEN</button>
    <button onclick="setEraser(true)">ERASER</button>
    <button id="clear" onclick="clearCanvas()">CLEAR ALL</button>
    <button id="save" onclick="saveDrawing()">SAVE PNG</button>

    <br><br>

    <canvas id="canvas"></canvas>

</div>

<!-- PHONE MODE -->
<div class="panel" id="phonePanel">

    <h2>📱 Phone Camera Studio</h2>

    <p>Enter the same room code as your desktop</p>

    <input id="phoneRoom"
           maxlength="20"
           value="air1234">

    <br>

    <button onclick="connectPhone()">
        CONNECT TO DESKTOP
    </button>

    <p id="phoneStatus" class="status">
        Not connected
    </p>

    <button onclick="startCamera()">
        START CAMERA
    </button>

    <br><br>

    <video id="video"
           autoplay
           playsinline
           muted></video>

    <p>
        <small>
            Show your hand to the camera.<br>
            Pinch thumb and index finger to draw.<br>
            Separate fingers to stop drawing.
        </small>
    </p>

</div>

<script src="https://unpkg.com/peerjs@1.5.5/dist/peerjs.min.js"></script>

<script type="module">

import {
    HandLandmarker,
    FilesetResolver
} from "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.14";

const $ = id => document.getElementById(id);

let peer = null;
let connection = null;
let handTracker = null;
let cameraStream = null;

let canvas = null;
let ctx = null;

let lastPoint = null;
let erasing = false;
let tracking = false;
let lastVideoTime = -1;

// --------------------------------
// DEVICE MODES
// --------------------------------

window.showDesktop = function() {
    $("menu").style.display = "none";
    $("desktopPanel").style.display = "block";
    $("phonePanel").style.display = "none";

    setupCanvas();
};

window.showPhone = function() {
    $("menu").style.display = "none";
    $("desktopPanel").style.display = "none";
    $("phonePanel").style.display = "block";
};

// --------------------------------
// CANVAS SETUP
// --------------------------------

function setupCanvas() {

    canvas = $("canvas");
    ctx = canvas.getContext("2d");

    const rect = canvas.getBoundingClientRect();
    const dpr = window.devicePixelRatio || 1;

    canvas.width = rect.width * dpr;
    canvas.height = rect.height * dpr;

    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);

    ctx.lineCap = "round";
    ctx.lineJoin = "round";
}

// --------------------------------
// DRAWING CONTROLS
// --------------------------------

window.setEraser = function(value) {
    erasing = value;

    $("desktopStatus").textContent =
        erasing ? "Eraser selected" : "Pen selected";
};

window.clearCanvas = function() {

    ctx.clearRect(
        0,
        0,
        canvas.width,
        canvas.height
    );

    lastPoint = null;
};

window.saveDrawing = function() {

    const link = document.createElement("a");

    link.download = "AirWriting.png";

    link.href = canvas.toDataURL("image/png");

    link.click();
};

// --------------------------------
// DRAW RECEIVED FINGER MOVEMENT
// --------------------------------

function drawOnDesktop(data) {

    if (!data.drawing) {
        lastPoint = null;
        return;
    }

    const x = data.x * canvas.clientWidth;
    const y = data.y * canvas.clientHeight;

    if (!lastPoint) {

        lastPoint = {x, y};

        return;
    }

    ctx.beginPath();

    ctx.moveTo(lastPoint.x, lastPoint.y);

    ctx.lineTo(x, y);

    ctx.strokeStyle = erasing
        ? "white"
        : $("penColor").value;

    ctx.lineWidth = Number($("penSize").value);

    ctx.stroke();

    lastPoint = {x, y};
}

// --------------------------------
// DESKTOP PEER CONNECTION
// --------------------------------

window.startDesktop = function() {

    const room = $("desktopRoom").value
        .trim()
        .toLowerCase()
        .replace(/[^a-z0-9_-]/g, "");

    if (!room) {
        alert("Enter a room code");
        return;
    }

    if (peer) {
        peer.destroy();
    }

    $("roomDisplay").textContent = room;

    $("desktopStatus").textContent =
        "Creating room...";

    peer = new Peer("airwriting-" + room);

    peer.on("open", () => {

        $("desktopStatus").textContent =
            "Room created! Waiting for phone...";

    });

    peer.on("connection", conn => {

        connection = conn;

        conn.on("open", () => {

            $("desktopStatus").textContent =
                "PHONE CONNECTED! Start air writing.";

        });

        conn.on("data", data => {

            if (data.type === "draw") {
                drawOnDesktop(data);
            }

        });

        conn.on("close", () => {

            lastPoint = null;

            $("desktopStatus").textContent =
                "Phone disconnected";

        });

    });

    peer.on("error", error => {

        $("desktopStatus").textContent =
            "Connection error: " + error.type;

    });
};

// --------------------------------
// PHONE CONNECTION
// --------------------------------

window.connectPhone = function() {

    const room = $("phoneRoom").value
        .trim()
        .toLowerCase()
        .replace(/[^a-z0-9_-]/g, "");

    if (!room) {
        alert("Enter the room code");
        return;
    }

    if (peer) {
        peer.destroy();
    }

    $("phoneStatus").textContent =
        "Connecting to desktop...";

    peer = new Peer();

    peer.on("open", () => {

        connection = peer.connect(
            "airwriting-" + room
        );

        connection.on("open", () => {

            $("phoneStatus").textContent =
                "CONNECTED TO DESKTOP!";

        });

        connection.on("error", error => {

            $("phoneStatus").textContent =
                "Connection error";

        });

        connection.on("close", () => {

            $("phoneStatus").textContent =
                "Disconnected";

        });

    });

    peer.on("error", error => {

        $("phoneStatus").textContent =
            "Connection error: " + error.type;

    });
};

// --------------------------------
// START PHONE CAMERA
// --------------------------------

window.startCamera = async function() {

    try {

        $("phoneStatus").textContent =
            "Requesting camera permission...";

        cameraStream =
            await navigator.mediaDevices.getUserMedia({

                audio: false,

                video: {
                    facingMode: "environment",
                    width: 640,
                    height: 480
                }

            });

        const video = $("video");

        video.srcObject = cameraStream;

        await video.play();

        $("phoneStatus").textContent =
            "Loading AI hand detection...";

        const vision =
            await FilesetResolver.forVisionTasks(
                "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.14/wasm"
            );

        handTracker =
            await HandLandmarker.createFromOptions(
                vision,
                {
                    baseOptions: {
                        modelAssetPath:
                        "https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task",
                        delegate: "GPU"
                    },

                    runningMode: "VIDEO",
                    numHands: 1
                }
            );

        $("phoneStatus").textContent =
            "AI READY! Show your hand.";

        tracking = true;

        trackHand();

    } catch (error) {

        $("phoneStatus").textContent =
            "ERROR: " + error.message;

    }

};

// --------------------------------
// HAND TRACKING
// --------------------------------

function trackHand() {

    if (!tracking) return;

    const video = $("video");

    if (
        !handTracker ||
        video.readyState < 2
    ) {

        requestAnimationFrame(trackHand);

        return;

    }

    if (video.currentTime !== lastVideoTime) {

        lastVideoTime = video.currentTime;

        const results =
            handTracker.detectForVideo(
                video,
                performance.now()
            );

        if (results.landmarks.length > 0) {

            const hand = results.landmarks[0];

            // Index fingertip
            const index = hand[8];

            // Thumb fingertip
            const thumb = hand[4];

            // Calculate pinch distance
            const dx = index.x - thumb.x;
            const dy = index.y - thumb.y;

            const distance =
                Math.sqrt(dx * dx + dy * dy);

            // Pinch to draw
            const drawing = distance < 0.055;

            // Mirrored camera position
            const x = 1 - index.x;
            const y = index.y;

            $("phoneStatus").textContent =
                drawing
                ? "✍ DRAWING"
                : "Finger detected - Pinch to draw";

            if (connection && connection.open) {

                connection.send({

                    type: "draw",

                    x: x,
                    y: y,

                    drawing: drawing

                });

            }

        } else {

            $("phoneStatus").textContent =
                "Hand not detected - Show your hand";

            if (connection && connection.open) {

                connection.send({
                    type: "draw",
                    drawing: false
                });

            }

        }

    }

    requestAnimationFrame(trackHand);

}

// --------------------------------
// CLEANUP
// --------------------------------

window.addEventListener("beforeunload", () => {

    tracking = false;

    if (cameraStream) {

        cameraStream.getTracks().forEach(
            track => track.stop()
        );

    }

    if (peer) {
        peer.destroy();
    }

});

</script>

</body>
</html>
