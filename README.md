from flask import Flask, request, render_template_string
import sqlite3
from datetime import datetime

app = Flask(__name__)
DB = "uc_shop.db"

# اطلاعات آزمایشی؛ بعداً تغییر بده
CARD_NUMBER = "0000-0000-0000-0000"
CARD_OWNER = "نام صاحب کارت"

PACKAGES = [
    ("60", "۶۰ یوسی", "قیمت بسته"),
    ("325", "۳۲۵ یوسی", "قیمت بسته"),
    ("660", "۶۶۰ یوسی", "قیمت بسته"),
    ("1800", "۱۸۰۰ یوسی", "قیمت بسته"),
]

def init_db():
    with sqlite3.connect(DB) as con:
        con.execute("""
            CREATE TABLE IF NOT EXISTS orders (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                player_id TEXT NOT NULL,
                package TEXT NOT NULL,
                contact TEXT,
                status TEXT NOT NULL,
                created_at TEXT NOT NULL
            )
        """)

HTML = """
<!DOCTYPE html>
<html lang="fa" dir="rtl">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>فروشگاه یوسی</title>
<style>
*{box-sizing:border-box}
body{
    margin:0;
    background:#0b1020;
    color:white;
    font-family:Tahoma,Arial;
}
header{
    background:linear-gradient(135deg,#172554,#6d28d9);
    padding:30px 15px;
    text-align:center;
}
header h1{margin:0 0 10px}
main{max-width:750px;margin:25px auto;padding:12px}
.box{
    background:#171f33;
    padding:20px;
    border-radius:15px;
    margin-bottom:18px;
    border:1px solid #303b59;
}
.packages{
    display:grid;
    grid-template-columns:repeat(2,1fr);
    gap:12px;
}
.package{
    background:#222d47;
    padding:15px;
    border-radius:12px;
    text-align:center;
}
label{display:block;margin:15px 0 7px}
input,select{
    width:100%;
    padding:13px;
    border-radius:9px;
    border:1px solid #465574;
    background:#0d1528;
    color:white;
    font-size:15px;
}
button{
    width:100%;
    padding:14px;
    margin-top:18px;
    background:#7c3aed;
    color:white;
    border:0;
    border-radius:10px;
    font-size:16px;
    font-weight:bold;
}
.notice{
    background:#302b1b;
    padding:15px;
    border-radius:10px;
    line-height:2;
    overflow-wrap:anywhere;
}
.small{color:#b9c4df;font-size:13px;line-height:2}
</style>
</head>
<body>
<header>
    <h1>فروشگاه یوسی</h1>
    <div>خرید یوسی پابجی موبایل</div>
</header>

<main>
<div class="box">
    <h2>بسته‌های یوسی</h2>
    <div class="packages">
    {% for p in packages %}
        <div class="package">
            <h3>{{ p[1] }}</h3>
            <div>{{ p[2] }}</div>
        </div>
    {% endfor %}
    </div>
</div>

<div class="box">
    <h2>ثبت سفارش</h2>
    <form method="POST" action="/order">
        <label>آیدی پابجی</label>
        <input name="player_id" required maxlength="40"
               placeholder="آیدی پابجی خود را وارد کنید">

        <label>انتخاب بسته</label>
        <select name="package" required>
        {% for p in packages %}
            <option value="{{ p[0] }}">{{ p[1] }} - {{ p[2] }}</option>
        {% endfor %}
        </select>

        <label>آیدی ارتباطی شما (اختیاری)</label>
        <input name="contact" maxlength="80"
               placeholder="مثلاً آیدی روبیکا">

        <button type="submit">ثبت سفارش</button>
    </form>
</div>

{% if order_id %}
<div class="box">
    <h2>سفارش ثبت شد</h2>
    <p>شماره سفارش: #{{ order_id }}</p>

    <div class="notice">
        <b>اطلاعات کارت</b><br>
        شماره کارت: {{ card_number }}<br>
        به نام: {{ card_owner }}
    </div>

    <p class="small">
        این نسخه آزمایشی است. تا زمان تنظیم قیمت و اطلاعات واقعی،
        پولی واریز نکنید. صاحب فروشگاه باید پرداخت را بررسی کند
        و یوسی را جداگانه ارسال کند.
    </p>
</div>
{% endif %}

</main>
</body>
</html>
"""

@app.route("/")
def home():
    return render_template_string(
        HTML,
        packages=PACKAGES,
        order_id=None,
        card_number=CARD_NUMBER,
        card_owner=CARD_OWNER
    )

@app.route("/order", methods=["POST"])
def order():
    player_id = request.form.get("player_id", "").strip()
    package_id = request.form.get("package", "").strip()
    contact = request.form.get("contact", "").strip()

    package = next(
        (p for p in PACKAGES if p[0] == package_id),
        None
    )

    if not player_id or not package:
        return "اطلاعات سفارش معتبر نیست.", 400

    init_db()

    with sqlite3.connect(DB) as con:
        cur = con.execute("""
            INSERT INTO orders
            (player_id, package, contact, status, created_at)
            VALUES (?, ?, ?, ?, ?)
        """, (
            player_id,
            package[1],
            contact,
            "در انتظار بررسی",
            datetime.now().strftime("%Y-%m-%d %H:%M")
        ))
        order_id = cur.lastrowid

    return render_template_string(
        HTML,
        packages=PACKAGES,
        order_id=order_id,
        card_number=CARD_NUMBER,
        card_owner=CARD_OWNER
    )

if __name__ == "__main__":
    init_db()
    app.run(host="127.0.0.1", port=5000, debug=False)
