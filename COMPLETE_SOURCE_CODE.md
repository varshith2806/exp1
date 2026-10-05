# CHATHURVEDA TALENT SCHOOL - COMPLETE SOURCE CODE
> `static/logo.png` is included separately because it is a binary image.

## `app.py`

```python
import os
import sqlite3
from datetime import date, datetime
from functools import wraps
from io import BytesIO
from pathlib import Path

from flask import Flask, flash, g, redirect, render_template, request, send_file, session, url_for
from werkzeug.security import check_password_hash, generate_password_hash

BASE_DIR = Path(__file__).resolve().parent
DB_PATH = BASE_DIR / "school.db"

app = Flask(__name__)
app.config["SECRET_KEY"] = os.getenv("SECRET_KEY", "change-this-secret-key-in-production")
app.config["SCHOOL_NAME"] = "CHATHURVEDA TALENT SCHOOL"
app.config["SCHOOL_LOCATION"] = "SURAMPALLY, TELANGANA"
app.config["SCHOOL_CODE"] = "CTS-SRM"
app.config["PRINCIPAL"] = "J. Shankar"

CLASS_ORDER = ["Nursery", "LKG", "UKG"] + [f"{i}th Class" if i not in (1,2,3) else {1:"1st Class",2:"2nd Class",3:"3rd Class"}[i] for i in range(1,11)]
SECTIONS = ["A", "B", "C", "D"]
ATTENDANCE_STATUSES = ["Present", "Absent", "Leave"]
PAYMENT_METHODS = ["Cash", "UPI", "Card"]

# Keep the original project's religion/caste terminology while moving storage to SQLite.
CASTE_DATA = {
    "Hindu": {
        "OC (Open Category)": ["Reddy", "Kamma", "Kapu", "Brahmin", "Arya Vysya", "Velama", "Kshatriya / Raju", "Rao", "Naidu", "Niyogi", "Deshastha", "Marwari", "Other OC"],
        "BC-A": ["Agnikulakshatriya / Palli / Vadabalija", "Boyer / Valmiki", "Gudala", "Gangaputra / Besta", "Jilakarra", "Kaikala / Sengundhar", "Kintala Kalinga", "Nayi-Brahmin / Mangala", "Rajaka / Chakali", "Vaddera", "Yata", "Other BC-A"],
        "BC-B": ["Dudekula / Pinjari", "Goud / Ediga", "Kummari / Shalivahana", "Kuruba / Kuruma", "Padmasali / Sali", "Perika", "Somavansha Kshatriya", "Swakulasali", "Viswabrahmin / Viswakarma", "Yadava / Golla", "Other BC-B"],
        "BC-C": ["Scheduled Caste Converts to Christianity", "Other BC-C"],
        "BC-D": ["Gawara", "Kandra", "Koppula Velama", "Munnuru Kapu", "Nagavaddilu", "Nayanavar", "Polinati Velama", "Srisayana / Senapathulu", "Surya Balija", "Turpu Kapu", "Vanikula Kshatriya", "Other BC-D"],
        "BC-E": ["Shaik / Sheikh", "Syed", "Pathan", "Attar Saibulu", "Dhobi Muslim", "Labbai", "Other BC-E"],
        "SC (Scheduled Caste)": ["Madiga", "Mala", "Adi Andhra", "Adi Dravida", "Arundhatiya", "Beda Jangam", "Bindla", "Chamar / Mochi", "Dandasi", "Dhor", "Dom / Domban", "Godagali", "Gosangi", "Jaggali", "Jambuvulu", "Mala Jangam", "Mang", "Matangi", "Mehtar", "Panchama", "Relli", "Samagara", "Samban", "Yatala", "Other SC Subcaste"],
        "ST (Scheduled Tribe)": ["Andh", "Bagata", "Bhil", "Chenchu", "Gadaba", "Gond", "Goudu", "Jatapu", "Koya", "Konda Dhora", "Konda Kapu", "Konda Reddi", "Kotia", "Savara", "Sugali / Lambada / Banjara", "Thoti", "Valmiki", "Yenadi", "Yerukula", "Other ST Subcaste"]
    },
    "Islam": {"General": ["Syed", "Sheikh", "Pathan", "Mughal", "Khan"], "BC-E": ["Shaik / Sheikh", "Syed", "Pathan", "Attar Saibulu", "Dhobi Muslim", "Labbai", "Garadi", "Fakir", "Other BC-E"]},
    "Christianity": {"General": ["Christian General"], "BC-C": ["Scheduled Caste Converts to Christianity", "Other BC-C"]},
    "Sikhism": {"General": ["Jat Sikh", "Ramgarhia", "Khatri Sikh", "Arora Sikh", "Other Sikh"]},
    "Buddhism": {"General": ["Neo-Buddhist", "Theravada", "Mahayana", "Other Buddhist"]},
    "Jainism": {"General": ["Digambara", "Svetambara", "Oswal", "Agarwal Jain", "Other Jain"]},
    "Other": {"General": ["General / Other"]}
}


def get_db():
    if "db" not in g:
        g.db = sqlite3.connect(DB_PATH)
        g.db.row_factory = sqlite3.Row
        g.db.execute("PRAGMA foreign_keys = ON")
    return g.db


@app.teardown_appcontext
def close_db(_error=None):
    db = g.pop("db", None)
    if db is not None:
        db.close()


def init_db():
    db = sqlite3.connect(DB_PATH)
    db.execute("PRAGMA foreign_keys = ON")
    db.executescript("""
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT UNIQUE NOT NULL,
        password_hash TEXT NOT NULL,
        role TEXT NOT NULL CHECK(role IN ('admin','user')),
        full_name TEXT NOT NULL,
        active INTEGER NOT NULL DEFAULT 1,
        created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
    );

    CREATE TABLE IF NOT EXISTS students (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        admission_no TEXT UNIQUE NOT NULL,
        name TEXT NOT NULL,
        dob TEXT NOT NULL,
        current_class TEXT NOT NULL,
        from_class TEXT,
        to_class TEXT,
        section TEXT NOT NULL,
        father TEXT,
        mother TEXT,
        phone TEXT,
        religion TEXT,
        caste TEXT,
        subcaste TEXT,
        nationality TEXT DEFAULT 'Indian',
        place_of_birth TEXT,
        admission_date TEXT NOT NULL,
        village TEXT,
        school_fee REAL NOT NULL DEFAULT 0,
        vehicle_required TEXT NOT NULL DEFAULT 'No',
        vehicle_fee REAL NOT NULL DEFAULT 0,
        active INTEGER NOT NULL DEFAULT 1,
        created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
        updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
    );

    CREATE TABLE IF NOT EXISTS payments (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        receipt_no TEXT UNIQUE NOT NULL,
        student_id INTEGER NOT NULL,
        payment_date TEXT NOT NULL,
        method TEXT NOT NULL,
        school_amount REAL NOT NULL DEFAULT 0,
        vehicle_amount REAL NOT NULL DEFAULT 0,
        total_amount REAL NOT NULL DEFAULT 0,
        created_by INTEGER,
        created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
        FOREIGN KEY(student_id) REFERENCES students(id) ON DELETE CASCADE,
        FOREIGN KEY(created_by) REFERENCES users(id) ON DELETE SET NULL
    );

    CREATE TABLE IF NOT EXISTS attendance (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        student_id INTEGER NOT NULL,
        attendance_date TEXT NOT NULL,
        status TEXT NOT NULL CHECK(status IN ('Present','Absent','Leave')),
        remarks TEXT,
        marked_by INTEGER,
        UNIQUE(student_id, attendance_date),
        FOREIGN KEY(student_id) REFERENCES students(id) ON DELETE CASCADE,
        FOREIGN KEY(marked_by) REFERENCES users(id) ON DELETE SET NULL
    );

    CREATE TABLE IF NOT EXISTS notifications (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        student_id INTEGER NOT NULL,
        channel TEXT NOT NULL,
        recipient TEXT NOT NULL,
        message TEXT NOT NULL,
        status TEXT NOT NULL,
        provider_sid TEXT,
        created_by INTEGER,
        created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
        FOREIGN KEY(student_id) REFERENCES students(id) ON DELETE CASCADE,
        FOREIGN KEY(created_by) REFERENCES users(id) ON DELETE SET NULL
    );
    """)
    admin = db.execute("SELECT id FROM users WHERE username='admin'").fetchone()
    if not admin:
        default_password = os.getenv("ADMIN_PASSWORD", "admin123")
        db.execute("INSERT INTO users(username,password_hash,role,full_name) VALUES(?,?,?,?)", ("admin", generate_password_hash(default_password), "admin", "School Administrator"))
    db.commit()
    db.close()


@app.before_request
def ensure_db():
    if not DB_PATH.exists():
        init_db()


@app.context_processor
def inject_globals():
    return {
        "school_name": app.config["SCHOOL_NAME"],
        "school_location": app.config["SCHOOL_LOCATION"],
        "school_code": app.config["SCHOOL_CODE"],
        "principal": app.config["PRINCIPAL"],
        "today": date.today().isoformat(),
        "current_user": current_user()
    }


def current_user():
    uid = session.get("user_id")
    if not uid:
        return None
    return get_db().execute("SELECT * FROM users WHERE id=? AND active=1", (uid,)).fetchone()


def login_required(view):
    @wraps(view)
    def wrapped(*args, **kwargs):
        if not current_user():
            return redirect(url_for("login", next=request.path))
        return view(*args, **kwargs)
    return wrapped


def role_required(*roles):
    def decorator(view):
        @wraps(view)
        def wrapped(*args, **kwargs):
            user = current_user()
            if not user:
                return redirect(url_for("login", next=request.path))
            if user["role"] not in roles:
                flash("You do not have permission to perform that action.", "danger")
                return redirect(url_for("dashboard"))
            return view(*args, **kwargs)
        return wrapped
    return decorator


def clean(value):
    return (value or "").strip()


def money(value):
    return f"₹{float(value or 0):,.2f}"


def parse_amount(value):
    try:
        n = float(value or 0)
        if n < 0:
            raise ValueError
        return round(n, 2)
    except (ValueError, TypeError):
        raise ValueError("Amount must be a valid non-negative number.")


def next_admission_no():
    year = date.today().year
    prefix = f"CTS-{year}-"
    row = get_db().execute("SELECT admission_no FROM students WHERE admission_no LIKE ? ORDER BY id DESC LIMIT 1", (prefix + "%",)).fetchone()
    if not row:
        return prefix + "001"
    try:
        n = int(row["admission_no"].split("-")[-1]) + 1
    except ValueError:
        n = get_db().execute("SELECT COUNT(*) c FROM students").fetchone()["c"] + 1
    return prefix + str(n).zfill(3)


def receipt_no(payment_id=None):
    year = date.today().year
    if payment_id is None:
        row = get_db().execute("SELECT COALESCE(MAX(id),0)+1 n FROM payments").fetchone()
        payment_id = row["n"]
    return f"CTS-{year}-{int(payment_id):05d}"


def student_balances(student_id):
    db = get_db()
    student = db.execute("SELECT school_fee, vehicle_fee FROM students WHERE id=?", (student_id,)).fetchone()
    paid = db.execute("SELECT COALESCE(SUM(school_amount),0) school_paid, COALESCE(SUM(vehicle_amount),0) vehicle_paid FROM payments WHERE student_id=?", (student_id,)).fetchone()
    school_paid = float(paid["school_paid"])
    vehicle_paid = float(paid["vehicle_paid"])
    return {
        "school_paid": school_paid,
        "vehicle_paid": vehicle_paid,
        "school_due": max(float(student["school_fee"]) - school_paid, 0),
        "vehicle_due": max(float(student["vehicle_fee"]) - vehicle_paid, 0),
        "total_paid": school_paid + vehicle_paid,
        "total_due": max(float(student["school_fee"]) - school_paid, 0) + max(float(student["vehicle_fee"]) - vehicle_paid, 0)
    }


def enrich_student(row):
    item = dict(row)
    item.update(student_balances(row["id"]))
    return item


def number_to_words(num):
    n = int(round(float(num or 0)))
    if n == 0:
        return "Zero Rupees Only"
    ones = ["", "One", "Two", "Three", "Four", "Five", "Six", "Seven", "Eight", "Nine", "Ten", "Eleven", "Twelve", "Thirteen", "Fourteen", "Fifteen", "Sixteen", "Seventeen", "Eighteen", "Nineteen"]
    tens = ["", "", "Twenty", "Thirty", "Forty", "Fifty", "Sixty", "Seventy", "Eighty", "Ninety"]
    def two(x):
        if x < 20: return ones[x]
        return tens[x // 10] + (" " + ones[x % 10] if x % 10 else "")
    parts = []
    if n >= 10000000:
        parts += [two(n // 10000000), "Crore"]
        n %= 10000000
    if n >= 100000:
        parts += [two(n // 100000), "Lakh"]
        n %= 100000
    if n >= 1000:
        parts += [two(n // 1000), "Thousand"]
        n %= 1000
    if n >= 100:
        parts += [ones[n // 100], "Hundred"]
        n %= 100
    if n:
        if parts: parts.append("and")
        parts.append(two(n))
    return " ".join(parts) + " Rupees Only"


def date_to_words(value):
    if not value:
        return "N/A"
    try:
        d = datetime.strptime(value, "%Y-%m-%d")
        return d.strftime("%d %B %Y")
    except ValueError:
        return value


def academic_years(student):
    try:
        start = datetime.strptime(student["admission_date"], "%Y-%m-%d").year
    except Exception:
        start = date.today().year
    order = {name: i for i, name in enumerate(CLASS_ORDER)}
    a, b = order.get(student.get("from_class")), order.get(student.get("to_class"))
    years = (b - a + 1) if a is not None and b is not None and b >= a else 1
    return f"{start} – {start + years - 1}"


@app.route("/login", methods=["GET", "POST"])
def login():
    if current_user():
        return redirect(url_for("dashboard"))
    if request.method == "POST":
        username = clean(request.form.get("username"))
        password = request.form.get("password", "")
        user = get_db().execute("SELECT * FROM users WHERE username=? AND active=1", (username,)).fetchone()
        if user and check_password_hash(user["password_hash"], password):
            session.clear()
            session["user_id"] = user["id"]
            flash(f"Welcome, {user['full_name']}.", "success")
            return redirect(request.args.get("next") or url_for("dashboard"))
        flash("Invalid username or password.", "danger")
    return render_template("login.html")


@app.route("/logout")
def logout():
    session.clear()
    flash("You have been logged out.", "success")
    return redirect(url_for("login"))


@app.route("/")
@login_required
def dashboard():
    db = get_db()
    counts = {
        "students": db.execute("SELECT COUNT(*) c FROM students WHERE active=1").fetchone()["c"],
        "payments": db.execute("SELECT COUNT(*) c FROM payments").fetchone()["c"],
        "today_attendance": db.execute("SELECT COUNT(*) c FROM attendance WHERE attendance_date=?", (date.today().isoformat(),)).fetchone()["c"],
    }
    totals = db.execute("SELECT COALESCE(SUM(school_fee+vehicle_fee),0) total_fee FROM students WHERE active=1").fetchone()["total_fee"]
    paid = db.execute("SELECT COALESCE(SUM(school_amount+vehicle_amount),0) total_paid FROM payments").fetchone()["total_paid"]
    counts["total_fee"] = float(totals)
    counts["total_paid"] = float(paid)
    counts["total_due"] = max(counts["total_fee"] - counts["total_paid"], 0)
    recent = db.execute("""
        SELECT p.*, s.name, s.admission_no FROM payments p JOIN students s ON s.id=p.student_id
        ORDER BY p.id DESC LIMIT 8
    """).fetchall()
    return render_template("dashboard.html", counts=counts, recent=recent)


@app.route("/students")
@login_required
def students():
    db = get_db()
    q = clean(request.args.get("q"))
    if q:
        like = f"%{q}%"
        rows = db.execute("""SELECT * FROM students WHERE active=1 AND
            (name LIKE ? OR admission_no LIKE ? OR phone LIKE ? OR father LIKE ? OR village LIKE ? OR current_class LIKE ?)
            ORDER BY id DESC""", (like, like, like, like, like, like)).fetchall()
    else:
        rows = db.execute("SELECT * FROM students WHERE active=1 ORDER BY id DESC").fetchall()
    data = [enrich_student(r) for r in rows]
    return render_template("students.html", students=data, q=q)


@app.route("/students/new", methods=["GET", "POST"])
@login_required
def student_new():
    if request.method == "POST":
        return save_student(None)
    return render_template("student_form.html", student=None, balances=None, caste_data=CASTE_DATA, classes=CLASS_ORDER, sections=SECTIONS)


@app.route("/students/<int:student_id>/edit", methods=["GET", "POST"])
@login_required
def student_edit(student_id):
    db = get_db()
    row = db.execute("SELECT * FROM students WHERE id=? AND active=1", (student_id,)).fetchone()
    if not row:
        flash("Student not found.", "danger")
        return redirect(url_for("students"))
    if request.method == "POST":
        return save_student(student_id)
    return render_template("student_form.html", student=dict(row), balances=student_balances(student_id), caste_data=CASTE_DATA, classes=CLASS_ORDER, sections=SECTIONS)


def save_student(student_id):
    db = get_db()
    f = request.form
    try:
        name = clean(f.get("name")); dob = clean(f.get("dob")); current_class = clean(f.get("current_class")); section = clean(f.get("section"))
        admission_no = clean(f.get("admission_no")) or (db.execute("SELECT admission_no FROM students WHERE id=?", (student_id,)).fetchone()[0] if student_id else next_admission_no())
        if not name or not dob or not current_class or not section:
            raise ValueError("Student name, date of birth, class and section are required.")
        school_fee = parse_amount(f.get("school_fee")); vehicle_fee = parse_amount(f.get("vehicle_fee"))
        vehicle_required = clean(f.get("vehicle_required")) or "No"
        admission_date = clean(f.get("admission_date")) or date.today().isoformat()
        fields = (admission_no, name, dob, current_class, clean(f.get("from_class")) or current_class, clean(f.get("to_class")) or current_class,
                  section, clean(f.get("father")), clean(f.get("mother")), clean(f.get("phone")), clean(f.get("religion")), clean(f.get("caste")),
                  clean(f.get("subcaste")), clean(f.get("nationality")) or "Indian", clean(f.get("place_of_birth")), admission_date, clean(f.get("village")),
                  school_fee, vehicle_required, vehicle_fee)
        if student_id:
            db.execute("""UPDATE students SET admission_no=?,name=?,dob=?,current_class=?,from_class=?,to_class=?,section=?,father=?,mother=?,phone=?,religion=?,caste=?,subcaste=?,nationality=?,place_of_birth=?,admission_date=?,village=?,school_fee=?,vehicle_required=?,vehicle_fee=?,updated_at=CURRENT_TIMESTAMP WHERE id=?""", fields + (student_id,))
            flash("Student details updated successfully.", "success")
        else:
            cur = db.execute("""INSERT INTO students(admission_no,name,dob,current_class,from_class,to_class,section,father,mother,phone,religion,caste,subcaste,nationality,place_of_birth,admission_date,village,school_fee,vehicle_required,vehicle_fee)
                VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)""", fields)
            student_id = cur.lastrowid
            initial_school = parse_amount(f.get("initial_school_paid")); initial_vehicle = parse_amount(f.get("initial_vehicle_paid"))
            if initial_school > school_fee or initial_vehicle > vehicle_fee:
                raise ValueError("Initial payment cannot be greater than the corresponding fee.")
            if initial_school or initial_vehicle:
                cur = db.execute("INSERT INTO payments(receipt_no,student_id,payment_date,method,school_amount,vehicle_amount,total_amount,created_by) VALUES('PENDING',?,?,?,?,?,?,?)", (student_id, admission_date, clean(f.get("initial_payment_method")) or "Cash", initial_school, initial_vehicle, initial_school + initial_vehicle, current_user()["id"]))
                p_id = cur.lastrowid
                db.execute("UPDATE payments SET receipt_no=? WHERE id=?", (receipt_no(p_id), p_id))
            flash(f"Student registered successfully. Admission No: {admission_no}", "success")
        db.commit()
    except sqlite3.IntegrityError:
        db.rollback()
        flash("Admission/registration number already exists. Please use a unique number.", "danger")
        return redirect(request.url)
    except ValueError as e:
        db.rollback()
        flash(str(e), "danger")
        return redirect(request.url)
    return redirect(url_for("student_detail", student_id=student_id))


@app.route("/students/<int:student_id>")
@login_required
def student_detail(student_id):
    db = get_db()
    row = db.execute("SELECT * FROM students WHERE id=? AND active=1", (student_id,)).fetchone()
    if not row:
        flash("Student not found.", "danger")
        return redirect(url_for("students"))
    student = enrich_student(row)
    payments = db.execute("SELECT p.*, u.full_name FROM payments p LEFT JOIN users u ON u.id=p.created_by WHERE p.student_id=? ORDER BY p.id DESC", (student_id,)).fetchall()
    return render_template("student_detail.html", student=student, payments=payments, academic_years=academic_years(student))


@app.route("/students/<int:student_id>/delete", methods=["POST"])
@role_required("admin")
def student_delete(student_id):
    db = get_db()
    db.execute("UPDATE students SET active=0, updated_at=CURRENT_TIMESTAMP WHERE id=?", (student_id,))
    db.commit()
    flash("Student record archived. Payment and attendance history remains in the database.", "success")
    return redirect(url_for("students"))


@app.route("/students/<int:student_id>/payments", methods=["POST"])
@login_required
def add_payment(student_id):
    db = get_db()
    student = db.execute("SELECT * FROM students WHERE id=? AND active=1", (student_id,)).fetchone()
    if not student:
        flash("Student not found.", "danger")
        return redirect(url_for("students"))
    try:
        school_amount = parse_amount(request.form.get("school_amount")); vehicle_amount = parse_amount(request.form.get("vehicle_amount"))
        balances = student_balances(student_id)
        if school_amount > balances["school_due"] or vehicle_amount > balances["vehicle_due"]:
            raise ValueError("Payment cannot be greater than the remaining fee balance.")
        total = school_amount + vehicle_amount
        if total <= 0:
            raise ValueError("Enter a payment amount greater than zero.")
        method = clean(request.form.get("method")) or "Cash"
        if method not in PAYMENT_METHODS: raise ValueError("Invalid payment method.")
        payment_date = clean(request.form.get("payment_date")) or date.today().isoformat()
        cur = db.execute("INSERT INTO payments(receipt_no,student_id,payment_date,method,school_amount,vehicle_amount,total_amount,created_by) VALUES('PENDING',?,?,?,?,?,?,?)", (student_id, payment_date, method, school_amount, vehicle_amount, total, current_user()["id"]))
        pid = cur.lastrowid
        rno = receipt_no(pid)
        db.execute("UPDATE payments SET receipt_no=? WHERE id=?", (rno, pid))
        db.commit()
        flash(f"Payment saved. Receipt {rno} generated.", "success")
        return redirect(url_for("receipt", payment_id=pid))
    except ValueError as e:
        db.rollback(); flash(str(e), "danger")
        return redirect(url_for("student_detail", student_id=student_id))


@app.route("/payments/<int:payment_id>/receipt")
@login_required
def receipt(payment_id):
    db = get_db()
    p = db.execute("SELECT p.*, s.*, u.full_name AS created_by_name FROM payments p JOIN students s ON s.id=p.student_id LEFT JOIN users u ON u.id=p.created_by WHERE p.id=?", (payment_id,)).fetchone()
    if not p:
        flash("Receipt not found.", "danger")
        return redirect(url_for("students"))
    student = enrich_student(db.execute("SELECT * FROM students WHERE id=?", (p["student_id"],)).fetchone())
    prior = db.execute("SELECT COALESCE(SUM(school_amount),0) school, COALESCE(SUM(vehicle_amount),0) vehicle FROM payments WHERE student_id=? AND id < ?", (p["student_id"], payment_id)).fetchone()
    return render_template("receipt.html", payment=p, student=student, prior=prior, number_to_words=number_to_words)


@app.route("/students/<int:student_id>/bonafide")
@login_required
def bonafide(student_id):
    db=get_db(); row=db.execute("SELECT * FROM students WHERE id=?",(student_id,)).fetchone()
    if not row: return redirect(url_for("students"))
    student=enrich_student(row)
    return render_template("bonafide.html", student=student, academic_years=academic_years(student), date_to_words=date_to_words)


@app.route("/students/<int:student_id>/tc")
@login_required
def tc(student_id):
    db=get_db(); row=db.execute("SELECT * FROM students WHERE id=?",(student_id,)).fetchone()
    if not row: return redirect(url_for("students"))
    student=enrich_student(row)
    return render_template("tc.html", student=student, date_to_words=date_to_words)


@app.route("/students/<int:student_id>/id-card")
@login_required
def id_card(student_id):
    db=get_db(); row=db.execute("SELECT * FROM students WHERE id=?",(student_id,)).fetchone()
    if not row: return redirect(url_for("students"))
    student=enrich_student(row)
    qr_data = f"CTS|{student['admission_no']}|{student['name']}|{student['current_class']}|{student['section']}"
    qr_image = ""
    try:
        import qrcode
        import base64
        qr = qrcode.make(qr_data)
        buf = BytesIO(); qr.save(buf, format="PNG")
        qr_image = "data:image/png;base64," + base64.b64encode(buf.getvalue()).decode()
    except Exception:
        qr_image = ""
    return render_template("id_card.html", student=student, qr_image=qr_image)


@app.route("/attendance", methods=["GET", "POST"])
@login_required
def attendance():
    db = get_db()
    selected_date = clean(request.values.get("attendance_date")) or date.today().isoformat()
    selected_class = clean(request.values.get("current_class"))
    selected_section = clean(request.values.get("section"))
    if request.method == "POST":
        student_ids = request.form.getlist("student_id")
        for sid in student_ids:
            status = request.form.get(f"status_{sid}", "Present")
            remarks = clean(request.form.get(f"remarks_{sid}"))
            db.execute("""INSERT INTO attendance(student_id,attendance_date,status,remarks,marked_by) VALUES(?,?,?,?,?)
                ON CONFLICT(student_id,attendance_date) DO UPDATE SET status=excluded.status, remarks=excluded.remarks, marked_by=excluded.marked_by""", (int(sid), selected_date, status, remarks, current_user()["id"]))
        db.commit()
        flash("Attendance saved successfully.", "success")
    rows=[]
    if selected_class and selected_section:
        rows=db.execute("""SELECT s.*, a.status, a.remarks FROM students s LEFT JOIN attendance a ON a.student_id=s.id AND a.attendance_date=?
            WHERE s.active=1 AND s.current_class=? AND s.section=? ORDER BY s.name""", (selected_date,selected_class,selected_section)).fetchall()
    return render_template("attendance.html", students=rows, classes=CLASS_ORDER, sections=SECTIONS, selected_date=selected_date, selected_class=selected_class, selected_section=selected_section, statuses=ATTENDANCE_STATUSES)


@app.route("/reports")
@login_required
def reports():
    db=get_db()
    class_filter=clean(request.args.get("current_class")); section=clean(request.args.get("section"))
    where="WHERE s.active=1"; params=[]
    if class_filter: where += " AND s.current_class=?"; params.append(class_filter)
    if section: where += " AND s.section=?"; params.append(section)
    students_rows=db.execute(f"SELECT s.* FROM students s {where} ORDER BY s.current_class,s.section,s.name",params).fetchall()
    total_fee=sum(float(r["school_fee"])+float(r["vehicle_fee"]) for r in students_rows)
    total_paid=db.execute(f"SELECT COALESCE(SUM(p.total_amount),0) x FROM payments p JOIN students s ON s.id=p.student_id {where}",params).fetchone()["x"]
    return render_template("reports.html", students=[enrich_student(r) for r in students_rows], total_fee=total_fee, total_paid=float(total_paid), total_due=max(total_fee-float(total_paid),0), classes=CLASS_ORDER, sections=SECTIONS, class_filter=class_filter, section=section)


@app.route("/reports/students.xlsx")
@login_required
def export_students_xlsx():
    try:
        from openpyxl import Workbook
        from openpyxl.styles import Font
    except ImportError:
        flash("Excel export needs openpyxl. Run: pip install -r requirements.txt", "danger")
        return redirect(url_for("reports"))
    rows=get_db().execute("SELECT * FROM students WHERE active=1 ORDER BY id").fetchall()
    wb=Workbook(); ws=wb.active; ws.title="Students"
    headers=["Admission No","Student Name","DOB","Class","Section","Father","Mother","Phone","Religion","Caste","Subcaste","Village","School Fee","School Paid","School Due","Vehicle","Vehicle Fee","Vehicle Paid","Vehicle Due","Admission Date"]
    ws.append(headers)
    for c in ws[1]: c.font=Font(bold=True)
    for r in rows:
        s=enrich_student(r); ws.append([s["admission_no"],s["name"],s["dob"],s["current_class"],s["section"],s["father"],s["mother"],s["phone"],s["religion"],s["caste"],s["subcaste"],s["village"],s["school_fee"],s["school_paid"],s["school_due"],s["vehicle_required"],s["vehicle_fee"],s["vehicle_paid"],s["vehicle_due"],s["admission_date"]])
    for col in ws.columns: ws.column_dimensions[col[0].column_letter].width=min(max(len(str(col[0].value or ""))+2,12),28)
    buf=BytesIO(); wb.save(buf); buf.seek(0)
    return send_file(buf,as_attachment=True,download_name=f"CTS_Students_{date.today().isoformat()}.xlsx",mimetype="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet")


@app.route("/reports/payments.xlsx")
@login_required
def export_payments_xlsx():
    try:
        from openpyxl import Workbook
        from openpyxl.styles import Font
    except ImportError:
        flash("Excel export needs openpyxl. Run: pip install -r requirements.txt", "danger")
        return redirect(url_for("reports"))
    rows=get_db().execute("SELECT p.*,s.admission_no,s.name FROM payments p JOIN students s ON s.id=p.student_id ORDER BY p.id DESC").fetchall()
    wb=Workbook(); ws=wb.active; ws.title="Payments"; headers=["Receipt No","Date","Admission No","Student","Method","School Amount","Vehicle Amount","Total","Created By"]
    ws.append(headers)
    for c in ws[1]: c.font=Font(bold=True)
    for r in rows: ws.append([r["receipt_no"],r["payment_date"],r["admission_no"],r["name"],r["method"],r["school_amount"],r["vehicle_amount"],r["total_amount"],r["created_by"]])
    buf=BytesIO(); wb.save(buf); buf.seek(0)
    return send_file(buf,as_attachment=True,download_name=f"CTS_Payments_{date.today().isoformat()}.xlsx",mimetype="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet")


@app.route("/reports/payments.pdf")
@login_required
def export_payments_pdf():
    try:
        from reportlab.lib import colors
        from reportlab.lib.pagesizes import A4, landscape
        from reportlab.lib.styles import getSampleStyleSheet
        from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph, Spacer
    except ImportError:
        flash("PDF export needs reportlab. Run: pip install -r requirements.txt", "danger")
        return redirect(url_for("reports"))
    rows=get_db().execute("SELECT p.*,s.admission_no,s.name FROM payments p JOIN students s ON s.id=p.student_id ORDER BY p.id DESC").fetchall()
    buf=BytesIO(); doc=SimpleDocTemplate(buf,pagesize=landscape(A4),rightMargin=25,leftMargin=25,topMargin=25,bottomMargin=25)
    styles=getSampleStyleSheet(); story=[Paragraph(app.config["SCHOOL_NAME"],styles["Title"]),Paragraph("Payment Report",styles["Heading2"]),Spacer(1,10)]
    data=[["Receipt","Date","Adm No","Student","Mode","School","Vehicle","Total"]]
    for r in rows: data.append([r["receipt_no"],r["payment_date"],r["admission_no"],r["name"],r["method"],money(r["school_amount"]),money(r["vehicle_amount"]),money(r["total_amount"])])
    table=Table(data,repeatRows=1)
    table.setStyle(TableStyle([("BACKGROUND",(0,0),(-1,0),colors.HexColor("#17365d")),("TEXTCOLOR",(0,0),(-1,0),colors.white),("GRID",(0,0),(-1,-1),0.4,colors.grey),("FONTSIZE",(0,0),(-1,-1),8),("VALIGN",(0,0),(-1,-1),"MIDDLE")]))
    story.append(table); doc.build(story); buf.seek(0)
    return send_file(buf,as_attachment=True,download_name=f"CTS_Payments_{date.today().isoformat()}.pdf",mimetype="application/pdf")


@app.route("/change-password", methods=["GET", "POST"])
@login_required
def change_password():
    user = current_user()
    if request.method == "POST":
        old = request.form.get("old_password", "")
        new = request.form.get("new_password", "")
        confirm = request.form.get("confirm_password", "")
        if not check_password_hash(user["password_hash"], old):
            flash("Current password is incorrect.", "danger")
        elif len(new) < 6:
            flash("New password must contain at least 6 characters.", "danger")
        elif new != confirm:
            flash("New password and confirmation do not match.", "danger")
        else:
            db = get_db()
            db.execute("UPDATE users SET password_hash=? WHERE id=?", (generate_password_hash(new), user["id"]))
            db.commit()
            flash("Password changed successfully.", "success")
            return redirect(url_for("dashboard"))
    return render_template("change_password.html")


@app.route("/users", methods=["GET", "POST"])
@role_required("admin")
def users():
    db=get_db()
    if request.method=="POST":
        username=clean(request.form.get("username")); password=request.form.get("password",""); role=clean(request.form.get("role")); full_name=clean(request.form.get("full_name"))
        if not username or len(password)<6 or role not in ("admin","user") or not full_name:
            flash("Enter full name, username, role and a password of at least 6 characters.","danger")
        else:
            try:
                db.execute("INSERT INTO users(username,password_hash,role,full_name) VALUES(?,?,?,?)",(username,generate_password_hash(password),role,full_name)); db.commit(); flash("User created.","success")
            except sqlite3.IntegrityError: db.rollback(); flash("Username already exists.","danger")
    rows=db.execute("SELECT id,username,role,full_name,active,created_at FROM users ORDER BY id").fetchall()
    return render_template("users.html",users=rows)


@app.route("/users/<int:user_id>/toggle", methods=["POST"])
@role_required("admin")
def toggle_user(user_id):
    if user_id==current_user()["id"]:
        flash("You cannot deactivate your own account.","danger")
        return redirect(url_for("users"))
    db=get_db(); db.execute("UPDATE users SET active=CASE active WHEN 1 THEN 0 ELSE 1 END WHERE id=?",(user_id,)); db.commit(); flash("User status updated.","success"); return redirect(url_for("users"))


@app.route("/students/<int:student_id>/notify/<channel>", methods=["POST"])
@login_required
def notify(student_id, channel):
    db=get_db(); s=db.execute("SELECT * FROM students WHERE id=? AND active=1",(student_id,)).fetchone()
    if not s: flash("Student not found.","danger"); return redirect(url_for("students"))
    balances=student_balances(student_id)
    message=(f"{app.config['SCHOOL_NAME']}: Fee reminder for {s['name']} ({s['admission_no']}). "
             f"School due {money(balances['school_due'])}, vehicle due {money(balances['vehicle_due'])}, total due {money(balances['total_due'])}.")
    phone=clean(s["phone"])
    if not phone:
        flash("This student has no parent phone number.","danger"); return redirect(url_for("student_detail",student_id=student_id))
    status="Failed"; provider_sid=""; error=""
    try:
        from twilio.rest import Client
        sid=os.getenv("TWILIO_ACCOUNT_SID"); token=os.getenv("TWILIO_AUTH_TOKEN")
        if not sid or not token: raise RuntimeError("Twilio credentials are not configured.")
        client=Client(sid,token)
        to = phone if phone.startswith("+") else "+91" + phone.lstrip("0")
        if channel=="sms":
            sender=os.getenv("TWILIO_SMS_FROM")
            if not sender: raise RuntimeError("TWILIO_SMS_FROM is not configured.")
            result=client.messages.create(body=message,from_=sender,to=to)
        elif channel=="whatsapp":
            sender=os.getenv("TWILIO_WHATSAPP_FROM","whatsapp:+14155238886")
            result=client.messages.create(body=message,from_=sender,to="whatsapp:"+to)
        else:
            raise RuntimeError("Unsupported notification channel.")
        status="Sent"; provider_sid=result.sid
        flash(f"{channel.upper()} notification sent.","success")
    except Exception as exc:
        error=str(exc); flash(f"Notification not sent: {error}","danger")
    db.execute("INSERT INTO notifications(student_id,channel,recipient,message,status,provider_sid,created_by) VALUES(?,?,?,?,?,?,?)",(student_id,channel,phone,message,status,provider_sid,current_user()["id"]))
    db.commit()
    return redirect(url_for("student_detail",student_id=student_id))


@app.template_filter("money")
def money_filter(v): return money(v)

@app.template_filter("date_in")
def date_in_filter(v):
    if not v: return ""
    try: return datetime.strptime(v,"%Y-%m-%d").strftime("%d-%m-%Y")
    except Exception: return v


if __name__ == "__main__":
    init_db()
    app.run(host="127.0.0.1", port=int(os.getenv("PORT","5000")), debug=os.getenv("FLASK_DEBUG","0")=="1")

```

