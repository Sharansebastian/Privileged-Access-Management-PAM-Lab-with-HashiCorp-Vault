# Privileged-Access-Management-PAM-Lab-with-HashiCorp-Vault
I can't get a full CyberArk license for free, but you can build a scaled-down, conceptually real version of exactly what it does — vaulting, controlled access, and logging — using a genuinely industry-recognized open-source tool: HashiCorp Vault.

As a beginner and intermediate learner in IAM, Initially the plan was to build a Vault issue you a temporary access token, and review the audit log of everything that happened — recreating the Vault + CPM + PSM logging concepts but getting into the project, it gets developed to big. 

# What I / you'll build:
A running Vault server on your own computer, where you'll store a "secret" (a fake privileged password), retrieve it through the proper controlled process (not just reading a file), watch Vault issue you a temporary access token, and review the audit log of everything that happened — recreating the Vault + CPM + PSM logging concepts.

By the end, you've hands-on demonstrated all four PAM pillars in a real, running system: vaulting a credential instead of leaving it in plain text, retrieving it through a controlled interface instead of direct access, issuing just-in-time, expiring access instead of standing permanent access, and generating a genuine audit log of every access event — the same core architecture CyberArk implements at enterprise scale.

Note: Eventually you'll get angry trying this project, even i got. but trying out this will give you a experience. 

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

# Step 7 — Review the audit log (simulating PSM session recording)

- Real PAM tools log every single access event. Let's turn that on and see it for ourselves. Back in your second terminal window (with the Root Token active again), type:

   Windows:

      vault audit enable file file_path=C:\vault\vault-audit.log

   Mac:

      vault audit enable file file_path=/tmp/vault-audit.log

- Now repeat the retrieval from Step 5 one more time:

      vault kv get secret/database-admin

- Then open the log file to see what got recorded:


  Windows: type "notepad C:\vault\vault-audit.log"
  and press Enter
  Mac: "type cat /tmp/vault-audit.log " and press Enter

- You'll see a dense block of JSON text — don't worry about reading every field. Just notice: every single retrieval you just performed is permanently recorded, including the time it happened. This is the exact same principle as CyberArk's PSM session recording, just in log form instead of video form.

# Step 8 — Shut everything down

Go back to your first terminal window (the one still running vault server -dev) and press Ctrl + C to stop the server. Everything you built was local and temporary — closing it removes the dev server completely, which is expected and fine for a learning setup like this.

# Where i got interested and went in more

Added these 4 things:


Create two identities: 
      
    admin, operator

Give them different Vault policies: 

    Admin can manage the secret; Operator can retrieve it but cannot modify it.

Demonstrate token lifecycle

    Login/authenticate
    Receive token
    Use token
    Show TTL/expiration
    Let it expire or revoke it
    Demonstrate that access subsequently fails.
Show the audit trail

    Authentication event
    Secret read
    Token issuance/use
    Failed unauthorized attempt
    Token revocation/expiration

Here's how each piece maps to the UI (http://127.0.0.1:8200, logged in with your Root Token):

1. Create the two policies


Left sidebar → Policies → ACL Policies → Create ACL policy


Name it admin-policy, paste in the same HCL from before:

    path "secret/data/database-admin" {
      capabilities = ["create", "read", "update", "delete"]
    }
Save. Repeat for operator-policy with:

    path "secret/data/database-admin" {
      capabilities = ["read"]
    }


2. Create the two identities (tokens)

In your second terminal window (with Root Token active):

    vault token create -policy="admin-policy" -display-name="admin-user" -ttl=5m

Copy the token it prints.

    vault token create -policy="operator-policy" -display-name="operator-user" -ttl=3m

Copy this one too.

Then back in the browser:

   Sign out of the UI (top-right menu).
  
   On the login screen, paste the operator token as the credential → sign in.
   
   Browse to Secrets → secret/ → database-admin → confirm you can view it but editing/saving fails.
   
   Sign out, sign back in with the admin token → confirm editing succeeds.

3. Demonstrate access differences

  Do this step in the CLI, as the UI doesn't offer this step
- Log in using the operator token as the credential and type:

      vault kv get secret/database-admin
  Should get an error or permission denied

- Try editing/saving it →

      vault kv put secret/database-admin username="dbadmin" password="SuperSecretPassword1234"
   should get an error or permission denied
  
- Log out by & log back in with the admin token

      set VAULT_TOKEN=paste_admin_token

  → try editing → it succeeds.

4. Token lifecycle (TTL + revocation)

Part A: You've already covered expiration. Since you set -ttl=2m and -ttl=5m and tested logging in after they ran out — that's the "let it expire" half of Step 4 already done. ✅

Part B: Now demonstrate manual revocation (the other half)

This is different from expiration — instead of waiting for a token to die naturally, an admin actively kills it early (e.g., if someone's laptop was stolen, or their access needs to be cut immediately).

1. Create a fresh token to revoke (so you're not reusing an already-expired one):

       vault token create -policy="operator-policy" -display-name="operator-user-2" -ttl=1h

Copy this new token. Notice the TTL is long (1 hour) — the point here is proving revocation kills it immediately, not because it expired.

2. Confirm it currently works — log into via CLI:

       set VAULT_TOKEN=paste_new_operator_token
       vault kv get secret/database-admin

→ should succeed.

3. Switch back to Root Token, then revoke it:

       set VAULT_TOKEN=paste_your_root_token
       vault token revoke paste_new_operator_token

4. Prove access now fails — switch back to the revoked token and try again:

       set VAULT_TOKEN=paste_new_operator_token
       vault kv get secret/database-admin

→ should now fail with permission denied / invalid token — even though its TTL (1 hour) hadn't run out yet. This is the key proof: revocation cuts access instantly, regardless of remaining TTL

5. Audit trail

This part is CLI-only — Vault's dev-mode UI doesn't have a built-in log viewer for the audit file. You'd still need to enable it via CLI (Step 7) and read vault-audit.log in Notepad/cat, even if everything else was done through the browser. The UI actions still get logged to that same file, since audit logging captures all access regardless of whether it came from CLI or UI.

I've also created entities, groups but it is a long process and above my project definition. But I have uploaded the workflow of what i did everything till above this project definition. Check that too.

# Note: Eventually you'll get angry trying this project, even i got. but trying out this will give you a experience. 
