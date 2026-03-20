# Colleague Setup Guide: DeluluDocu & GitHub Sync

Follow these steps to synchronize your Zoho Deluge scripts with our shared GitHub repository and Obsidian vault.

## 1. GitHub Account & Access

- **Be Added as a Collaborator**: Make sure the repository owner has added your GitHub username to the `zoho-scripts` repo as a collaborator with "Write" access.
- **Create a Personal Access Token (PAT)**:
    1. Go to your GitHub [Developer Settings](https://github.com/settings/tokens).
    2. Click **Generate new token (classic)**.
    3. Name it "DeluluDocu Sync".
    4. Select the **repo** scope (Full control of private repositories).
    5. **Save the token!** You will need it for the extension.

---

## 2. Clone the Repository to Your Machine

Open your Terminal and run the following commands to create your local copy and set up the Obsidian vault:

bash

# 1. Navigate to where you want to store your scripts

cd ~/Documents

# 2. Clone the repository

git clone https://github.com/martinkp93/zoho-scripts.git

# 3. Enter the directory

cd zoho-scripts

---

## 3. Install the DeluluDocu Extension

1. **Download the Extension Folder**: Get the "deluludocu-github-extension" folder from the repository or your colleague.
2. **Open Chrome/Edge**: Go to `chrome://extensions/` or `edge://extensions/`.
3. **Enable Developer Mode**: Toggle the switch in the top-right corner.
4. **Load Unpacked**: Click "Load unpacked" and select the extension folder.
5. **Configure**:
    - Click the extension icon in your toolbar.
    - Enter the **GitHub Owner** (`martinkp93`), **Repository** (`zoho-scripts`), and your **GitHub Token**.
    - Enter your **Gemini API Key** (optional, for AI documentation).

---

## 4. Connect Obsidian

1. Open Obsidian and click **Open folder as vault**.
2. Select the `zoho-scripts` folder you cloned in Step 2.
3. **Install the "Obsidian Git" Plugin**:
    - Go to Settings > Community Plugins > Browse.
    - Search for **Obsidian Git** and install/enable it.
    - Configure it to **Pull changes** automatically every 5-10 minutes.

---

## How to Sync a Script

1. Open any Deluge script in Zoho CRM or Zoho Billing.
2. Make a small change (or just add a space).
3. Click **Save**.
4. **Result**: The extension will automatically documentation the change and push it to GitHub. You (and your colleague) will see the update in your Obsidian vault after the next pull.

---

IMPORTANT

**Don't Forget**: If you manually move or rename folders in Obsidian, the extension will "find" them automatically during the next sync. No manual path mapping is ever required!