## `requirements.txt`

```text
Flask>=3.0,<4.0
openpyxl>=3.1,<4.0
reportlab>=4.0,<5.0
qrcode[pil]>=7.4,<9.0
Pillow>=10.0,<13.0
# Optional for SMS/WhatsApp integration:
twilio>=9.0,<10.0

```

## `run.bat`

```text
@echo off
cd /d "%~dp0"
where py >nul 2>nul
if errorlevel 1 (
  echo Python was not found. Install Python 3.11+ and enable Add Python to PATH.
  pause
  exit /b 1
)
if not exist ".venv\Scripts\python.exe" (
  echo Creating virtual environment...
  py -m venv .venv
)
call ".venv\Scripts\activate.bat"
python -m pip install -r requirements.txt
python app.py
pause

```

## `backup_db.py`

```python
from pathlib import Path
from datetime import datetime
import shutil

BASE=Path(__file__).resolve().parent
DB=BASE/'school.db'
BACKUP_DIR=BASE/'backups'
BACKUP_DIR.mkdir(exist_ok=True)
if not DB.exists():
    raise SystemExit('school.db does not exist yet. Run app.py once first.')
out=BACKUP_DIR/f"school_{datetime.now():%Y%m%d_%H%M%S}.db"
shutil.copy2(DB,out)
print(f'Backup created: {out}')

```

