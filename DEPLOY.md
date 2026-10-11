# Deploy your receiving page

**Deploy the same `index.html` that worked in the [quick start](QUICKSTART.md).**
Your page uses QRKeyboard's hosted SDK, phone page, and relay. Publishing this
page does not require deploying those services or handling Cloudflare Worker
credentials.

## Already have a website?

Add the HTML to your existing app and publish it through your usual process.
Use HTTPS. Keep the script URL `https://qrkeyboard.com/v1.js`; it tells the SDK
where the hosted API and phone page live. Put your input handler after the script
and after the QR/output elements, as in the starter.

If your app has a Content Security Policy, allow scripts and HTTP requests to
`https://qrkeyboard.com` and WebSocket connections to `wss://qrkeyboard.com`.
The inline starter script also needs your CSP's nonce/hash, or move it into an
allowed JavaScript file. No origin registration is needed; the API supports
cross-origin browser integration.

For production you can pin an exact SDK build with Subresource Integrity instead
of the always-latest `/v1.js`; see
[Pin a version](DEVELOPER_GUIDE.md#pin-a-version-for-production).

## Need a host? Upload a folder

Put only your public app files in a folder named `site`, with `index.html` at its
root. In Cloudflare's dashboard, open **Workers & Pages**, choose **Create
application → Get started → Drag and drop your files**, name the project, upload
`site`, and select **Deploy site**. Open the resulting HTTPS `pages.dev` address.
This starter needs no build. A Cloudflare account is required for hosting there.
[Cloudflare Direct Upload](https://developers.cloudflare.com/pages/get-started/direct-upload/).

## Prefer the terminal?

With Node/npm and a Cloudflare account, run beside the working `index.html`:

```bash
mkdir -p site
cp index.html site/index.html
npx wrangler login
npx wrangler pages deploy site
```

Follow the project-name and branch prompts and open the printed HTTPS URL.
Edit `site/index.html` and repeat the deploy command. If you edit the original
file instead, copy it into `site` again before deploying. Upload only your app folder,
not this repository or any credentials. Direct Upload projects cannot later be
switched to Git integration; use a separate Git-integrated project for automatic
publishing. [Cloudflare Direct Upload](https://developers.cloudflare.com/pages/get-started/direct-upload/).

## Verify the published page

1. Open its HTTPS URL on the receiving computer or display.
2. Confirm that a fresh QR appears.
3. Scan it with your phone and check the destination name.
4. Connect and type `hello screen`; check that text and Backspace work.
5. On the updated release, test Finish and confirm old visitor text is cleared
   before the next QR appears.

Your deployed page still uses the currently deployed QRKeyboard service. Publishing
an HTML file does not enable service features listed as pending in
[release status](RELEASE_STATUS.md). Ordinary integrations need no environment
variables or API secrets.

## Troubleshooting

| What you see | First thing to check |
|---|---|
| No QR; `QRKeyboard is not defined` | `v1.js` failed to load, or loaded after your setup script; check internet/CSP/ad blockers and script order |
| Crypto/security error | Open the page over HTTPS, or locally at `http://127.0.0.1`; avoid a plain-HTTP LAN IP |
| Phone cannot open the computer's localhost | Scan the generated QR; do not type the local development URL into the phone |
| Connected but no text on screen | Live mode needs a `key` handler; message mode needs a `text` handler |
| Named actions or message mode do nothing | Check whether the updated hosted service has been released |
| Phone says the screen is in use | Disconnect or Finish on the existing phone. A phone that dropped keeps its place for 5 minutes; `finish()` on the screen clears it at once |
| Creating sessions is rate-limited | Wait a minute; avoid repeatedly reloading or creating a keyboard on every render |
| Finish leaves private data visible | Add a `finish` handler that clears application-owned data; await asynchronous cleanup |

For a framework app, initialise the keyboard once when the component mounts and
call `keyboard.close()` when it unmounts. `close()` ends the session for good and
tells the phone to scan again; `keyboard.finish()` hands the screen to the next
visitor with a fresh QR. Updated lifecycle methods require the new release.
