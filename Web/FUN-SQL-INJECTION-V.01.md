## FUN-SQL-INJECTION-V.01
---

+ app.py

```python
from flask import Flask, render_template, request, render_template, redirect, url_for, make_response
from filter import sql_filter
from dataCheck import createSession, check_login, check_solve, getSessionData
from encode import encoder
import json
import pandas as pd
import os
import sqlite3

app = Flask(__name__)

app.secret_key = os.urandom(32)

FLAG = "[**DELETED**]"
PASSWORD = "[**DELETED**]"

def authTBL_insert(sql):
    try:
        conn = sqlite3.connect("auth.db")
        conn.execute(sql)
        conn.commit()
        conn.close()
        return True
    except Exception as e:
        print(e)
        return False

def authTBL_select(sql):
    try:
        conn = sqlite3.connect("auth.db")
        rows = pd.read_sql_query(sql, conn)
        return dict(rows)
    except Exception as e:
        print(e)
        return None
        
def secret_check(rows):
    for i in range(10):
        rows["comment"][i] = "access denied!" if rows["user_group"][i] == 'vip' or rows["user_group"][i] == 'admin' else rows["comment"][i]
    return rows

def table_create():
    sql = "CREATE TABLE IF NOT EXISTS authTBL (id INT, username STRING, password STRING, user_group STRING, comment STRING)"
    sql2 = "CREATE TABLE IF NOT EXISTS adminTBL (admin_name STRING, admin_group STRING, secret_key STRING)"
    authTBL_insert(sql)
    authTBL_insert(sql2)

def first_insert():
    sql = f"""INSERT INTO authTBL (id, username, password, user_group, comment) VALUES 
        (1, 'admin', '{PASSWORD}', 'admin', 'hello world'),
        [**DELETED**]
    """
    sql2 = "INSERT INTO adminTBL (admin_name, admin_group, secret_key) VALUES ('super_admin', 'super_admin', '[**DELETED**]')"
    authTBL_insert(sql)
    authTBL_insert(sql2)

@app.route("/")
def index():
    message = "hello you login ok! you can access /admin page!" if check_login(request.cookies.get("session", "")) == True else "you are not login or not admin"
    return render_template("index.html", message=message)

@app.route("/auth", methods=["GET", "POST"])
def login():
    if request.method == "GET":
        return render_template("auth.html")
    elif request.method == "POST":
        password = request.form.get("password", "").strip()
        filter_pw = sql_filter(password)
        if filter_pw:
            return render_template("auth.html", message="this is sql filter!")
        row = authTBL_select(f"SELECT username FROM authTBL WHERE password='{encoder(password)}'")
        try:
            if row and row["username"][0] == 'admin':
                resp = redirect(url_for("index"))
                resp.set_cookie('session', createSession(row["username"][0]))
                return resp
            else:
                return render_template("auth.html", message="login fail or you are not admin")
        except:
            return render_template("auth.html", message="error")

@app.route("/admin", methods=["GET", "POST"])
def admin():
    if check_login(request.cookies.get("session", "")):
        rows = authTBL_select("SELECT id, username, user_group, comment FROM authTBL")
        return render_template("admin.html", flag=FLAG, data=rows if getSessionData(None, request.cookies.get("admin", "")) else secret_check(rows))
    else:
        return "<script>alert('access denied!'); location.href='/'</script>"

@app.route("/admin/singup", methods=["GET", "POST"])
def singup():
    if check_login(request.cookies.get("session", "")):
        if request.method == "GET":
            return render_template("singup.html")
        elif request.method == "POST":
            admin_name = request.form.get("name", "")
            if sql_filter(admin_name) == True:
                return render_template("singup.html", message="sql injection filter!")
            check_id = authTBL_select(f"SELECT admin_name FROM adminTBL WHERE admin_name='{getSessionData(request.cookies.get('session', ''))}'")
            if check_id["admin_name"].empty == False:
                return render_template("singup.html", message="already singup!")
            else:
                try:
                    isResult = authTBL_insert(f"INSERT INTO adminTBL (admin_name, admin_group, secret_key) VALUES ('{admin_name}', 'admin', '{os.urandom(16).hex()}')")
                    if isResult:
                        return "<script>alert('singup success!'); location.href='/'</script>"
                    else:
                        return render_template("singup.html", message="fail singup!")
                except:
                    return render_template("singup.html", message="fail singup!")
    else:
        return "<script>alert('access denied!'); location.href='/'</script>"

@app.route("/admin/auth", methods=["GET", "POST"])
def auth():
    if check_login(request.cookies.get("session", "")):
        if request.method == "GET":
            return render_template("admin_auth.html")
        elif request.method == "POST":
            key = request.form.get("key", "")
            row = authTBL_select(f"SELECT admin_group, secret_key FROM adminTBL WHERE admin_name='{getSessionData(request.cookies.get('session', ''))}'")
            try:
                if str(row["secret_key"][0]) == str(key):
                    resp = redirect(url_for("index"))
                    resp.set_cookie("admin", createSession(row["admin_group"][0], True))
                    return resp
                return render_template("admin_auth.html", message="invaild auth key.")
            except:
                return render_template("admin_auth.html", message="error")
    else:
        return "<script>alert('access denied!'); location.href='/'</script>"

@app.route("/fix_comment", methods=["POST"])
def fix_comment():
    if check_login(request.cookies.get("session", "")):
        data = dict(request.form)
        if sql_filter(data['new_comment']) == True:
            return json.dumps({"status": 403, "message": "sql injection filter!"})
        if getSessionData(None, request.cookies.get("admin", "")) == "super_admin":
            isUpdate = authTBL_insert(f"UPDATE authTBL SET comment='{data['new_comment']}' WHERE username='FLAG'")
            if isUpdate:
                return json.dumps({"status": 200, "message": check_solve(None, 3)})
            return json.dumps({"status": 500, "message": "error"})
        return json.dumps({"status": 403, "message": "you are not super admin"})
    else:
        return json.dumps({"status": 403, "message": "access denied!"})

if __name__ == "__main__":
    table_create()
    first_insert()
    app.run(host='0.0.0.0', port=8000)
```

<br>

+ encode.py

```python
import hashlib

def encoder(password):
    return hashlib.md5(password.encode("utf-8")).digest()
```

<br><br>

## Solution
---