## `.env.example`

```text
# Change this in a real deployment
SECRET_KEY=replace-with-a-long-random-secret
ADMIN_PASSWORD=change-admin-password
PORT=5000
FLASK_DEBUG=0

# Optional Twilio SMS / WhatsApp integration
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_SMS_FROM=
TWILIO_WHATSAPP_FROM=whatsapp:+14155238886

```

## `README.md`

```text
# CHATHURVEDA TALENT SCHOOL - Management System

A corrected replacement for the original single-page localStorage application.

## What changed

- Permanent SQLite database (`school.db`) instead of browser `localStorage`.
- Secure login with Admin/User roles and password hashing.
- Student registration/edit/search/archive.
- Fee collection with payment history and unique receipt numbers.
- Printable fee receipt, Bonafide/Conduct Certificate and Transfer Certificate.
- Printable student ID card with QR code.
- Daily attendance by class/section.
- Fee dashboard, due calculations and Excel/PDF exports.
- Admin account management.
- Optional SMS and WhatsApp reminders through Twilio; messages are sent only when an authorized user clicks the button.
- School logo included as `static/logo.png`.
- Database records survive closing the browser and restarting the application.

## Windows setup

Open Command Prompt or PowerShell inside this folder:

```text
cd D:\Chaturveda-School-Management
py -m venv .venv
.venv\Scripts\activate
py -m pip install -r requirements.txt
py app.py
```

Then open:

```text
http://127.0.0.1:5000
```

First login:

```text
Username: admin
Password: admin123
```

Create a new admin/user account from **Users** and change the default password before actual school use.

## Database

The application automatically creates `school.db` on first run. Do not delete this file if you want to keep records.

For backup, simply copy `school.db` while the application is stopped. For a larger multi-computer deployment, move the same application to a server and consider PostgreSQL instead of SQLite.

## SMS / WhatsApp

Create a Twilio account, set the variables in `.env.example` as system environment variables, and restart the app. The notification buttons will then send the fee reminder to the parent's saved phone number.

WhatsApp also requires the relevant Twilio WhatsApp sender/template setup depending on the type of message and account.

## Important operational note

The app is designed for local/school-office use. Before exposing it to the public internet, use a production WSGI server, HTTPS, a strong secret key, a strong admin password, regular database backups and appropriate access controls.

```

