# Privileged-Access-Management-PAM-Lab-with-HashiCorp-Vault
I can't get a full CyberArk license for free, but you can build a scaled-down, conceptually real version of exactly what it does — vaulting, controlled access, and logging — using a genuinely industry-recognized open-source tool: HashiCorp Vault.

As a beginner and intermediate learner in IAM, Initially the plan was to build a Vault issue you a temporary access token, and review the audit log of everything that happened — recreating the Vault + CPM + PSM logging concepts but getting into the project, it gets developed to big. 

# What I / you'll build:
A running Vault server on your own computer, where you'll store a "secret" (a fake privileged password), retrieve it through the proper controlled process (not just reading a file), watch Vault issue you a temporary access token, and review the audit log of everything that happened — recreating the Vault + CPM + PSM logging concepts.

By the end, you've hands-on demonstrated all four PAM pillars in a real, running system: vaulting a credential instead of leaving it in plain text, retrieving it through a controlled interface instead of direct access, issuing just-in-time, expiring access instead of standing permanent access, and generating a genuine audit log of every access event — the same core architecture CyberArk (Chapter 20) implements at enterprise scale.

## Step 1 — Install HashiCorp Vault

On Windows:


- Open your browser and go to developer.hashicorp.com/vault/install
- Under Windows, click the link for the latest version's .zip file and download it
- Once downloaded, right-click the .zip file → Extract All → choose to extract it to C:\vault
- Open your Start menu, type cmd, and open Command Prompt
- Type this and press Enter to move into that folder:

      cd C:\vault

On Mac:

- Open your terminal (Cmd+Space, type terminal, press Enter)
- If you don't already have Homebrew (a package installer for Mac), install it by typing:

      bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

and pressing Enter, then following any on-screen prompts 3. 
- Once Homebrew is ready, type:

      brew tap hashicorp/tap
      brew install hashicorp/tap/vault

- Check it worked (both systems): type this and press Enter:

      vault --version

- You should see a version number printed. If you get an error on Windows, make sure you're still inside the C:\vault folder (type cd C:\vault again) and try " vault.exe --version " instead.

# Step 2 — Start Vault in learning mode

- Type this and press Enter:


      vault server -dev

- Leave this window open — this is your running Vault server, similar to leaving the OAuth project's server running in Part 2. You'll see a wall of text print out, including two important lines:


  Unseal Key — ignore this for now, it's for production setups


  Root Token — copy this value somewhere safe. This is your master credential for talking to this Vault server (in a real company, this exact "root token has too much power" problem is itself a PAM concern — root tokens are only ever used briefly during initial setup)

# Step 3 — Open a second terminal window and connect to Vault

- Since your first terminal window is busy running the server, open a brand new terminal window (repeat Step 0's opening method) — you'll use this second window to actually interact with Vault.


In this new window, tell it where your Vault server is and log in using the Root Token you copied:

- Windows (Command Prompt):


      set VAULT_ADDR=http://127.0.0.1:8200
      set VAULT_TOKEN=paste_your_root_token_here
      cd C:\vault

- Mac (Terminal):


      export VAULT_ADDR=http://127.0.0.1:8200
      export VAULT_TOKEN=paste_your_root_token_here

# Step 4 — Store your first "privileged secret"

- Let's pretend this is a database admin password that needs vaulting. Type:

vault kv put secret/database-admin username="dbadmin" password="SuperSecretPassword123"

- Press Enter. Vault confirms it was stored, encrypted, inside the vault — this is the exact same concept as CyberArk's Digital Vault

# Step 5 — Retrieve the secret the "proper" way

- Instead of a person just knowing this password, they'd request it through Vault when needed. Type:

vault kv get secret/database-admin

- You'll see the username and password printed back — this simulates the controlled retrieval process. Notice: unlike a password sitting visibly in a text file, this retrieval just got logged automatically (you'll see this in Step 7).

# Step 6 — Issue a short-lived access token (simulating Just-in-Time access)

- This recreates the JIT concept from Chapter 19 — instead of permanent access, you'll create a token that self-destructs after a short time window. Type:

vault token create -ttl=2m

- This creates a brand-new access token that automatically expires in 2 minutes. Copy the new token it prints out.

- Now, still in this same window, temporarily switch to using that limited token instead of your root token:


  Windows:


        set VAULT_TOKEN=paste_the_new_token_here


  Mac:


      export VAULT_TOKEN=paste_the_new_token_here

- Try running the same command from Step 5 again (vault kv get secret/database-admin) — it still works, because the token is still valid. Now wait 2 minutes, doing nothing, and run the exact same command again. This time, it should fail with a permission/expired error — that expiration failure is Just-in-Time access working exactly as designed. Switch back to your Root Token afterward (repeat the set/export command from Step 3) to keep going.
