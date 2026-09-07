
# This is the code for the app with the SQLi vulernability

```python
from flask import Flask, request, render_template
import pymysql

app = Flask(__name__)

def get_db():
    return pymysql.connect(
        host='localhost',
        user='flaskuser',
        password='FlaskPass123!',
        database='homelab'
    )

@app.route('/', methods=['GET'])
def home():
    return render_template('login.html', result=None)

@app.route('/login', methods=['POST'])
def login():
    username = request.form['username']
    password = request.form['password']

    conn = get_db()
    cursor = conn.cursor()

    # Intentionally vulnerable: user input concatenated directly into the query
    query = f"SELECT hostname, username, password, role, notes FROM credentials WHERE username='{username}' AND password='{password}'"
    cursor.execute(query)
    rows = cursor.fetchall()

    conn.close()

    return render_template('login.html', result=rows)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```