## `CODE_REVIEW.md`

```text
# Thorough review of the original uploaded HTML application

The original file was a 1,644-line single HTML/CSS/JavaScript application. Its JavaScript is syntactically valid, but several design and data-integrity limitations make it unsuitable as the long-term school database.

## Problems found

1. **Browser localStorage is the only database.** Student and payment records live in the browser profile, not a central database. Closing the browser usually keeps them, but clearing site data, changing browser/profile/device, private browsing, or browser storage failure can remove access.
2. **No login or authorization.** Anyone who opens the page can view, edit, delete and clear records.
3. **The Clear All Stored Data button is dangerous.** It calls `localStorage.clear()` and `sessionStorage.clear()`, which clears all storage for the origin, not only the school's two keys.
4. **Payment history is not authoritative.** During student editing, `schoolPaid` and `vehiclePaid` can be directly changed without creating a payment transaction. This can make the displayed paid amount disagree with payment history.
5. **Receipt numbering is based on array length.** Deleting records can make a later receipt number collide with an older number. The new system uses a database payment ID.
6. **Student IDs use `Date.now()`.** This is not a reliable database identifier for a multi-user application.
7. **No multi-user support.** Two office computers cannot safely share the same browser-local data.
8. **No database transaction/rollback.** Browser writes can fail or become inconsistent if storage is unavailable or quota is reached.
9. **No attendance module.**
10. **No Excel/PDF reporting.**
11. **No notification provider integration.**
12. **No user/admin dashboards or user management.**
13. **No backup mechanism.**
14. **No server-side validation.** Client-side JavaScript validation can be bypassed by editing the page or storage.
15. **Search can fail on older/incomplete records.** Expressions such as `student.father.toLowerCase()` and `student.phone.includes()` assume those values always exist.
16. **Academic-year calculation is off by one in inclusive class ranges.** It calculates `endYear = startYear + yearsStudied`; an inclusive one-year study period should end in the same year.
17. **The currency number-to-words converter does not correctly handle Indian crore grouping.** The first two-digit group is labeled as lakh, so larger values can be wrong.
18. **Initial registration payments are forced to Cash.** The registration form has no payment method for the initial payment in the original code.
19. **No structured payment audit trail.** There is no authenticated user recorded against each payment.
20. **Deleting a student also deletes all payment history from localStorage.** This is undesirable for financial records. The replacement archives students instead.
21. **Certificates use hard-coded school/principal details.** The replacement centralizes school configuration in `app.py`.
22. **Large table UI becomes very wide.** The replacement uses focused student pages and a cleaner report layout.
23. **No permanent document/photo/ID-card workflow.** The replacement adds printable ID cards and QR codes.

## New architecture

The replacement in this folder uses Flask + SQLite. The database is `school.db`; payments and attendance are relational records linked to students; login passwords are hashed; and admin-only actions are protected server-side.

```

