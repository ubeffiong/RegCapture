# Relay: fingerprints for remote-desktop, Citrix and cloud-only websites

Normally the website talks to the app on the **same PC as the browser** (`127.0.0.1`). That stops working when the browser
runs somewhere else than the reader: Citrix / RDP / VDI sessions, or a kiosk where the browser is a thin client. For those
cases the app can connect **out** to a small relay, and the website reaches the reader through it. Nothing is opened
inbound on the workstation.

```
 browser (any stack) --wss--> relay <--wss-- desktop app (outbound only) --> reader
        ^ ticket from your backend
```

You need the relay **only** for those cases. Plain "browser and reader on the same PC" needs no relay and no server.

## Deploy (no containers needed)

The relay is one small program. **Download `RegCapture-Relay-win-x64.zip` from the [latest release](https://github.com/ubeffiong/RegCapture/releases/latest)**: it is self-contained (nothing else to install). Unzip it on the server and run `RegCapture.Relay.exe`. (To build it from source instead:)

```
dotnet publish GenericFingerPrint.Relay -c Release -r win-x64 --self-contained -o relay     # copy the folder to the server, run relay\RegCapture.Relay.exe
```

Run it as a Windows service (`sc create RegCaptureRelay binPath= "C:\relay\RegCapture.Relay.exe"`), or behind IIS /
nginx / Caddy for TLS. Always use `wss://` across networks (terminate TLS in your normal reverse proxy); plain `ws://` is
acceptable only inside a trusted facility network.

Configure `appsettings.json` (or environment variables such as `Relay__TicketSecret`):

```json
{
  "Relay": {
    "TicketSecret": "at-least-32-random-characters",
    "AllowedOrigins": [ "https://hmis.example.org" ],
    "Agents": { "ward3-pc2": "<sha256 hex of that workstation's secret>" }
  }
}
```

Helper commands (they do not start the server):

```
RegCapture.Relay.exe new-agent ward3-pc2     # prints the workstation secret ONCE + the hash to paste above
RegCapture.Relay.exe hash-secret <secret>
RegCapture.Relay.exe ticket ward3-pc2 https://hmis.example.org nurse42   # test ticket (needs Relay__TicketSecret)
```

`--urls http://0.0.0.0:8080` changes the listening address (default 8080).

## Configure the workstation app

In the app's `settings.json` (`Relay` section): `Enabled: true`, `Url: "wss://relay.example.org/ws/agent"`,
`AgentId`, `AgentSecret`. The app reconnects automatically if the link drops.

## Website side (any stack)

1. **Your backend issues a ticket** for the logged-in user and the workstation they sit at. A ticket is a standard
   HS256 JWT signed with `TicketSecret`: `{"aud":"fingerprint-relay","agent":"ward3-pc2","exp":<now+5min>}` (optional
   `origin`, `sub`). Examples:

   ```js
   // Node
   const jwt = require('jsonwebtoken');
   const ticket = jwt.sign({ aud: 'fingerprint-relay', agent }, SECRET, { algorithm: 'HS256', expiresIn: '5m' });
   ```
   ```python
   # Python
   import jwt, time
   ticket = jwt.encode({"aud": "fingerprint-relay", "agent": agent, "exp": int(time.time()) + 300}, SECRET, algorithm="HS256")
   ```
   ```php
   // PHP (firebase/php-jwt)
   $ticket = Firebase\JWT\JWT::encode(['aud'=>'fingerprint-relay','agent'=>$agent,'exp'=>time()+300], $secret, 'HS256');
   ```
   ```csharp
   // C# (System.IdentityModel.Tokens.Jwt)
   var t = new JwtSecurityToken(audience: "fingerprint-relay", claims: new[]{ new Claim("agent", agent) },
       expires: DateTime.UtcNow.AddMinutes(5), signingCredentials: new SigningCredentials(new SymmetricSecurityKey(key), "HS256"));
   var ticket = new JwtSecurityTokenHandler().WriteToken(t);
   ```
   ```java
   // Java (jjwt)
   String ticket = Jwts.builder().audience().add("fingerprint-relay").and().claim("agent", agent)
       .expiration(new Date(System.currentTimeMillis() + 300_000)).signWith(Keys.hmacShaKeyFor(key)).compact();
   ```
   ```ruby
   # Ruby
   ticket = JWT.encode({ aud: 'fingerprint-relay', agent: agent, exp: Time.now.to_i + 300 }, SECRET, 'HS256')
   ```
   ```go
   // Go (golang-jwt)
   t, _ := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{"aud":"fingerprint-relay","agent":agent,"exp":time.Now().Add(5*time.Minute).Unix()}).SignedString(secret)
   ```
   Expose it as an authenticated endpoint, e.g. `GET /fingerprint/ticket` returning `{"ticket":"..."}`. The relay caps ticket
   lifetime at one hour, ends the connection when the ticket expires, and only ever serves the agent named in the ticket.

2. **The page uses the SDK** in `relay` mode (or `auto` to prefer the local app and fall back):

   ```html
   <script src="fingerprint-client.js"></script><script src="fingerprint-capture.js"></script>
   <form method="post" action="/patients/42/fingerprint">
     <fingerprint-capture transport="relay" relay-url="wss://relay.example.org/ws/browser" ticket-url="/fingerprint/ticket" name="template">
     </fingerprint-capture>
   </form>
   ```
   or in code: `new FingerprintClient({ transport: 'relay', relay: { url, ticket: () => fetch('/fingerprint/ticket').then(r => r.json()).then(j => j.ticket) } })`.

## What the relay enforces

- Browsers: `Origin` must be in `AllowedOrigins`; valid, unexpired HS256 ticket (never `alg:none`); optional `origin` claim must match.
- Workstations: agent id + secret (stored only as SHA-256 hashes); a newer connection replaces an older one.
- Only the app's own routes are forwarded (`GET status|devices|hello`, `POST` capture/validate/cancel), message size and in-flight requests per browser are capped, a closed tab cancels its scan on the workstation, unanswered requests time out.
- The relay stores nothing and never logs fingerprint data; it still carries the template in transit, so use `wss://`.

## What was verified

Automated tests cover the ticket validator, relay authentication/authorization, forwarding, cancel, disconnect-cancels-scan,
offline agent, agent isolation, size limit, and the real JavaScript SDK capturing and aborting through a live relay and agent
(with a fake reader). **Not verified:** a real hardware scan through the relay, behaviour behind a particular reverse proxy,
the `<fingerprint-capture>` element inside a real browser page, and long-running soak/load.
