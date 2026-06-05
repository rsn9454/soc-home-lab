# Step 14 — osTicket Install + Kibana Webhook Integration

## Objective
Install osTicket on the Windows host using XAMPP, complete the initial configuration, then wire Kibana's webhook connector to the osTicket API so that every detection rule alert automatically opens a ticket in the SOC ticketing system — closing the loop from raw log → SIEM alert → analyst ticket.

---

## Prerequisites
- At least one detection rule (SSH brute force, RDP brute force, or Apollo C2) is active and firing alerts in Kibana Security
- VM1 (ELK + Fleet) is running — Kibana must be accessible 

---

## Part 1 — Install XAMPP

XAMPP bundles Apache, MySQL, and PHP — everything osTicket needs to run.

### Step 1 — Download XAMPP
Go to [apachefriends.org](https://www.apachefriends.org) and download the
latest XAMPP for Windows installer (PHP 8.x version).

### Step 2 — Install XAMPP
Run the installer as Administrator:

| Setting | Value |
|---|---|
| Components | Apache, MySQL, PHP, phpMyAdmin (defaults) |
| Install path | `C:\xampp` (keep default) |

When installation completes and the control panel opens, start both services:
- Click **Start** next to **Apache**
- Click **Start** next to **MySQL**

Both should show green **Running** status.

### Step 3 — Verify XAMPP is working
Open a browser on the Windows host and go to:
```
http://localhost
```
You should see the XAMPP dashboard page. Apache is working.

---

## Part 2 — Deploy osTicket

### Step 1 — Download osTicket
Go to [osticket.com/download](https://osticket.com/download) and download
the latest stable release `.zip` file.

### Step 2 — Extract and deploy
1. Extract the downloaded zip
2. Inside the extracted folder, find the `upload` folder
3. Copy `upload` to `C:\xampp\htdocs\`
4. Rename it from `upload` to `osTicket`:
   ```
   C:\xampp\htdocs\osTicket\
   ```

### Step 3 — Configure the osTicket config file
Navigate to:
```
C:\xampp\htdocs\osTicket\include\
```

Rename `ost-sampleconfig.php` → `ost-config.php`

---

## Part 3 — Create the MySQL database

### Step 1 — Open phpMyAdmin
```
http://localhost/phpmyadmin
```
Login: username `root`, password blank (XAMPP default).

### Step 2 — Create the database
1. Click **New** in the left sidebar
2. Database name: `osticket`
3. Collation: `utf8mb4_general_ci`
4. Click **Create**

### Step 3 — Create a dedicated database user
1. Go to **User accounts → Add user account**
2. Fill in:

| Field | Value |
|---|---|
| Username | `osticket` |
| Host | `localhost` |
| Password | (strong password — store securely) |

3. Under **Database for user account** → tick **Grant all privileges on database "osticket"**
4. Click **Go**

---

## Part 4 — Run the osTicket web installer

Open the installer:
```
http://localhost/osTicket/setup/
```

### Check prerequisites
All required PHP extensions should show green with XAMPP. If any show red:
- Open `C:\xampp\php\php.ini`
- Find the relevant extension line (e.g. `;extension=intl`)
- Remove the leading semicolon to enable it
- Restart Apache in the XAMPP Control Panel

### Fill in the installation form

**System Settings:**
| Field | Value |
|---|---|
| Helpdesk Name | `SOC Lab Ticketing` |
| Default Email | `soc@lab.local` |

**Admin User:**
| Field | Value |
|---|---|
| First Name / Last Name | Your name |
| Email Address | `admin@lab.local` |
| Username | `admin` |
| Password | (strong — store securely) |

**Database Settings:**
| Field | Value |
|---|---|
| MySQL Table Prefix | `ost_` |
| MySQL Hostname | `localhost` |
| MySQL Database | `osticket` |
| MySQL Username | `osticket` |
| MySQL Password | (from Part 3) |

Click **Install Now**.

### After successful installation
The installer shows links to:
- **Staff Control Panel:** `http://localhost/osTicket/scp/`
- **Client Portal:** `http://localhost/osTicket/`

**Delete the setup folder immediately** — osTicket requires this:
```
Delete: C:\xampp\htdocs\osTicket\setup\
```

---

## Part 5 — Initial osTicket configuration

### Step 1 — Generate the API key

The Kibana webhook needs an API key to create tickets:

1. Go to **Admin Panel → Manage → API Keys**
2. Click **Add New API Key**

| Field | Value |
|---|---|
| IP Address | `127.0.0.1` |
| Notes | `Kibana alert webhook` |

3. Tick **Can Create Tickets**
4. Click **Add Key**
5. **Copy the API key immediately**

---

## Part 6 — Connect Kibana to osTicket (Webhook connector)

### Step 1 — Activate the Kibana trial licence
The Webhook connector requires the trial licence. In Kibana:

1. Go to **Management → Stack Management → Licence Management**
2. Click **Start trial**
3. Confirm — this is free and requires no credit card

### Step 2 — Create the Webhook connector
1. Go to **Management → Stack Management → Connectors**
2. Click **Create connector**
3. Select **Webhook**
4. Configure:

| Setting | Value |
|---|---|
| Connector name | `osTicket - SOC Lab` |
| Method | POST |
| URL | `http://127.0.0.1/osTicket/api/tickets.json` |
| Authentication | None |

5. Under **Add HTTP header**:

| Key | Value |
|---|---|
| `X-API-Key` | `YOUR_OSTICKET_API_KEY` |

6. Click **Save**

### Step 3 — Configure the request body template
After saving the connector, go back into it and set the **Body**:

Go to https://github.com/osTicket/osTicket/blob/develop/setup/doc/api/tickets.md

Copy the XML payload example and paste it into the body

### Step 4 — Test the connector
Click **Save & test** → **Run**

A test POST is sent to osTicket. Go to:
```
http://localhost/osTicket/scp/
```

Click **Admin Panel** → you should see a new test ticket in the queue.

---

## Key takeaway
The Kibana → osTicket integration is what transforms a passive detection platform into an active SOC workflow. Before this step, alerts existed only inside Kibana — a SOC analyst would have to manually check the Alerts page to notice them. After this step, every alert fires a webhook that opens a ticket automatically, creating an auditable trail of: what fired, when, and whether it was investigated. This is how every production SOC operates — the SIEM detects, the ticketing system assigns, and the analyst investigates and closes. Having built the entire chain from raw log to closed ticket in a home lab is one of the strongest demonstrations of end-to-end SOC workflow understanding you can show.

---

## VM status at end of step

| VM | Status |
|---|---|
| ELK + Fleet (VM1) | ✅ Running — Elasticsearch :9200, Kibana :5601, Fleet :8220 |
| Windows target (VM2) | ⏸ Suspended |
| Ubuntu target (VM3) | ⏸ Suspended |
| Mythic C2 (VM4) | ⏸ Suspended |
| osTicket (host) | ✅ Running — http://localhost/osTicket/scp/ |