## `templates/attendance.html`

```html
{% extends 'base.html' %}{% block content %}<div class="page-head"><div><h2>Attendance Management</h2><p>Mark Present, Absent or Leave by class and section.</p></div></div><form method="get" class="filterbar"><div><label>Date</label><input type="date" name="attendance_date" value="{{ selected_date }}"></div><div><label>Class</label><select name="current_class">{% for c in classes %}<option {% if selected_class==c %}selected{% endif %}>{{ c }}</option>{% endfor %}</select></div><div><label>Section</label><select name="section">{% for s in sections %}<option {% if selected_section==s %}selected{% endif %}>{{ s }}</option>{% endfor %}</select></div><button class="btn primary">Load Students</button></form>{% if students %}<form method="post" class="card"><input type="hidden" name="attendance_date" value="{{ selected_date }}"><input type="hidden" name="current_class" value="{{ selected_class }}"><input type="hidden" name="section" value="{{ selected_section }}"><div class="table-scroll"><table><thead><tr><th>#</th><th>Admission</th><th>Student</th><th>Status</th><th>Remarks</th></tr></thead><tbody>{% for s in students %}<tr><td>{{ loop.index }}</td><td>{{ s.admission_no }}</td><td><b>{{ s.name }}</b></td><td><input type="hidden" name="student_id" value="{{ s.id }}"><select name="status_{{ s.id }}">{% for st in statuses %}<option {% if (s.status or 'Present')==st %}selected{% endif %}>{{ st }}</option>{% endfor %}</select></td><td><input name="remarks_{{ s.id }}" value="{{ s.remarks or '' }}" placeholder="Optional"></td></tr>{% endfor %}</tbody></table></div><div class="button-row"><button class="btn primary">Save Attendance</button></div></form>{% elif selected_class %}<div class="card empty">No active students found for this class/section.</div>{% endif %}{% endblock %}

```

## `templates/base.html`

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>{{ school_name }} - Management System</title>
<link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
<body>
<header class="topbar">
  <div class="brand">
    <img src="{{ url_for('static', filename='logo.png') }}" alt="School Logo">
    <div><h1>{{ school_name }}</h1><p>{{ school_location }} · Student Management & Fee Management System</p></div>
  </div>
  {% if current_user %}
  <div class="userbox"><strong>{{ current_user['full_name'] }}</strong><span>{{ current_user['role']|upper }}</span><a href="{{ url_for('logout') }}">Logout</a></div>
  {% endif %}
</header>
{% if current_user %}
<nav class="nav">
  <a href="{{ url_for('dashboard') }}">Dashboard</a>
  <a href="{{ url_for('students') }}">Students</a>
  <a href="{{ url_for('student_new') }}">+ New Student</a>
  <a href="{{ url_for('attendance') }}">Attendance</a>
  <a href="{{ url_for('reports') }}">Reports & Export</a><a href="{{ url_for('change_password') }}">Change Password</a>
  {% if current_user['role']=='admin' %}<a href="{{ url_for('users') }}">Users</a>{% endif %}
</nav>
{% endif %}
<main class="page">
{% with messages=get_flashed_messages(with_categories=true) %}
  {% if messages %}<div class="flash-wrap">{% for cat,msg in messages %}<div class="flash {{ cat }}">{{ msg }}</div>{% endfor %}</div>{% endif %}
{% endwith %}
{% block content %}{% endblock %}
</main>
<footer><b>{{ school_name }}</b> · {{ school_location }} · School Code: {{ school_code }}<br><span>© 2026 Student Management System</span></footer>
<script src="{{ url_for('static', filename='app.js') }}"></script>
{% block scripts %}{% endblock %}
</body>
</html>

```

## `templates/bonafide.html`

```html
{% extends 'base.html' %}{% block content %}<div class="print-page"><div class="print-toolbar"><button class="btn primary" onclick="window.print()">🖨 Print Bonafide</button><a class="btn light" href="{{ url_for('student_detail',student_id=student.id) }}">Back</a></div><div class="document certificate"><div class="doc-head"><img src="{{ url_for('static',filename='logo.png') }}"><div><h1>{{ school_name }}</h1><h2>{{ school_location }}</h2><p>Recognized by Govt. of Telangana · School Code: {{ school_code }}</p></div></div><h2 class="certificate-title">BONAFIDE & CONDUCT CERTIFICATE</h2><div class="doc-meta"><span>Ref No: CTS/BON/{{ student.id }}</span><span>Date: {{ today|date_in }}</span></div><p class="certificate-text">This is to certify that <b>{{ student.name }}</b>, Son/Daughter of Sri <b>{{ student.father or 'N/A' }}</b> and Smt. <b>{{ student.mother or 'N/A' }}</b>, residing at <b>{{ student.village or 'N/A' }}</b>, is/was a regular student of this institution, studied from Class <b>{{ student.from_class }}</b> to Class <b>{{ student.to_class }}</b> during the academic year(s) <b>{{ academic_years }}</b>.</p><table class="doc-table"><tr><td>Admission No</td><td>{{ student.admission_no }}</td></tr><tr><td>Date of Birth</td><td>{{ student.dob|date_in }} ({{ date_to_words(student.dob) }})</td></tr><tr><td>Nationality / Religion / Caste / Subcaste</td><td>{{ student.nationality }} / {{ student.religion or '-' }} / {{ student.caste or '-' }} / {{ student.subcaste or '-' }}</td></tr><tr><td>General Conduct</td><td>Good</td></tr></table><div class="signatures"><span>School Seal</span><span>Principal<br><b>({{ principal }})</b></span></div></div></div>{% endblock %}

```

## `templates/change_password.html`

```html
{% extends 'base.html' %}{% block content %}<div class="page-head"><div><h2>Change Password</h2><p>Update your own login password.</p></div></div><section class="card" style="max-width:600px"><form method="post" class="stack-form"><label>Current Password</label><input type="password" name="old_password" required><label>New Password</label><input type="password" name="new_password" minlength="6" required><label>Confirm New Password</label><input type="password" name="confirm_password" minlength="6" required><button class="btn primary">Change Password</button></form></section>{% endblock %}

```

## `templates/dashboard.html`

```html
{% extends 'base.html' %}{% block content %}
<div class="page-head"><div><h2>Admin / User Dashboard</h2><p>Central control panel for students, fees, attendance and reports.</p></div><a class="btn primary" href="{{ url_for('student_new') }}">+ Register Student</a></div>
<div class="stats">
  <div class="stat"><span>Total Active Students</span><strong>{{ counts.students }}</strong></div>
  <div class="stat"><span>Total Fee</span><strong>{{ counts.total_fee|money }}</strong></div>
  <div class="stat"><span>Total Collected</span><strong>{{ counts.total_paid|money }}</strong></div>
  <div class="stat danger-card"><span>Total Due</span><strong>{{ counts.total_due|money }}</strong></div>
  <div class="stat"><span>Payments Recorded</span><strong>{{ counts.payments }}</strong></div>
  <div class="stat"><span>Today's Attendance Records</span><strong>{{ counts.today_attendance }}</strong></div>
</div>
<div class="grid-2">
<section class="card"><div class="card-head"><h3>Quick Actions</h3></div><div class="quick-grid">
<a href="{{ url_for('student_new') }}" class="quick">🎓<b>Register Student</b><span>Admission + initial fee</span></a>
<a href="{{ url_for('students') }}" class="quick">👥<b>Student Records</b><span>Search and manage</span></a>
<a href="{{ url_for('attendance') }}" class="quick">📋<b>Attendance</b><span>Daily class attendance</span></a>
<a href="{{ url_for('reports') }}" class="quick">📊<b>Reports</b><span>Excel/PDF exports</span></a>
{% if current_user['role']=='admin' %}<a href="{{ url_for('users') }}" class="quick">🔐<b>User Accounts</b><span>Admin/User access</span></a>{% endif %}
</div></section>
<section class="card"><div class="card-head"><h3>Recent Payments</h3><a href="{{ url_for('reports') }}">View reports</a></div><div class="table-scroll"><table><thead><tr><th>Receipt</th><th>Student</th><th>Date</th><th>Mode</th><th>Total</th></tr></thead><tbody>{% for p in recent %}<tr><td>{{ p.receipt_no }}</td><td>{{ p.name }}<small>{{ p.admission_no }}</small></td><td>{{ p.payment_date|date_in }}</td><td>{{ p.method }}</td><td class="paid">{{ p.total_amount|money }}</td></tr>{% else %}<tr><td colspan="5" class="empty">No payments yet.</td></tr>{% endfor %}</tbody></table></div></section>
</div>
{% endblock %}

```

## `templates/id_card.html`

```html
{% extends 'base.html' %}{% block content %}<div class="print-page"><div class="print-toolbar"><button class="btn primary" onclick="window.print()">🖨 Print ID Card</button><a class="btn light" href="{{ url_for('student_detail',student_id=student.id) }}">Back</a></div><div class="id-card"><div class="id-head"><img src="{{ url_for('static',filename='logo.png') }}"><div><b>{{ school_name }}</b><small>{{ school_location }}</small></div></div><div class="id-body"><div class="id-photo">STUDENT<br>PHOTO</div><div class="id-details"><h2>{{ student.name }}</h2><p><b>Admission:</b> {{ student.admission_no }}</p><p><b>Class:</b> {{ student.current_class }} - {{ student.section }}</p><p><b>DOB:</b> {{ student.dob|date_in }}</p><p><b>Father:</b> {{ student.father or '-' }}</p><p><b>Phone:</b> {{ student.phone or '-' }}</p><p><b>Village:</b> {{ student.village or '-' }}</p></div>{% if qr_image %}<img class="qr" src="{{ qr_image }}" alt="QR">{% endif %}</div><div class="id-foot">{{ school_name }} · {{ school_code }} · Valid Student Identity Card</div></div></div>{% endblock %}

```

## `templates/login.html`

```html
<!doctype html><html lang="en"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1"><title>Login - CHATHURVEDA TALENT SCHOOL</title><link rel="stylesheet" href="{{ url_for('static',filename='style.css') }}"></head>
<body class="login-page"><div class="login-card"><img src="{{ url_for('static',filename='logo.png') }}" class="login-logo" alt="School Logo"><h1>CHATHURVEDA TALENT SCHOOL</h1><p>SURAMPALLY · Secure Management Portal</p><form method="post"><label>Username</label><input name="username" autocomplete="username" required autofocus><label>Password</label><input type="password" name="password" autocomplete="current-password" required><button class="btn primary wide">Sign In</button></form><div class="login-note">Default first-run administrator: <b>admin</b> / <b>admin123</b>. Change the password/environment before real school use.</div></div></body></html>

