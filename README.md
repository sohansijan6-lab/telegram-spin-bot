import os
import time
import hmac
import hashlib
import json
import secrets
import sqlite3
import threading
from urllib.parse import parse_qsl

import requests
from flask import Flask, request, jsonify, Response


# =========================================================
# SETTINGS
# =========================================================

BOT_TOKEN = os.getenv("BOT_TOKEN", "YOUR_BOT_TOKEN")
WEBAPP_URL = os.getenv("WEBAPP_URL", "https://YOUR-DOMAIN.com")

PORT = int(os.getenv("PORT", "8080"))
DATABASE = "spinbot.db"

PRIZES = [1, 2, 3, 4, 5]

# 24 hours between spins
SPIN_COOLDOWN = 24 * 60 * 60

db_lock = threading.Lock()

app = Flask(__name__)


# =========================================================
# DATABASE
# =========================================================

def get_db():
    conn = sqlite3.connect(
        DATABASE,
        check_same_thread=False
    )
    conn.row_factory = sqlite3.Row
    return conn


def init_db():
    with db_lock:
        conn = get_db()

        conn.execute("""
            CREATE TABLE IF NOT EXISTS users (
                user_id INTEGER PRIMARY KEY,
                first_name TEXT DEFAULT '',
                username TEXT DEFAULT '',
                balance REAL DEFAULT 0,
                spins INTEGER DEFAULT 0,
                last_spin INTEGER DEFAULT 0,
                created_at INTEGER DEFAULT 0
            )
        """)

        conn.commit()
        conn.close()


def create_user(user):
    user_id = int(user["id"])

    first_name = user.get(
        "first_name",
        ""
    )

    username = user.get(
        "username",
        ""
    )

    with db_lock:
        conn = get_db()

        exists = conn.execute(
            "SELECT user_id FROM users WHERE user_id = ?",
            (user_id,)
        ).fetchone()

        if exists:

            conn.execute("""
                UPDATE users
                SET first_name = ?,
                    username = ?
                WHERE user_id = ?
            """, (
                first_name,
                username,
                user_id
            ))

        else:

            conn.execute("""
                INSERT INTO users
                (
                    user_id,
                    first_name,
                    username,
                    balance,
                    spins,
                    last_spin,
                    created_at
                )
                VALUES (?, ?, ?, 0, 0, 0, ?)
            """, (
                user_id,
                first_name,
                username,
                int(time.time())
            ))

        conn.commit()
        conn.close()


def get_user(user_id):

    with db_lock:
        conn = get_db()

        row = conn.execute(
            "SELECT * FROM users WHERE user_id = ?",
            (user_id,)
        ).fetchone()

        conn.close()

        return row


# =========================================================
# TELEGRAM API
# =========================================================

def telegram(method, data=None):

    url = (
        "https://api.telegram.org/bot"
        + BOT_TOKEN
        + "/"
        + method
    )

    try:

        response = requests.post(
            url,
            json=data or {},
            timeout=30
        )

        return response.json()

    except Exception as e:

        print("Telegram error:", e)

        return None


def send_message(chat_id, text, keyboard=None):

    data = {
        "chat_id": chat_id,
        "text": text
    }

    if keyboard:
        data["reply_markup"] = keyboard

    return telegram(
        "sendMessage",
        data
    )


# =========================================================
# OPEN WEB APP BUTTON
# =========================================================

def open_keyboard():

    return {
        "inline_keyboard": [
            [
                {
                    "text": "🎡 Open Spin",
                    "web_app": {
                        "url": WEBAPP_URL
                    }
                }
            ]
        ]
    }


# =========================================================
# BOT COMMANDS
# =========================================================

def handle_update(update):

    message = update.get("message")

    if not message:
        return

    user = message.get(
        "from",
        {}
    )

    chat_id = message.get(
        "chat",
        {}
    ).get("id")

    text = message.get(
        "text",
        ""
    )

    if not chat_id:
        return

    if text.startswith("/start"):

        create_user(user)

        name = user.get(
            "first_name",
            "Friend"
        )

        send_message(
            chat_id,

            f"👋 Hello {name}!\n\n"
            "🎡 Welcome to Spin & Win.\n\n"
            "Press the button below to open the Spin Web App.",

            open_keyboard()
        )

    elif text.startswith("/balance"):

        create_user(user)

        row = get_user(
            user["id"]
        )

        send_message(
            chat_id,

            "💰 Balance: $"
            + f"{float(row['balance']):.2f}"
            + "\n🎡 Spins: "
            + str(row["spins"])
        )

    elif text.startswith("/help"):

        send_message(
            chat_id,

            "/start - Open Spin App\n"
            "/balance - Check balance\n"
            "/help - Help"
        )


# =========================================================
# TELEGRAM LONG POLLING
# =========================================================

