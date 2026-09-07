
### This is the frontend for the vulernable app:

```
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Homelab Portal Login</title>
<style>
    * { box-sizing: border-box; }
    body {
        margin: 0;
        font-family: 'Segoe UI', Arial, sans-serif;
        background: #1e2530;
        color: #e6e9ef;
        display: flex;
        justify-content: center;
        align-items: center;
        min-height: 100vh;
    }
    .card {
        background: #262e3d;
        padding: 32px 36px;
        border-radius: 10px;
        box-shadow: 0 8px 24px rgba(0,0,0,0.35);
        width: 340px;
    }
    h1 {
        font-size: 20px;
        margin: 0 0 4px 0;
        color: #ffffff;
    }
    p.sub {
        margin: 0 0 20px 0;
        font-size: 13px;
        color: #9aa4b2;
    }
    label {
        display: block;
        font-size: 13px;
        margin-bottom: 6px;
        color: #c3c9d4;
    }
    input {
        width: 100%;
        padding: 10px 12px;
        margin-bottom: 16px;
        border: 1px solid #3a4456;
        border-radius: 6px;
        background: #1e2530;
        color: #e6e9ef;
        font-size: 14px;
    }
    input:focus {
        outline: none;
        border-color: #5a8dee;
    }
    button {
        width: 100%;
        padding: 11px;
        background: #5a8dee;
        border: none;
        border-radius: 6px;
        color: white;
        font-size: 14px;
        font-weight: 600;
        cursor: pointer;
    }
    button:hover {
        background: #4a7ddb;
    }
    .results {
        margin-top: 24px;
        border-top: 1px solid #3a4456;
        padding-top: 16px;
    }
    .results h2 {
        font-size: 14px;
        color: #9aa4b2;
        margin: 0 0 10px 0;
    }
    table {
        width: 100%;
        border-collapse: collapse;
        font-size: 12px;
    }
    th, td {
        text-align: left;
        padding: 6px 8px;
        border-bottom: 1px solid #3a4456;
    }
    th {
        color: #9aa4b2;
        font-weight: 600;
    }
    .no-result {
        font-size: 13px;
        color: #e06666;
        margin-top: 16px;
    }
</style>
</head>
<body>
    <div class="card">
        <h1>Homelab Portal</h1>
        <p class="sub">Sign in to continue</p>
        <form action="/login" method="POST">
            <label for="username">Username</label>
            <input type="text" id="username" name="username" autocomplete="off">
            <label for="password">Password</label>
            <input type="password" id="password" name="password">
            <button type="submit">Log In</button>
        </form>

        {% if result is not none %}
            {% if result|length > 0 %}
            <div class="results">
                <h2>Login Result</h2>
                <table>
                    <tr><th>Hostname</th><th>Username</th><th>Password</th><th>Role</th><th>Notes</th></tr>
                    {% for row in result %}
                    <tr>
                        <td>{{ row[0] }}</td>
                        <td>{{ row[1] }}</td>
                        <td>{{ row[2] }}</td>
                        <td>{{ row[3] }}</td>
                        <td>{{ row[4] }}</td>
                    </tr>
                    {% endfor %}
                </table>
            </div>
            {% else %}
            <p class="no-result">Invalid credentials.</p>
            {% endif %}
        {% endif %}
    </div>
</body>
</html>

```