```

## `templates/receipt.html`

```html
{% extends 'base.html' %}{% block content %}<div class="print-page"><div class="print-toolbar"><button class="btn primary" onclick="window.print()">🖨 Print Receipt</button><a class="btn light" href="{{ url_for('student_detail',student_id=student.id) }}">Back</a></div><div class="document"><div class="doc-head"><img src="{{ url_for('static',filename='logo.png') }}"><div><h1>{{ school_name }}</h1><h2>{{ school_location }}</h2><p>FEE ACKNOWLEDGEMENT RECEIPT</p></div></div><div class="doc-meta"><span><b>Receipt No:</b> {{ payment.receipt_no }}</span><span><b>Date:</b> {{ payment.payment_date|date_in }}</span></div><div class="receipt-info"><p><b>Student:</b> {{ student.name }}</p><p><b>Admission No:</b> {{ student.admission_no }}</p><p><b>Class:</b> {{ student.current_class }} - {{ student.section }}</p><p><b>Father:</b> {{ student.father or '-' }}</p><p><b>Phone:</b> {{ student.phone or '-' }}</p><p><b>Village:</b> {{ student.village or '-' }}</p><p><b>Payment Mode:</b> {{ payment.method }}</p></div><table class="doc-table"><tr><th>Particular</th><th>Total Fee</th><th>Previous Paid</th><th>Current Paid</th><th>Balance Due</th></tr><tr><td>School Fee</td><td>{{ student.school_fee|money }}</td><td>{{ prior.school|money }}</td><td>{{ payment.school_amount|money }}</td><td>{{ student.school_due|money }}</td></tr><tr><td>Vehicle Fee</td><td>{{ student.vehicle_fee|money }}</td><td>{{ prior.vehicle|money }}</td><td>{{ payment.vehicle_amount|money }}</td><td>{{ student.vehicle_due|money }}</td></tr><tr class="total"><td>Grand Total</td><td>{{ (student.school_fee+student.vehicle_fee)|money }}</td><td>{{ (prior.school+prior.vehicle)|money }}</td><td>{{ payment.total_amount|money }}</td><td>{{ student.total_due|money }}</td></tr></table><p class="words"><b>Amount Received in Words:</b> {{ number_to_words(payment.total_amount) }}</p><div class="signatures"><span>Parent Signature</span><span>Authorized Signature</span></div></div></div>{% endblock %}

```

## `templates/reports.html`

```html
{% extends 'base.html' %}{% block content %}<div class="page-head"><div><h2>Fee Reports & Export</h2><p>Review fee balances and export records for office use.</p></div></div><div class="stats"><div class="stat"><span>Total Fee</span><strong>{{ total_fee|money }}</strong></div><div class="stat"><span>Collected</span><strong>{{ total_paid|money }}</strong></div><div class="stat danger-card"><span>Due</span><strong>{{ total_due|money }}</strong></div><div class="stat"><span>Students</span><strong>{{ students|length }}</strong></div></div><form class="filterbar" method="get"><div><label>Class</label><select name="current_class"><option value="">All Classes</option>{% for c in classes %}<option {% if class_filter==c %}selected{% endif %}>{{ c }}</option>{% endfor %}</select></div><div><label>Section</label><select name="section"><option value="">All Sections</option>{% for s in sections %}<option {% if section==s %}selected{% endif %}>{{ s }}</option>{% endfor %}</select></div><button class="btn primary">Apply Filter</button></form><div class="card"><div class="button-row"><a class="btn green" href="{{ url_for('export_students_xlsx') }}">⬇ Excel - Students</a><a class="btn blue" href="{{ url_for('export_payments_xlsx') }}">⬇ Excel - Payments</a><a class="btn orange" href="{{ url_for('export_payments_pdf') }}">⬇ PDF - Payments</a></div><div class="table-scroll"><table><thead><tr><th>Admission</th><th>Student</th><th>Class</th><th>School Fee</th><th>School Paid</th><th>School Due</th><th>Vehicle Due</th><th>Total Due</th></tr></thead><tbody>{% for s in students %}<tr><td>{{ s.admission_no }}</td><td>{{ s.name }}</td><td>{{ s.current_class }} - {{ s.section }}</td><td>{{ s.school_fee|money }}</td><td class="paid">{{ s.school_paid|money }}</td><td class="due">{{ s.school_due|money }}</td><td class="due">{{ s.vehicle_due|money }}</td><td class="due"><b>{{ s.total_due|money }}</b></td></tr>{% else %}<tr><td colspan="8" class="empty">No records.</td></tr>{% endfor %}</tbody></table></div></div>{% endblock %}

```

## `templates/student_detail.html`

```html
{% extends 'base.html' %}{% block content %}
<div class="page-head"><div><h2>{{ student.name }}</h2><p>{{ student.admission_no }} · {{ student.current_class }} - Section {{ student.section }}</p></div><div class="button-row"><a class="btn orange" href="{{ url_for('student_edit',student_id=student.id) }}">Edit</a><a class="btn light" href="{{ url_for('students') }}">Back</a></div></div>
<div class="profile-grid"><section class="card"><h3>Student Profile</h3><div class="detail-grid">{% for label,key in [('Admission No','admission_no'),('DOB','dob'),('Class','current_class'),('Section','section'),('Father','father'),('Mother','mother'),('Phone','phone'),('Village','village'),('Religion','religion'),('Caste','caste'),('Subcaste','subcaste'),('Nationality','nationality'),('Place of Birth','place_of_birth'),('Admission Date','admission_date'),('Academic Years','__academic')] %}<div><span>{{ label }}</span><b>{% if key=='__academic' %}{{ academic_years }}{% elif key in ['dob','admission_date'] %}{{ student[key]|date_in }}{% else %}{{ student[key] or '-' }}{% endif %}</b></div>{% endfor %}</div></section>
<section class="card"><h3>Fee Summary</h3><div class="fee-summary"><div><span>School Fee</span><b>{{ student.school_fee|money }}</b></div><div><span>School Paid</span><b class="paid">{{ student.school_paid|money }}</b></div><div><span>School Due</span><b class="due">{{ student.school_due|money }}</b></div><div><span>Vehicle Fee</span><b>{{ student.vehicle_fee|money }}</b></div><div><span>Vehicle Paid</span><b class="paid">{{ student.vehicle_paid|money }}</b></div><div><span>Vehicle Due</span><b class="due">{{ student.vehicle_due|money }}</b></div><div class="total"><span>Total Due</span><b class="due">{{ student.total_due|money }}</b></div></div>
<div class="button-row"><a class="btn primary" href="#payment">Record Payment</a><a class="btn blue" href="{{ url_for('id_card',student_id=student.id) }}">Student ID Card</a><a class="btn purple" href="{{ url_for('bonafide',student_id=student.id) }}">Bonafide</a><a class="btn orange" href="{{ url_for('tc',student_id=student.id) }}">TC</a></div></section></div>
<section class="card" id="payment"><div class="card-head"><h3>Record Fee Payment</h3><span class="badge">Balance: {{ student.total_due|money }}</span></div><form method="post" action="{{ url_for('add_payment',student_id=student.id) }}" class="payment-form"><div><label>Payment Date</label><input type="date" name="payment_date" value="{{ today }}" required></div><div><label>Payment Method</label><select name="method">{% for m in ['Cash','UPI','Card'] %}<option>{{ m }}</option>{% endfor %}</select></div><div><label>School Payment</label><input type="number" min="0" step="0.01" name="school_amount" value="0"></div><div><label>Vehicle Payment</label><input type="number" min="0" step="0.01" name="vehicle_amount" value="0"></div><button class="btn primary">Save Payment & Generate Receipt</button></form></section>
<section class="card"><div class="card-head"><h3>Notification</h3><span>{{ student.phone or 'No parent phone number' }}</span></div><div class="button-row"><form method="post" action="{{ url_for('notify',student_id=student.id,channel='sms') }}"><button class="btn blue" {% if not student.phone %}disabled{% endif %}>Send SMS Fee Reminder</button></form><form method="post" action="{{ url_for('notify',student_id=student.id,channel='whatsapp') }}"><button class="btn green" {% if not student.phone %}disabled{% endif %}>Send WhatsApp Reminder</button></form></div><p class="hint">SMS/WhatsApp requires Twilio credentials in the environment. The app does not send anything automatically.</p></section>
<section class="card"><div class="card-head"><h3>Payment History</h3></div><div class="table-scroll"><table><thead><tr><th>Receipt</th><th>Date</th><th>Mode</th><th>School</th><th>Vehicle</th><th>Total</th><th>Created By</th><th></th></tr></thead><tbody>{% for p in payments %}<tr><td>{{ p.receipt_no }}</td><td>{{ p.payment_date|date_in }}</td><td>{{ p.method }}</td><td>{{ p.school_amount|money }}</td><td>{{ p.vehicle_amount|money }}</td><td class="paid">{{ p.total_amount|money }}</td><td>{{ p.full_name or '-' }}</td><td><a class="mini blue" href="{{ url_for('receipt',payment_id=p.id) }}">Receipt</a></td></tr>{% else %}<tr><td colspan="8" class="empty">No payments recorded.</td></tr>{% endfor %}</tbody></table></div></section>
{% endblock %}

