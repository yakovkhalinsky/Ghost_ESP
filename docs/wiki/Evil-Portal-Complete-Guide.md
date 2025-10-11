# Evil Portal Complete Guide

## Table of Contents
1. [Overview](#overview)
2. [How Evil Portal Works](#how-evil-portal-works)
3. [Architecture](#architecture)
4. [Commands Reference](#commands-reference)
5. [Creating Portal Files](#creating-portal-files)
6. [Templates and Examples](#templates-and-examples)
7. [File Structure](#file-structure)
8. [Data Capture](#data-capture)
9. [Troubleshooting](#troubleshooting)
10. [Best Practices](#best-practices)

## Overview

Evil Portal is a captive portal system that creates fake WiFi hotspots with custom login pages. When users connect to the hotspot, they are redirected to a login page that can capture credentials and keystrokes. This is designed for educational and authorized penetration testing purposes only.

### Key Features
- **Custom HTML Portals**: Create your own login pages
- **Credential Capture**: Automatically captures form submissions
- **Keystroke Logging**: Records all keystrokes and input changes
- **Multiple File Support**: Serve HTML, CSS, JS, PNG, JPG files
- **SD Card Integration**: Store portal files and captured data
- **UART Upload**: Upload HTML directly via serial connection
- **Built-in Portal**: Default Google-style login page

## How Evil Portal Works

### 1. WiFi Access Point Creation
When you start an evil portal, the ESP32 creates a WiFi access point with your specified SSID and password (optional for open networks).

### 2. Captive Portal Detection
The system implements captive portal detection mechanisms:
- **Android**: Responds to connectivity checks at `/generate_204`
- **Apple**: Responds to connectivity checks at `/hotspot-detect.html`
- **Microsoft**: Responds to connectivity checks at `/ncsi.txt`
- **GNOME**: Responds to connectivity checks at `/success.txt`

### 3. DNS Hijacking
All DNS requests are redirected to the ESP32's IP address (192.168.4.1), ensuring users see your portal page regardless of what URL they try to visit.

### 4. HTTP Server
The ESP32 runs an HTTP server that:
- Serves your custom HTML portal pages
- Handles form submissions at `/login` endpoint
- Captures keystrokes via JavaScript injection
- Serves static assets (CSS, JS, images)

### 5. Data Capture
The system automatically injects JavaScript into every served page that:
- Captures form submissions
- Records keystrokes in real-time
- Sends data to `/api/log` endpoint
- Logs all captured data to SD card files

## Architecture

### Core Components

#### 1. WiFi Manager (`wifi_manager.c`)
- Manages WiFi access point creation
- Handles captive portal detection
- Implements HTTP server with multiple endpoints
- Manages data capture and logging

#### 2. SD Card Manager (`sd_card_manager.c`)
- Lists available portal files from `/mnt/ghostesp/evil_portal/portals/`
- Manages file indexing for captured data
- Handles file operations for portal assets

#### 3. Command Line Interface (`commandline.c`)
- Implements `startportal`, `stopportal`, `listportals` commands
- Handles `evilportal` command for UART HTML upload
- Manages command parsing and validation

### File Structure

```
SD Card Structure:
/ghostesp/
├── evil_portal/
│   ├── portals/           # Custom portal HTML files
│   │   ├── myportal.html
│   │   ├── hotel_login.html
│   │   └── ...
│   ├── portal_creds_0.txt # Captured credentials
│   ├── portal_creds_1.txt
│   ├── portal_keystrokes_0.txt # Captured keystrokes
│   └── portal_keystrokes_1.txt
```

### Runtime Paths
- **Portal Directory**: `/mnt/ghostesp/evil_portal/portals/`
- **Credentials Logs**: `/mnt/ghostesp/evil_portal/portal_creds_N.txt`
- **Keystroke Logs**: `/mnt/ghostesp/evil_portal/portal_keystrokes_N.txt`

## Commands Reference

### startportal
Starts an evil portal with specified parameters.

**Syntax:**
```
startportal [FilePath] [AP_SSID] [PSK]
```

**Parameters:**
- `FilePath`: Path to HTML file or "default" for built-in portal
- `AP_SSID`: WiFi network name (required)
- `PSK`: WiFi password (optional, creates open network if omitted)

**Examples:**
```bash
# Start with built-in portal
startportal default "Free WiFi"

# Start with custom portal file
startportal myportal.html "Hotel WiFi"

# Start with password-protected network
startportal default "Secure WiFi" "password123"
```

**Path Handling:**
- If path doesn't start with `/mnt/`, automatically prepends `/mnt/ghostesp/evil_portal/portals/`
- Use "default" for the built-in Google-style portal
- File must have `.html` extension

### stopportal
Stops the currently running evil portal.

**Syntax:**
```
stopportal
```

### listportals
Lists all available HTML portal files on the SD card.

**Syntax:**
```
listportals
```

**Output:**
Shows all `.html` files in `/mnt/ghostesp/evil_portal/portals/` directory.

### evilportal
Configures evil portal HTML content via UART buffer.

**Syntax:**
```
evilportal -c sethtmlstr
```

**Process:**
1. Run: `evilportal -c sethtmlstr`
2. Send `[HTML/BEGIN]` marker over UART
3. Send HTML content over UART (max 2048 bytes)
4. Send `[HTML/CLOSE]` marker over UART
5. Run `startportal` (will use buffered HTML)

**Limitations:**
- Maximum 2048 bytes for UART upload
- HTML is stored in memory buffer
- Buffer is cleared when portal stops

## Creating Portal Files

### Method 1: SD Card Files (Recommended)

1. **Prepare SD Card:**
   - Format SD card to FAT32
   - Create directory: `/ghostesp/evil_portal/portals/`

2. **Create HTML File:**
   - Create your HTML file with `.html` extension
   - Place in `/ghostesp/evil_portal/portals/` directory
   - Use any filename (e.g., `hotel_login.html`)

3. **Start Portal:**
   ```bash
   startportal hotel_login.html "Hotel WiFi"
   ```

### Method 2: UART Upload

1. **Start Upload Process:**
   ```bash
   evilportal -c sethtmlstr
   ```

2. **Send HTML Content:**
   - Send `[HTML/BEGIN]` marker
   - Send your HTML content (max 2048 bytes)
   - Send `[HTML/CLOSE]` marker

3. **Start Portal:**
   ```bash
   startportal default "Free WiFi"
   ```

### Method 3: Using SingleFile Extension

1. **Find Target Login Page:**
   - Visit the login page you want to replicate
   - Ensure it works well on mobile devices

2. **Save with SingleFile:**
   - Use SingleFile browser extension
   - Save the complete page as HTML
   - This includes all CSS and images inline

3. **Modify Form Action:**
   - Open saved HTML file in text editor
   - Find `<form>` tag
   - Change `action` to `/login`
   - Ensure `method="post"`

4. **Save and Deploy:**
   - Save as `.html` file
   - Place in SD card portal directory
   - Start portal with your file

## Templates and Examples

### Built-in Default Portal
The system includes a Google-style login page with:
- Professional styling
- Email and password fields
- Form action pointing to `/get`
- Built-in keystroke capture JavaScript
- Mobile-responsive design

### Simple Template
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <title>WiFi Login</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background: #f5f5f5;
            margin: 0;
            padding: 20px;
        }
        .container {
            max-width: 400px;
            margin: 50px auto;
            background: white;
            padding: 30px;
            border-radius: 8px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        .form-group {
            margin-bottom: 20px;
        }
        label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        input[type="text"], input[type="password"] {
            width: 100%;
            padding: 12px;
            border: 1px solid #ddd;
            border-radius: 4px;
            box-sizing: border-box;
        }
        button {
            width: 100%;
            padding: 12px;
            background: #007bff;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            font-size: 16px;
        }
        button:hover {
            background: #0056b3;
        }
    </style>
</head>
<body>
    <div class="container">
        <h2>WiFi Login Required</h2>
        <p>Please enter your credentials to access the network.</p>
        
        <form action="/login" method="post">
            <div class="form-group">
                <label for="username">Username:</label>
                <input type="text" id="username" name="username" required>
            </div>
            
            <div class="form-group">
                <label for="password">Password:</label>
                <input type="password" id="password" name="password" required>
            </div>
            
            <button type="submit">Login</button>
        </form>
    </div>
</body>
</html>
```

### Hotel WiFi Template
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <title>Hotel WiFi Access</title>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            margin: 0;
            padding: 20px;
            min-height: 100vh;
        }
        .container {
            max-width: 450px;
            margin: 50px auto;
            background: white;
            padding: 40px;
            border-radius: 12px;
            box-shadow: 0 8px 32px rgba(0,0,0,0.1);
        }
        .hotel-logo {
            text-align: center;
            margin-bottom: 30px;
        }
        .hotel-logo h1 {
            color: #333;
            margin: 0;
            font-size: 28px;
        }
        .hotel-logo p {
            color: #666;
            margin: 5px 0 0 0;
        }
        .form-group {
            margin-bottom: 25px;
        }
        label {
            display: block;
            margin-bottom: 8px;
            font-weight: 600;
            color: #333;
        }
        input[type="text"], input[type="password"] {
            width: 100%;
            padding: 15px;
            border: 2px solid #e1e5e9;
            border-radius: 8px;
            box-sizing: border-box;
            font-size: 16px;
            transition: border-color 0.3s;
        }
        input[type="text"]:focus, input[type="password"]:focus {
            outline: none;
            border-color: #667eea;
        }
        button {
            width: 100%;
            padding: 15px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-size: 16px;
            font-weight: 600;
            transition: transform 0.2s;
        }
        button:hover {
            transform: translateY(-2px);
        }
        .info {
            background: #f8f9fa;
            padding: 15px;
            border-radius: 8px;
            margin-top: 20px;
            font-size: 14px;
            color: #666;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="hotel-logo">
            <h1>🏨 Grand Hotel</h1>
            <p>Complimentary WiFi Access</p>
        </div>
        
        <form action="/login" method="post">
            <div class="form-group">
                <label for="room">Room Number:</label>
                <input type="text" id="room" name="room" placeholder="e.g., 101" required>
            </div>
            
            <div class="form-group">
                <label for="lastname">Last Name:</label>
                <input type="text" id="lastname" name="lastname" placeholder="Your last name" required>
            </div>
            
            <button type="submit">Connect to WiFi</button>
        </form>
        
        <div class="info">
            <strong>Instructions:</strong><br>
            Enter your room number and last name to access our complimentary WiFi network.
        </div>
    </div>
</body>
</html>
```

## File Structure

### Supported File Types
The evil portal server supports serving these file types:
- **HTML**: `.html` - Portal pages
- **CSS**: `.css` - Stylesheets
- **JavaScript**: `.js` - Scripts
- **Images**: `.png`, `.jpg` - Graphics

### File Serving
- Files are served from the same directory as the main HTML file
- Static assets are automatically served when referenced in HTML
- File paths are relative to the portal file location

### Directory Structure
```
/mnt/ghostesp/evil_portal/portals/
├── myportal.html          # Main portal file
├── style.css              # Stylesheet (if external)
├── script.js              # JavaScript (if external)
├── logo.png               # Images
└── background.jpg
```

## Data Capture

### Automatic JavaScript Injection
Every served HTML page automatically receives JavaScript injection that:

1. **Captures Form Submissions:**
   - Monitors all form elements
   - Records field names and values
   - Sends data to `/api/log` endpoint

2. **Keystroke Logging:**
   - Records all keystrokes in real-time
   - Captures input field changes
   - Logs character-by-character input

3. **Data Format:**
   ```
   timestamp|tag|name/id|value
   1703123456789|input|username|john_doe
   1703123456790|input|password|secret123
   ```

### Captured Data Storage
- **Credentials**: Saved to `portal_creds_N.txt`
- **Keystrokes**: Saved to `portal_keystrokes_N.txt`
- **Indexing**: Files are automatically indexed (0, 1, 2, etc.)
- **Location**: `/mnt/ghostesp/evil_portal/` directory

### Form Processing
- Forms should use `action="/login"` and `method="post"`
- All form data is automatically captured and logged
- Original form behavior is preserved for user experience

## Troubleshooting

### Common Issues

#### "I don't see the WiFi network"
**Solutions:**
1. Wait 30 seconds - network creation takes time
2. Turn phone WiFi off and on
3. Check command syntax
4. Restart the ESP32 device
5. Verify SSID doesn't contain special characters

#### "I see the network but no login page"
**Solutions:**
1. Disconnect from other WiFi networks
2. Try opening `http://neverssl.com` in browser
3. Manually navigate to `http://192.168.4.1`
4. Clear browser cache and cookies
5. Try different device/browser

#### "The page looks broken on mobile"
**Solutions:**
1. Use mobile-responsive CSS
2. Test on multiple devices
3. Use the provided simple template
4. Avoid complex layouts
5. Keep file size under 50KB

#### "Portal file not found"
**Solutions:**
1. Check file is in correct directory: `/ghostesp/evil_portal/portals/`
2. Verify file has `.html` extension
3. Use `listportals` command to verify
4. Check SD card is properly mounted
5. Ensure file is not corrupted

#### "No data being captured"
**Solutions:**
1. Verify form uses `action="/login"` and `method="post"`
2. Check SD card has write permissions
3. Ensure form fields have `name` attributes
4. Test with built-in portal first
5. Check logs for error messages

### Debug Commands
```bash
# List available portals
listportals

# Check SD card status
sdinfo

# View system logs
log

# Test with default portal
startportal default "Test WiFi"
```

## Best Practices

### Portal Design
1. **Keep it Simple:**
   - Use minimal, clean designs
   - Avoid complex animations
   - Focus on mobile compatibility

2. **Make it Believable:**
   - Use realistic branding
   - Include proper instructions
   - Match target environment style

3. **Mobile-First:**
   - Design for mobile devices first
   - Use responsive CSS
   - Test on actual mobile devices

### File Management
1. **File Naming:**
   - Use descriptive names
   - Avoid spaces and special characters
   - Keep names under 64 characters

2. **File Size:**
   - Keep HTML files under 50KB
   - Optimize images
   - Use inline CSS/JS when possible

3. **Testing:**
   - Test portals before deployment
   - Verify on multiple devices
   - Check form submission works

### Security Considerations
1. **Legal Use Only:**
   - Only use on networks you own
   - Get explicit permission
   - Follow local laws and regulations

2. **Data Handling:**
   - Secure captured data
   - Delete data after testing
   - Don't store sensitive information

3. **Network Security:**
   - Use strong WiFi passwords
   - Monitor network activity
   - Disable when not in use

### Performance Optimization
1. **Memory Management:**
   - Keep HTML files small
   - Use efficient CSS
   - Minimize JavaScript

2. **Network Performance:**
   - Optimize images
   - Use compressed assets
   - Minimize HTTP requests

3. **Battery Life:**
   - Stop portal when not needed
   - Use efficient code
   - Monitor power consumption

---

## Quick Reference

### Essential Commands
```bash
# Start portal with default page
startportal default "Free WiFi"

# Start portal with custom file
startportal myportal.html "Hotel WiFi"

# List available portals
listportals

# Stop portal
stopportal

# Upload HTML via UART
evilportal -c sethtmlstr
```

### File Locations
- **Portal Files**: `/ghostesp/evil_portal/portals/`
- **Captured Data**: `/ghostesp/evil_portal/`
- **Runtime Path**: `/mnt/ghostesp/evil_portal/portals/`

### Supported Formats
- **HTML**: Portal pages
- **CSS**: Stylesheets  
- **JS**: JavaScript
- **PNG/JPG**: Images

### Default Network
- **IP**: 192.168.4.1
- **Gateway**: 192.168.4.1
- **DNS**: 192.168.4.1

---

*Remember: Evil Portal is for educational and authorized testing purposes only. Always ensure you have proper authorization before using this tool.*