def bot_loop():

    print("Telegram bot started.")

    offset = 0

    while True:

        try:

            result = telegram(
                "getUpdates",
                {
                    "offset": offset,
                    "timeout": 30
                }
            )

            if not result:
                time.sleep(3)
                continue

            if not result.get("ok"):
                print(result)
                time.sleep(5)
                continue

            for update in result.get(
                "result",
                []
            ):

                offset = (
                    update["update_id"] + 1
                )

                try:
                    handle_update(update)

                except Exception as e:

                    print(
                        "Update error:",
                        e
                    )

        except Exception as e:

            print(
                "Bot loop error:",
                e
            )

            time.sleep(5)


# =========================================================
# TELEGRAM WEB APP SECURITY
# =========================================================

def validate_init_data(init_data):

    if not init_data:
        return None

    try:

        data = dict(
            parse_qsl(
                init_data,
                keep_blank_values=True
            )
        )

        received_hash = data.pop(
            "hash",
            None
        )

        if not received_hash:
            return None

        data_check_string = "\n".join(
            f"{key}={data[key]}"
            for key in sorted(data.keys())
        )

        secret_key = hmac.new(
            b"WebAppData",
            BOT_TOKEN.encode(),
            hashlib.sha256
        ).digest()

        calculated_hash = hmac.new(
            secret_key,
            data_check_string.encode(),
            hashlib.sha256
        ).hexdigest()

        if not hmac.compare_digest(
            calculated_hash,
            received_hash
        ):
            return None

        auth_date = int(
            data.get(
                "auth_date",
                "0"
            )
        )

        if int(time.time()) - auth_date > 86400:
            return None

        user_text = data.get(
            "user"
        )

        if not user_text:
            return None

        return json.loads(
            user_text
        )

    except Exception as e:

        print(
            "Validation error:",
            e
        )

        return None


# =========================================================
# WEB APP
# =========================================================

HTML = """
<!DOCTYPE html>

<html>

<head>

<meta charset="UTF-8">

<meta
 name="viewport"
 content="width=device-width,initial-scale=1.0"
>

<title>Spin & Win</title>

<script src="https://telegram.org/js/telegram-web-app.js"></script>

<style>

* {
    box-sizing: border-box;
}

body {
    margin: 0;
    min-height: 100vh;
    background: var(--tg-theme-bg-color, #101010);
    color: var(--tg-theme-text-color, white);
    font-family: Arial, sans-serif;
    text-align: center;
}

.container {
    max-width: 450px;
    margin: auto;
    padding: 20px;
}

h1 {
    margin-top: 10px;
}

.balance {
    margin: 20px 0;
    padding: 15px;
    border-radius: 15px;
    background: var(
        --tg-theme-secondary-bg-color,
        #202020
    );
    font-size: 22px;
}

.wheel-box {
    position: relative;
    width: 300px;
    height: 300px;
    margin: 25px auto;
}

.pointer {
    position: absolute;
    z-index: 10;
    top: -12px;
    left: 50%;
    transform: translateX(-50%);
    width: 0;
    height: 0;
    border-left: 18px solid transparent;
    border-right: 18px solid transparent;
    border-top: 40px solid red;
}

.wheel {
    width: 300px;
    height: 300px;
    border-radius: 50%;
    border: 7px solid white;

    background:
        conic-gradient(
            #ff4757 0deg 72deg,
            #ffa502 72deg 144deg,
            #ffdd59 144deg 216deg,
            #2ed573 216deg 288deg,
            #3742fa 288deg 360deg
        );

    transition:
        transform 4s cubic-bezier(
            .17,.67,.12,.99
        );

    position: relative;
}

.prize {
    position: absolute;
    width: 100%;
    text-align: center;
    left: 0;
    top: 135px;
    font-size: 25px;
    font-weight: bold;
    color: white;
    text-shadow: 2px 2px 4px black;
}

.p1 { transform: rotate(-36deg) translateY(-105px); }
.p2 { transform: rotate(36deg) translateY(-105px); }
.p3 { transform: rotate(108deg) translateY(-105px); }
.p4 { transform: rotate(180deg) translateY(-105px); }
.p5 { transform: rotate(252deg) translateY(-105px); }

button {
    width: 100%;
    padding: 17px;
    border: 0;
    border-radius: 15px;
    background: var(
        --tg-theme-button-color,
        #2481cc
    );
    color: var(
        --tg-theme-button-text-color,
        white
    );
    font-size: 20px;
    font-weight: bold;
}

button:disabled {
    opacity: .5;
}

.result {
    margin-top: 20px;
    min-height: 35px;
    font-size: 23px;
    font-weight: bold;
}

</style>

</head>

<body>

<div class="container">

<h1>🎡 Spin & Win</h1>

<div class="balance">
💰 Balance:
<span id="balance">$0.00</span>
</div>

<div class="wheel-box">

<div class="pointer"></div>

<div id="wheel" class="wheel">

<div class="prize p1">$1</div>
<div class="prize p2">$2</div>
<div class="prize p3">$3</div>
<div class="prize p4">$4</div>
<div class="prize p5">$5</div>

</div>

</div>

<button
id="spinButton"
onclick="spin()"
>
🎰 SPIN
</button>

<div
id="result"
class="result"
></div>

</div>


<script>

const tg =
    window.Telegram.WebApp;

tg.ready();

tg.expand();


function initData() {

    return tg.initData || "";

}


async function loadBalance() {

    try {

        const response =
            await fetch(
                "/api/me",
                {
                    method: "POST",

                    headers: {
                        "Content-Type":
                            "application/json"
                    },

                    body: JSON.stringify({
                        initData:
                            initData()
                    })
                }
            );

        const data =
            await response.json();

        if (data.ok) {

            document.getElementById(
                "balance"
            ).innerText =
                "$" +
                data.balance.toFixed(2);

        }

    } catch (e) {

        console.log(e);

    }

}


async function spin() {

    const button =
        document.getElementById(
            "spinButton"
        );

    const result =
        document.getElementById(
            "result"
        );

    button.disabled = true;

    result.innerText =
        "🎡 Spinning...";


    try {

        const response =
            await fetch(
                "/api/spin",
                {
                    method: "POST",

                    headers: {
                        "Content-Type":
                            "application/json"
                    },

                    body: JSON.stringify({
                        initData:
                            initData()
                    })
                }
            );


        const data =
            await response.json();


        if (!data.ok) {

            result.innerText =
                "⚠️ " +
                data.error;

            button.disabled =
                false;

            return;
        }


        const section =
            360 / 5;

        const target =
            360 -
            (
                data.index *
                section
            ) -
            (section / 2);

        const rotation =
            360 * 6 +
            target;


        document.getElementById(
            "wheel"
        ).style.transform =
            "rotate(" +
            rotation +
            "deg)";


        setTimeout(
            function() {

                document.getElementById(
                    "balance"
                ).innerText =
                    "$" +
                    data.balance.toFixed(2);

                result.innerText =
                    "🎉 You won $" +
                    data.prize.toFixed(2) +
                    "!";

                button.disabled =
                    false;

            },
            4200
        );


    } catch (e) {

        result.innerText =
            "❌ Network error.";

        button.disabled =
            false;

    }

}


loadBalance();

</script>

</body>

</html>
"""