```

## `templates/student_form.html`

```html
{% extends 'base.html' %}{% block content %}
<div class="page-head"><div><h2>{{ 'Edit Student' if student else 'Student Registration' }}</h2><p>All important student and fee data is stored permanently in SQLite.</p></div><a class="btn light" href="{{ url_for('students') }}">Back to Records</a></div>
<form method="post" class="card form-card">
<div class="section-label">Student & Parent Details</div><div class="form-grid">
<div><label>Admission / Reg No</label><input name="admission_no" value="{{ student.admission_no if student else '' }}" placeholder="Auto generated if blank"></div>
<div><label>Student Name *</label><input name="name" value="{{ student.name if student else '' }}" required></div>
<div><label>Date of Birth *</label><input type="date" name="dob" value="{{ student.dob if student else '' }}" required></div>
<div><label>Current Class *</label><select name="current_class" required>{% for c in classes %}<option {% if student and student.current_class==c %}selected{% endif %}>{{ c }}</option>{% endfor %}</select></div>
<div><label>Studied From Class</label><select name="from_class">{% for c in classes %}<option {% if student and student.from_class==c %}selected{% endif %}>{{ c }}</option>{% endfor %}</select></div>
<div><label>Studied To Class</label><select name="to_class">{% for c in classes %}<option {% if student and student.to_class==c %}selected{% endif %}>{{ c }}</option>{% endfor %}</select></div>
<div><label>Section *</label><select name="section" required>{% for s in sections %}<option {% if student and student.section==s %}selected{% endif %}>{{ s }}</option>{% endfor %}</select></div>
<div><label>Father Name</label><input name="father" value="{{ student.father if student else '' }}"></div>
<div><label>Mother Name</label><input name="mother" value="{{ student.mother if student else '' }}"></div>
<div><label>Parent Phone</label><input name="phone" inputmode="tel" value="{{ student.phone if student else '' }}" placeholder="10-digit / +91 number"></div>
<div><label>Nationality</label><input name="nationality" value="{{ student.nationality if student else 'Indian' }}"></div>
<div><label>Place of Birth</label><input name="place_of_birth" value="{{ student.place_of_birth if student else '' }}"></div>
<div><label>Admission Date</label><input type="date" name="admission_date" value="{{ student.admission_date if student else today }}"></div>
<div><label>Village / Residence</label><input name="village" value="{{ student.village if student else '' }}"></div>
<div><label>Religion</label><select id="religion" name="religion" onchange="updateCasteOptions()"><option value="">Select Religion</option>{% for r in caste_data.keys() %}<option {% if student and student.religion==r %}selected{% endif %}>{{ r }}</option>{% endfor %}</select></div>
<div><label>Caste Category</label><select id="caste" name="caste" onchange="updateSubcasteOptions()"><option value="">Select Caste</option></select></div>
<div><label>Subcaste</label><select id="subcaste" name="subcaste"><option value="">Select Subcaste</option></select></div>
</div>
<div class="section-label">Fee Structure</div><div class="form-grid">
<div><label>Total School Fee (₹)</label><input type="number" min="0" step="0.01" name="school_fee" value="{{ student.school_fee if student else 0 }}"></div>
<div><label>School Paid</label><input class="readonly" readonly value="{{ balances.school_paid if balances else 0 }}"></div>
<div><label>School Due</label><input class="readonly" readonly value="{{ balances.school_due if balances else 0 }}"></div>
<div><label>Vehicle Required?</label><select name="vehicle_required"><option {% if not student or student.vehicle_required=='No' %}selected{% endif %}>No</option><option {% if student and student.vehicle_required=='Yes' %}selected{% endif %}>Yes</option></select></div>
<div><label>Total Vehicle Fee (₹)</label><input type="number" min="0" step="0.01" name="vehicle_fee" value="{{ student.vehicle_fee if student else 0 }}"></div>
<div><label>Vehicle Paid</label><input class="readonly" readonly value="{{ balances.vehicle_paid if balances else 0 }}"></div>
<div><label>Vehicle Due</label><input class="readonly" readonly value="{{ balances.vehicle_due if balances else 0 }}"></div>
{% if not student %}<div><label>Initial School Payment (₹)</label><input type="number" min="0" step="0.01" name="initial_school_paid" value="0"></div><div><label>Initial Vehicle Payment (₹)</label><input type="number" min="0" step="0.01" name="initial_vehicle_paid" value="0"></div><div><label>Initial Payment Mode</label><select name="initial_payment_method"><option>Cash</option><option>UPI</option><option>Card</option></select></div>{% endif %}
</div>
<div class="button-row"><button class="btn primary">{{ 'Update Student' if student else 'Save Student' }}</button><a class="btn light" href="{{ url_for('students') }}">Cancel</a></div>
</form>
{% endblock %}
{% block scripts %}<script>window.CASTE_DATA={{ caste_data|tojson }};window.SELECTED_CASTE={{ (student.caste if student else '')|tojson }};window.SELECTED_SUBCASTE={{ (student.subcaste if student else '')|tojson }};document.addEventListener('DOMContentLoaded',()=>{updateCasteOptions();});</script>{% endblock %}

```

## `templates/students.html`

```html
{% extends 'base.html' %}{% block content %}
<div class="page-head"><div><h2>Student Records</h2><p>Permanent SQLite database records. Data is no longer dependent on browser localStorage.</p></div><a class="btn primary" href="{{ url_for('student_new') }}">+ New Student</a></div>
<form class="searchbar" method="get"><input name="q" value="{{ q }}" placeholder="Search name, admission no, phone, father, village, class..."><button class="btn primary">Search</button>{% if q %}<a class="btn light" href="{{ url_for('students') }}">Clear</a>{% endif %}</form>
<div class="card"><div class="table-scroll"><table class="wide"><thead><tr><th>Adm No</th><th>Student</th><th>DOB</th><th>Class</th><th>Sec</th><th>Parent</th><th>Phone</th><th>School Fee</th><th>School Paid</th><th>School Due</th><th>Vehicle</th><th>Vehicle Due</th><th>Actions</th></tr></thead><tbody>
{% for s in students %}<tr><td><b>{{ s.admission_no }}</b></td><td><b>{{ s.name }}</b><small>{{ s.village or '' }}</small></td><td>{{ s.dob|date_in }}</td><td>{{ s.current_class }}</td><td>{{ s.section }}</td><td>{{ s.father or '-' }}<small>{{ s.mother or '-' }}</small></td><td>{{ s.phone or '-' }}</td><td>{{ s.school_fee|money }}</td><td class="paid">{{ s.school_paid|money }}</td><td class="due">{{ s.school_due|money }}</td><td>{{ s.vehicle_required }}</td><td class="due">{{ s.vehicle_due|money }}</td><td class="actions"><a class="mini blue" href="{{ url_for('student_detail',student_id=s.id) }}">Open</a><a class="mini orange" href="{{ url_for('student_edit',student_id=s.id) }}">Edit</a>{% if current_user['role']=='admin' %}<form method="post" action="{{ url_for('student_delete',student_id=s.id) }}" onsubmit="return confirm('Archive this student record?')"><button class="mini red">Archive</button></form>{% endif %}</td></tr>{% else %}<tr><td colspan="13" class="empty">No student records found.</td></tr>{% endfor %}
</tbody></table></div></div>
{% endblock %}

```

## `templates/tc.html`

```html
{% extends 'base.html' %}{% block content %}<div class="print-page"><div class="print-toolbar"><button class="btn primary" onclick="window.print()">🖨 Print Transfer Certificate</button><a class="btn light" href="{{ url_for('student_detail',student_id=student.id) }}">Back</a></div><div class="document certificate"><div class="doc-head"><img src="{{ url_for('static',filename='logo.png') }}"><div><h1>{{ school_name }}</h1><h2>{{ school_location }}</h2><p>School Code: {{ school_code }}</p></div></div><h2 class="certificate-title">TRANSFER CERTIFICATE</h2><div class="doc-meta"><span>TC No: CTS-TC-{{ student.id }}</span><span>Date: {{ today|date_in }}</span></div><table class="doc-table tc-table"><tr><td>1. Name of the Pupil</td><td>{{ student.name }}</td></tr><tr><td>2. Father / Guardian</td><td>{{ student.father or '-' }}</td></tr><tr><td>3. Mother</td><td>{{ student.mother or '-' }}</td></tr><tr><td>4. Nationality, Religion, Caste & Subcaste</td><td>{{ student.nationality }} - {{ student.religion or '-' }} / {{ student.caste or '-' }} / {{ student.subcaste or '-' }}</td></tr><tr><td>5. Place of Birth</td><td>{{ student.place_of_birth or student.village or '-' }}</td></tr><tr><td>6. Date of Birth (Figures)</td><td>{{ student.dob|date_in }}</td></tr><tr><td>7. Date of Birth (Words)</td><td>{{ date_to_words(student.dob) }}</td></tr><tr><td>8. Date of Admission</td><td>{{ student.admission_date|date_in }}</td></tr><tr><td>9. Class in which Studying / Left</td><td>{{ student.current_class }} - Section {{ student.section }}</td></tr><tr><td>10. Medium of Instruction</td><td>English</td></tr><tr><td>11. Reason for Leaving</td><td>Parent's Request</td></tr><tr><td>12. Date of Application for TC</td><td>{{ today|date_in }}</td></tr><tr><td>13. General Conduct & Character</td><td>Good</td></tr></table><div class="signatures"><span>School Seal</span><span>Principal<br><b>({{ principal }})</b></span></div></div></div>{% endblock %}

```

## `templates/users.html`

```html
{% extends 'base.html' %}{% block content %}<div class="page-head"><div><h2>User & Admin Accounts</h2><p>Only administrators can create or deactivate accounts.</p></div></div><div class="grid-2"><section class="card"><h3>Create User</h3><form method="post" class="stack-form"><label>Full Name</label><input name="full_name" required><label>Username</label><input name="username" required><label>Role</label><select name="role"><option value="user">User</option><option value="admin">Admin</option></select><label>Password</label><input type="password" name="password" minlength="6" required><button class="btn primary">Create Account</button></form></section><section class="card"><h3>Existing Accounts</h3><div class="table-scroll"><table><thead><tr><th>Name</th><th>Username</th><th>Role</th><th>Status</th><th></th></tr></thead><tbody>{% for u in users %}<tr><td>{{ u.full_name }}</td><td>{{ u.username }}</td><td>{{ u.role }}</td><td>{{ 'Active' if u.active else 'Inactive' }}</td><td>{% if u.id != current_user.id %}<form method="post" action="{{ url_for('toggle_user',user_id=u.id) }}"><button class="mini {{ 'red' if u.active else 'green' }}">{{ 'Deactivate' if u.active else 'Activate' }}</button></form>{% endif %}</td></tr>{% endfor %}</tbody></table></div></section></div>{% endblock %}

```

## `static/app.js`

```javascript
function updateCasteOptions(){
  const religion=document.getElementById('religion');
  const caste=document.getElementById('caste');
  if(!religion||!caste)return;
  const data=window.CASTE_DATA||{};
  const selected=window.SELECTED_CASTE||'';
  caste.innerHTML='<option value="">Select Caste</option>';
  Object.keys(data[religion.value]||{}).forEach(v=>{const o=document.createElement('option');o.value=v;o.textContent=v;if(v===selected)o.selected=true;caste.appendChild(o);});
  updateSubcasteOptions();
}
function updateSubcasteOptions(){
  const religion=document.getElementById('religion'), caste=document.getElementById('caste'), sub=document.getElementById('subcaste');
  if(!religion||!caste||!sub)return;
  const selected=window.SELECTED_SUBCASTE||'';
  sub.innerHTML='<option value="">Select Subcaste</option>';
  ((window.CASTE_DATA||{})[religion.value]||{})[caste.value]?.forEach(v=>{const o=document.createElement('option');o.value=v;o.textContent=v;if(v===selected)o.selected=true;sub.appendChild(o);});
  window.SELECTED_SUBCASTE='';
}
function autoHideFlashes(){setTimeout(()=>document.querySelectorAll('.flash').forEach(x=>x.style.opacity='0'),6000);}
document.addEventListener('DOMContentLoaded',autoHideFlashes);

