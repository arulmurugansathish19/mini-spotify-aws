# Part 4: The database

- **What:** A managed MySQL database for users, with sign up and login added to the app.
- **Services:** RDS, Secrets Manager, KMS, IAM, security groups, NAT gateway.
- **Result:** Users can sign up and log in, and the database password is never written in the code.

## Goal
Give Mini Spotify a memory for users. The app needs to store accounts and check logins,
and the database must stay hidden from the internet, with its password kept safe.

## Why RDS and not a database on the app server
Later, Auto Scaling will create and delete app servers by itself. If user data lived on one of them,
it would disappear with the server. RDS keeps the data separate, and AWS handles backups and patches.

## What I built

| Component | Details | Why |
|---|---|---|
| `db-sg` | Inbound MySQL (port 3306) from `private-sg` only | Only the app servers can reach the database. Not the jump server, not my laptop, not the internet. |
| DB subnet group `mini-spotify-db-subnets` | My two private subnets, in two zones | Tells RDS which subnets it may live in. Only private ones, so it can never be placed where the internet can reach it. |
| RDS MySQL database `mini-spotify-db` | Smallest burstable micro class, single instance, public access **No**, security group `db-sg` | Small and cheap for a learning project. Public access off keeps it hidden. |
| Password in Secrets Manager | RDS created and stores a strong password for me | The password is never typed or written in the code or on GitHub |
| KMS | Encrypts the secret | Nobody can read the password without permission |
| IAM policy `MiniSpotifyReadDbSecret` | Allows `secretsmanager:GetSecretValue` on that one secret, attached to `MiniSpotifyEC2Role` | Least privilege: the app server can read this one password and nothing else |
| NAT gateway (recreated) | In a public subnet, with the private route table pointing to it | The app server needs a way out to reach Secrets Manager and install `pymysql` |

## How the app finds the password
1. The app server wears the IAM role, so it needs no keys.
2. The code asks Secrets Manager for the secret, using the secret's address from an environment variable.
3. The secret holds the database username and password.
4. The code connects to the database using the endpoint from another environment variable.

## Commands I used

| Command | Meaning | Why |
|---|---|---|
| `pip install pymysql` | Install the library that talks to MySQL | The database equivalent of `boto3` for S3 |
| `export DB_HOST=<database endpoint>` | Save the database address as a setting | The app reads it, so it is not in the code |
| `export DB_SECRET_ARN='<secret ARN>'` | Save the secret's address as a setting | Single quotes stop the terminal from misreading the `!` in the secret's name |
| `export SECRET_KEY=$(python -c "import secrets; print(secrets.token_hex(16))")` | Make a random string | Flask uses it to sign the login cookie, so nobody can fake one |
| `python db_test.py` | Connect to the database and print its version | Proves the chain works: app server, Secrets Manager, database |
| `python create_tables.py` | Create the `users` table | A table is like a sheet with one row per user |
| `pkill gunicorn` | Stop any old copy of the app | An old copy can hold port 5000 |
| `gunicorn -b 0.0.0.0:5000 app:app` | Start the app | Same as before, now with login |

## The users table

| Column | Meaning |
|---|---|
| `id` | A number the database gives each user |
| `email` | The user's email, with `UNIQUE` so two users cannot share one |
| `password_hash` | A scrambled version of the password, never the real one |
| `created_at` | When the account was made |

## How sign up and login work
- **Sign up:** the app checks the email is not taken, scrambles the password with a hash, and saves a new row.
- **Login:** the app finds the user by email and compares the typed password with the saved hash.
- **Session:** a signed cookie says the person is logged in. Song routes return 401 if there is no session.
- **Safe queries:** the code uses `%s` placeholders, so nobody can sneak harmful text into the database (SQL injection).

## Tests

| Test | What I did | Expected result | What it proves |
|---|---|---|---|
| 1 | Open the home page | Redirected to the login page | The app protects the player |
| 2 | Sign up with an email and a password of 6+ characters | Sent to the login page | A new row was saved in RDS |
| 3 | Log in | The player opens and songs play | Passwords are checked, and the session works |
| 4 | Log out, then log in with a wrong password | "Wrong email or password." | The app really checks passwords |
| 5 | Sign up again with the same email | "That email is already registered." | The user is stored in the database, not just in the browser |

## Which AWS service each step used

| Step | Service |
|---|---|
| Create `db-sg` | VPC |
| Create the DB subnet group | RDS |
| Create the database | RDS |
| Read the secret's address | Secrets Manager |
| Create the policy and attach it to the role | IAM |
| Start and stop the servers | EC2 |
| Create the NAT gateway, edit the route table, release the Elastic IP | VPC |

## How the services connect

- The DB subnet group points to two private subnets (RDS to VPC).
- The database is attached to `db-sg` (RDS to VPC).
- `db-sg` allows `private-sg`, which the app server uses (VPC to EC2).
- The database password is stored in Secrets Manager, and KMS encrypts it.
- The IAM policy allows reading that one secret, and the role carries the policy (IAM to Secrets Manager).
- The app server wears the role, so it can read the secret (IAM to EC2).
- The NAT gateway gives the app server a way out to reach Secrets Manager (VPC to EC2).

## Problems I fixed
1. **`ValidationException: Invalid name`:** I had pasted the secret's address with extra characters.
   I printed it with `echo "[$DB_SECRET_ARN]"` (the brackets show hidden spaces), copied the ARN again from the
   console, and set it inside single quotes.
2. **Choosing the database size:** the first list showed large production classes. I switched to burstable
   classes and picked the micro size, because RDS charges by the hour.

## Security choices
- The database has **no public access** and sits in private subnets.
- `db-sg` allows only the app servers, on port 3306.
- The password lives in Secrets Manager, and is encrypted with KMS.
- The role can read only that one secret.
- User passwords are stored as hashes.

## Cleanup
Deleted the NAT gateway, released its Elastic IP, stopped both servers, and stopped or deleted the database,
because RDS and the NAT gateway charge by the hour.

## Screenshots
**Database available, public access off**
![Database](screenshots/p4-01-db-available.png)

**Database firewall rule**
![db-sg](screenshots/p4-02-db-sg.png)

**Secret in Secrets Manager (value not shown)**
![Secret](screenshots/p4-03-secret.png)

**App server connected to the database**
![Connected](screenshots/p4-04-db-connected.png)

**Sign up page**
![Sign up](screenshots/p4-05-signup.png)

**Duplicate email rejected**
![Duplicate email](screenshots/p4-06-duplicate-email.png)

**Player after login**
![Player](screenshots/p4-07-player-after-login.png)
