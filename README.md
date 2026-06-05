# Camera Bridge for Google Apps Script

A lightweight cross-origin camera bridge for Google Apps Script apps.

## The Problem

Google Apps Script HtmlService runs the frontend inside a sandboxed iframe.
On modern deployments, Google does not allow direct camera access inside this iframe.

Because of this, calling:

navigator.mediaDevices.getUserMedia({ video: true })

inside an Apps Script frontend can fail with:

Permissions policy violation: camera is not allowed in this document

Since Google controls the iframe permissions, this cannot be fixed from inside the Apps Script HTML code.

## The Solution

This bridge opens a small external popup window hosted on GitHub Pages.

The popup is outside the Apps Script iframe, so it can request normal browser camera access. After the user takes a photo, the image is converted to a Base64 Data URL and sent back to the main Apps Script window using postMessage.

Google Apps Script App
      |
      | 1. window.open()
      v
GitHub Pages Camera Bridge
      |
      | 2. Camera opens and photo is captured
      v
Base64 image
      |
      | 3. window.opener.postMessage(photoData)
      v
Google Apps Script App

## How It Works

### 1. Bridge Page

The external index.html starts the camera with:

navigator.mediaDevices.getUserMedia({
  video: true,
  audio: false
});

After the photo is taken, it sends the image back:

window.opener.postMessage({
  type: "CAMERA_CAPTURE_SUCCESS",
  base64DataUrl: base64DataUrl
}, "*");

window.close();

### 2. Apps Script Integration

Open the camera bridge popup:

let capturedPhotoBase64 = "";

function startCamera() {
  const bridgeUrl = "https://your-user-name.github.io/GAS-camera-bridge/";

  window.open(
    bridgeUrl,
    "cameraWindow",
    "width=430,height=650,status=no,menubar=no,toolbar=no"
  );
}

Listen for the captured photo:

window.addEventListener("message", function(event) {
  if (event.origin !== "https://your-user-name.github.io") return;

  if (event.data && event.data.type === "CAMERA_CAPTURE_SUCCESS") {
    capturedPhotoBase64 = event.data.base64DataUrl;
    console.log("Photo captured:", capturedPhotoBase64);
  }
});

Use the photo in Apps Script:

if (!capturedPhotoBase64) {
  alert("Please take a photo first.");
  return;
}

const photoBase64 = capturedPhotoBase64;
const cleanBase64 = photoBase64.split(",")[1];

## Short README Version

# Camera Bridge for Google Apps Script

Google Apps Script HtmlService runs inside a sandboxed iframe.
Because Google does not allow camera access in this iframe, direct calls to:

navigator.mediaDevices.getUserMedia({ video: true })

can fail with:

Permissions policy violation: camera is not allowed in this document

## Workaround

This project opens a small external popup hosted on GitHub Pages.
The popup runs outside the Apps Script iframe, requests native camera access, captures a photo, converts it to Base64, sends it back with postMessage, and closes itself.

Apps Script UI -> window.open() -> Camera Bridge -> Capture Photo -> postMessage() -> Apps Script UI

## Apps Script Example

let capturedPhotoBase64 = "";

function startCamera() {
  const bridgeUrl = "https://your-user-name.github.io/GAS-camera-bridge/";

  window.open(
    bridgeUrl,
    "cameraWindow",
    "width=430,height=650,status=no,menubar=no,toolbar=no"
  );
}

window.addEventListener("message", function(event) {
  if (event.origin !== "https://your-user-name.github.io") return;

  if (event.data && event.data.type === "CAMERA_CAPTURE_SUCCESS") {
    capturedPhotoBase64 = event.data.base64DataUrl;
  }
});

## Security Advantages

- No API keys: The bridge does not need external credentials or server tokens.
- Origin check: The Apps Script app only accepts messages from your GitHub Pages domain.
- Temporary data: The photo is handled only in browser memory and sent directly back to the Apps Script window.
- No server storage: The bridge does not save or upload images anywhere.
