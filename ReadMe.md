# 🔑 Recovering Saved Website Passwords from Chromium‑Based Browsers

*A Live‑System DFIR Workflow for Windows E01 Images*



---

## 📌 Table of Contents

1. [Why This Matters](#why-this-matters)
2. [Why a Live System Is Required](#why-a-live-system-is-required)
3. [Technical Background](#technical-background)
4. [Tools and Dependencies](#tools-and-dependencies)
5. [Workflow (Step‑by‑Step)](#workflow-step-by-step)
6. [Troubleshooting Tips](#troubleshooting-tips)
7. [Ethics and Legal Considerations](#ethics-and-legal-considerations)
8. [Contributing](#contributing)
9. [Acknowledgements](#acknowledgements)
10. [License](#license)

---

## 🚀 Why This Matters

- **Market Reality** – Chromium‑based browsers (Chrome, Edge, Brave, Opera) account for **≈ 65 %** of global usage.
- **Investigation Impact** – Recovered credentials prove account ownership, reveal lateral movement, and expose password reuse.
- **Threat Landscape** – Modern infostealers (RedLine, Raccoon, Vidar, etc.) use the *exact* chain below to harvest passwords.
- **Evidentiary Value** – Plain‑text passwords fortify timelines, attribution, and damage assessments in insider‑threat, fraud, or data‑leak cases.

---

## 🖥️ Why a Live System Is Required

Chromium secures each password with:

1. A random **AES‑256 key** stored in `Local State` → `"os_crypt" → "encrypted_key"`.
2. That key is **DPAPI‑protected**, linked to the user’s Windows credentials.
3. Each password in the `Login Data` SQLite DB is encrypted with this AES key.

> **Offline decryption fails** because DPAPI can only be unlocked inside the original logon context.
>
> **Solution:** Boot the suspect E01 image as a **virtual machine** and operate within that user context.

Need a VM? Follow the companion workflow [**Convert E01 → Bootable VM**](../Convert_Forensic_Image_to_VM).

---

## 🔬 Technical Background

| File                  | Key Data         | Purpose                  |
| --------------------- | ---------------- | ------------------------ |
| `Local State` (JSON)  | `encrypted_key`  | AES key, DPAPI‑encrypted |
| `Login Data` (SQLite) | `password_value` | Encrypted site passwords |

`password_value` structure:

```
v10 | 12‑byte IV | ciphertext | 16‑byte GCM tag
```

Decryption chain:

1. Strip the `DPAPI` prefix → Base64‑decode → DPAPI‑decrypt → **AES key**
2. Parse IV, ciphertext, tag from `password_value`
3. AES‑GCM decrypt → **plain‑text password**

---

## 🧰 Tools and Dependencies

| Stage     | Tool                                                                      | Use                                               |
| --------- | ------------------------------------------------------------------------- | ------------------------------------------------- |
| VM Build  | **FTK Imager**, **VBoxManage**, **VirtualBox**                            | Boot live Windows                                 |
| Secrets   | **Mimikatz**                                                              | Extract DPAPI master keys & decrypt `Local State` |
| DB Work   | **sqlite3** / DB Browser for SQLite                                       | Query `Login Data`                                |
| Scripting | **Python 3** + [`pycryptodomex`](https://pypi.org/project/pycryptodomex/) | AES‑GCM decryption scripts                        |
| Hashing   | `sha256sum` / `CertUtil`                                                  | Evidence integrity                                |

---

## ⚙️ Workflow (Step‑by‑Step)

### 1️⃣ Boot the VM

Attach `final_vm.vmdk`, enable **EFI** if needed, and log in with the recovered PIN/password.

### 2️⃣ Extract & Decrypt the AES Key

```powershell
mimikatz # privilege::debug
mimikatz # dpapi::decrypt /in:"%LOCALAPPDATA%\Google\Chrome\User Data\Local State" /unprotect
```

Save the **Base64‑decoded AES key** as `local_state_key.bin`.

### 3️⃣ Query the Encrypted Password

```bash
sqlite3 "Login Data" \
  "SELECT origin_url, username_value, password_value
     FROM logins
     WHERE origin_url LIKE '%fox.com%';" \
  > encrypted_password.txt
```

### 4️⃣ Split IV, Ciphertext, Tag

```bash
python split_cipher.py encrypted_password.txt
# ➞ produces iv.bin, ciphertext.bin, tag.bin
```

### 5️⃣ Decrypt the Password

```bash
python decrypt_chrome_pass.py \
  --key local_state_key.bin \
  --iv iv.bin \
  --cipher ciphertext.bin \
  --tag tag.bin
```

Example output:

```
Site  : https://www.fox.com/
User  : jane.doe@example.com
Pass  : MyS3cur3P@ss!
```

### 6️⃣ Document & Preserve

1. Screenshot results.
2. Hash all exported files using `SHA‑256`.
3. Record artefact paths (`Login Data`, `Local State`, DPAPI blobs).
4. Store evidence securely in the case file.

---

## 🧩 Troubleshooting Tips

| Issue                  | Likely Cause                       | Fix                                                    |
| ---------------------- | ---------------------------------- | ------------------------------------------------------ |
| VM BSOD or won’t boot  | Controller mismatch / EFI disabled | Enable EFI; match storage controller; assign ≥2 vCPUs. |
| `dpapi::decrypt` fails | Not elevated / wrong creds         | Run `privilege::debug`; verify PIN/password.           |
| AES‑GCM tag error      | Wrong IV or tag length             | Ensure IV = 12 bytes, tag = 16 bytes.                  |

---

## ⚖️ Ethics and Legal Considerations

- **Legal Authority** – Obtain warrant or client consent before decrypting user passwords.
- **Chain‑of‑Custody** – Log every command; hash every artefact.
- **Isolation** – Perform recovery in an isolated sandbox (no outbound network).
- **Reporting** – Mask passwords in deliverables unless otherwise required by court.

---

## 🤝 Contributing

Improvements welcome! Fork, branch, and open a pull request with fixes or version‑specific decryptors.

---

## 🙏 Acknowledgements

- [gentilkiwi / Mimikatz](https://github.com/gentilkiwi/mimikatz)
- [PyCryptodome](https://github.com/Legrandin/pycryptodome)
- **University of Baltimore** — Cyber Forensics Program
- **Prudential Associates** — Real‑world DFIR Environment

---

## 📜 License

Released for **educational and research purposes**.\
Please attribute the author if you reuse or adapt this workflow.

---

**Author**\
**Daniel Kwaku Ntiamoah Addai**\
Graduate Research Assistant — *University of Baltimore*\
Senior DFIR Analyst — *Prudential Associates*