```

## `static/style.css`

```css
:root{--navy:#17365d;--blue:#2874a6;--gold:#d9a441;--bg:#f3f6fa;--green:#198754;--red:#c0392b;--orange:#e67e22;--purple:#7d3c98;--text:#1f2937;--border:#d9e0e7}*{box-sizing:border-box}body{margin:0;font-family:Arial,Helvetica,sans-serif;background:var(--bg);color:var(--text)}a{text-decoration:none;color:var(--blue)}button,input,select{font:inherit}.topbar{background:linear-gradient(135deg,var(--navy),var(--blue));color:#fff;padding:14px 4%;display:flex;justify-content:space-between;gap:20px;align-items:center}.brand{display:flex;align-items:center;gap:14px}.brand img{width:70px;height:70px;object-fit:contain;border-radius:12px;background:#fff}.brand h1{font-size:24px;margin:0 0 4px}.brand p{margin:0;font-size:13px;opacity:.9}.userbox{display:flex;align-items:center;gap:10px;font-size:13px}.userbox span{background:rgba(255,255,255,.15);padding:5px 8px;border-radius:20px}.userbox a{color:#fff;border:1px solid rgba(255,255,255,.5);padding:7px 10px;border-radius:7px}.nav{background:#fff;border-bottom:1px solid var(--border);display:flex;gap:5px;padding:8px 4%;position:sticky;top:0;z-index:50;box-shadow:0 2px 8px #0000000b}.nav a{padding:10px 13px;border-radius:7px;color:var(--navy);font-weight:700;font-size:14px}.nav a:hover{background:#eef5fb}.page{width:94%;max-width:1550px;margin:22px auto;min-height:70vh}.page-head{display:flex;justify-content:space-between;align-items:center;gap:15px;margin-bottom:18px}.page-head h2{margin:0;color:var(--navy)}.page-head p{margin:6px 0 0;color:#667085}.card{background:#fff;border:1px solid #e6ebf0;border-radius:14px;padding:20px;margin-bottom:18px;box-shadow:0 5px 18px #17365d0b}.stats{display:grid;grid-template-columns:repeat(6,1fr);gap:14px;margin-bottom:18px}.stat{background:#fff;border-left:5px solid var(--blue);padding:18px;border-radius:12px;box-shadow:0 4px 15px #00000008}.stat span{display:block;color:#667085;font-size:13px}.stat strong{display:block;margin-top:8px;font-size:23px;color:var(--navy)}.danger-card{border-left-color:var(--red)}.grid-2{display:grid;grid-template-columns:1fr 1fr;gap:18px}.card-head{display:flex;justify-content:space-between;align-items:center;gap:10px;margin-bottom:14px}.card-head h3,.card h3{margin:0;color:var(--navy)}.quick-grid{display:grid;grid-template-columns:repeat(2,1fr);gap:12px}.quick{display:flex;flex-direction:column;gap:5px;padding:16px;border:1px solid var(--border);border-radius:10px;color:var(--navy);background:#fbfcfe}.quick:hover{border-color:var(--blue);transform:translateY(-1px)}.quick span{font-size:12px;color:#667085}.btn{display:inline-block;border:0;border-radius:8px;padding:10px 14px;font-weight:700;cursor:pointer;text-align:center}.primary{background:var(--navy);color:#fff}.blue{background:var(--blue);color:#fff}.green{background:var(--green);color:#fff}.orange{background:var(--orange);color:#fff}.purple{background:var(--purple);color:#fff}.light{background:#eef2f6;color:var(--navy)}.red{background:var(--red);color:#fff}.wide{width:100%}.button-row{display:flex;flex-wrap:wrap;gap:9px;margin-top:15px}.searchbar,.filterbar{display:flex;gap:10px;align-items:end;flex-wrap:wrap;background:#fff;padding:14px;border-radius:12px;border:1px solid var(--border);margin-bottom:18px}.searchbar input{flex:1;min-width:250px}.filterbar>div{min-width:190px}.filterbar button{margin-top:0}.form-card{padding:25px}.section-label{font-size:16px;font-weight:800;color:var(--blue);padding:11px 0;border-bottom:2px solid #e8edf2;margin:4px 0 16px}.form-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:15px}.form-grid>div,.payment-form>div,.stack-form{display:flex;flex-direction:column;gap:6px}.form-grid label,.payment-form label,.stack-form label,.filterbar label{font-weight:700;color:#475467;font-size:13px}input,select{width:100%;padding:11px 12px;border:1px solid #cbd5e1;border-radius:8px;background:#fff;outline:none}input:focus,select:focus{border-color:var(--blue);box-shadow:0 0 0 3px #2874a61a}.readonly{background:#f2f4f7!important;font-weight:700}.table-scroll{overflow:auto}.table-scroll table{width:100%;border-collapse:collapse;min-width:850px}.table-scroll th{background:var(--navy);color:#fff;text-align:left;padding:11px;white-space:nowrap}.table-scroll td{padding:10px;border-bottom:1px solid #edf0f3;white-space:nowrap;vertical-align:top}.table-scroll tr:hover{background:#f8fafc}.table-scroll small,.actions small{display:block;color:#667085;font-size:11px;margin-top:3px}.paid{color:#16803c;font-weight:700}.due{color:#c0392b;font-weight:700}.actions{display:flex;gap:5px;flex-wrap:wrap}.mini{display:inline-block;border:0;border-radius:6px;padding:6px 8px;font-size:11px;font-weight:700;cursor:pointer}.badge{padding:6px 10px;border-radius:20px;background:#eef5fb;color:var(--blue);font-weight:700}.profile-grid{display:grid;grid-template-columns:1.5fr 1fr;gap:18px}.detail-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:1px;background:#e7ebef;margin-top:14px}.detail-grid>div{background:#fff;padding:11px}.detail-grid span{display:block;font-size:11px;color:#667085}.detail-grid b{display:block;margin-top:4px}.fee-summary{display:grid;grid-template-columns:1fr 1fr;gap:10px;margin-top:14px}.fee-summary>div{padding:13px;background:#f8fafc;border-radius:8px}.fee-summary span{display:block;font-size:12px;color:#667085}.fee-summary b{display:block;margin-top:5px}.fee-summary .total{grid-column:1/-1;border:1px solid #f1d2d2}.payment-form{display:grid;grid-template-columns:repeat(4,1fr);gap:12px;align-items:end}.payment-form button{height:43px}.hint{font-size:12px;color:#667085;margin-top:10px}.empty{text-align:center;color:#667085;padding:25px!important}.flash-wrap{position:relative}.flash{padding:12px 15px;border-radius:8px;margin-bottom:12px;background:#eaf6ee;color:#176b38;border:1px solid #bde2c8;transition:opacity .5s}.flash.danger{background:#fdecec;color:#9f2620;border-color:#f1b8b4}.login-page{min-height:100vh;display:grid;place-items:center;background:linear-gradient(135deg,#17365d,#2874a6);padding:20px}.login-card{background:#fff;width:100%;max-width:430px;border-radius:18px;padding:30px;box-shadow:0 20px 60px #0004;text-align:center}.login-logo{width:140px;height:140px;object-fit:contain;margin:auto}.login-card h1{font-size:22px;color:var(--navy);margin:8px 0 4px}.login-card>p{color:#667085;margin:0 0 22px}.login-card form{text-align:left;display:grid;gap:8px}.login-card input{margin-bottom:8px}.login-note{margin-top:18px;background:#fff8e8;border:1px solid #f1d58c;padding:10px;border-radius:8px;font-size:12px;color:#6b5310}.document{width:850px;max-width:100%;margin:auto;background:#fff;border:3px double var(--navy);padding:30px;box-shadow:0 5px 20px #0001}.print-page{background:#eef2f6;padding:20px}.print-toolbar{width:850px;max-width:100%;margin:0 auto 12px;display:flex;gap:8px}.doc-head{display:flex;align-items:center;gap:15px;border-bottom:2px solid var(--navy);padding-bottom:12px}.doc-head img{width:90px;height:90px;object-fit:contain}.doc-head h1{margin:0;color:var(--navy);font-size:25px}.doc-head h2{margin:4px 0;font-size:16px}.doc-head p{margin:0;font-size:12px}.doc-meta{display:flex;justify-content:space-between;gap:10px;padding:14px 0;font-size:13px}.receipt-info{display:grid;grid-template-columns:1fr 1fr;gap:7px;font-size:13px;margin-bottom:15px}.doc-table{width:100%;border-collapse:collapse;font-size:12px}.doc-table th,.doc-table td{border:1px solid #333;padding:8px;text-align:left}.doc-table th{background:#edf2f7}.doc-table .total{font-weight:700}.words{margin-top:18px;font-size:13px}.signatures{display:flex;justify-content:space-between;margin-top:65px;font-weight:700;text-align:center}.certificate-title{text-align:center;margin:20px 0;color:var(--navy);text-decoration:underline}.certificate-text{line-height:1.9;text-align:justify;font-size:14px}.tc-table td:first-child{width:42%;font-weight:700}.id-card{width:430px;min-height:270px;margin:40px auto;background:#fff;border:2px solid var(--navy);border-radius:15px;overflow:hidden;box-shadow:0 8px 25px #0002}.id-head{display:flex;align-items:center;gap:10px;background:linear-gradient(135deg,var(--navy),var(--blue));color:#fff;padding:10px}.id-head img{width:48px;height:48px;object-fit:contain;background:#fff;border-radius:7px}.id-head b{display:block;font-size:15px}.id-head small{display:block}.id-body{display:grid;grid-template-columns:85px 1fr 75px;gap:10px;padding:15px;align-items:center}.id-photo{height:105px;border:1px dashed #98a2b3;display:grid;place-items:center;text-align:center;font-size:10px;color:#667085}.id-details h2{font-size:16px;margin:0 0 8px;color:var(--navy)}.id-details p{font-size:11px;margin:4px 0}.qr{width:70px;height:70px}.id-foot{text-align:center;background:#eef2f6;padding:8px;font-size:10px;font-weight:700}.stack-form{gap:8px}.stack-form input,.stack-form select{margin-bottom:5px}footer{background:var(--navy);color:#fff;text-align:center;padding:18px;margin-top:30px;font-size:12px}footer span{opacity:.75}
@media(max-width:1100px){.stats{grid-template-columns:repeat(3,1fr)}.form-grid{grid-template-columns:repeat(2,1fr)}.profile-grid,.grid-2{grid-template-columns:1fr}.payment-form{grid-template-columns:repeat(2,1fr)}}
@media(max-width:700px){.topbar{align-items:flex-start}.brand h1{font-size:18px}.brand img{width:55px;height:55px}.userbox{display:none}.nav{overflow:auto;white-space:nowrap}.stats{grid-template-columns:1fr 1fr}.form-grid,.detail-grid,.receipt-info,.payment-form{grid-template-columns:1fr}.page-head{align-items:flex-start;flex-direction:column}.quick-grid{grid-template-columns:1fr}.document{padding:16px}.doc-head img{width:65px;height:65px}.doc-head h1{font-size:18px}.id-card{width:100%}}
@media print{body{background:#fff}.topbar,.nav,footer,.flash-wrap,.print-toolbar{display:none!important}.page{width:100%;margin:0}.print-page{padding:0;background:#fff}.document{width:100%;box-shadow:none;border:2px solid #000}.id-card{box-shadow:none;margin:0 auto;page-break-inside:avoid}.card,.stats{box-shadow:none}.btn{display:none!important}}

```