# =========================================================
# ROUTES
# =========================================================

@app.route("/")
def home():

    return Response(
        HTML,
        mimetype="text/html"
    )


@app.route(
    "/health"
)
def health():

    return jsonify({
        "status": "ok"
    })


@app.route(
    "/api/me",
    methods=["POST"]
)
def api_me():

    body = request.get_json(
        silent=True
    ) or {}

    user = validate_init_data(
        body.get(
            "initData",
            ""
        )
    )

    if not user:

        return jsonify({
            "ok": False,
            "error":
                "Invalid Telegram session."
        }), 401


    create_user(user)

    row = get_user(
        user["id"]
    )


    return jsonify({

        "ok": True,

        "balance":
            float(row["balance"]),

        "spins":
            int(row["spins"])

    })


@app.route(
    "/api/spin",
    methods=["POST"]
)
def api_spin():

    body = request.get_json(
        silent=True
    ) or {}

    user = validate_init_data(
        body.get(
            "initData",
            ""
        )
    )

    if not user:

        return jsonify({
            "ok": False,
            "error":
                "Invalid Telegram session."
        }), 401


    user_id = int(
        user["id"]
    )

    create_user(user)


    with db_lock:

        conn = get_db()

        row = conn.execute(
            """
            SELECT *
            FROM users
            WHERE user_id = ?
            """,
            (user_id,)
        ).fetchone()


        now = int(
            time.time()
        )

        last_spin = int(
            row["last_spin"]
        )


        if last_spin:

            remaining = (
                SPIN_COOLDOWN
                -
                (now - last_spin)
            )

            if remaining > 0:

                hours = (
                    remaining // 3600
                )

                minutes = (
                    (remaining % 3600)
                    // 60
                )

                conn.close()

                return jsonify({

                    "ok": False,

                    "error":
                        "Next spin in "
                        + str(hours)
                        + "h "
                        + str(minutes)
                        + "m"

                }), 429


        index =
            secrets.randbelow(
                len(PRIZES)
            )

        prize =
            PRIZES[index]


        new_balance = (
            float(row["balance"])
            +
            prize
        )

        new_spins = (
            int(row["spins"])
            +
            1
        )


        conn.execute(
            """
            UPDATE users
            SET balance = ?,
                spins = ?,
                last_spin = ?
            WHERE user_id = ?
            """,
            (
                new_balance,
                new_spins,
                now,
                user_id
            )
        )

        conn.commit()
        conn.close()


    return jsonify({

        "ok": True,

        "prize":
            float(prize),

        "index":
            index,

        "balance":
            float(new_balance),

        "spins":
            new_spins

    })


# =========================================================
# START
# =========================================================

if __name__ == "__main__":

    init_db()

    threading.Thread(
        target=bot_loop,
        daemon=True
    ).start()

    app.run(
        host="0.0.0.0",
        port=PORT,
        debug=False
    )
