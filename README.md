# Creating the First Admin

## Problem Statement / Objective

Our applications will often have two broad types of users:

1. Regular users \- Users who perform the actions the application is primarily designed to support.
2. Administrators \- Users with additional privileges, such as modifying other users' data or status, managing protected resources, and configuring the application.

Regular users can usually create their accounts through the normal signup flow. However, allowing anyone to select an administrator role during signup would grant privileged access to unauthorized users, which can have unintended consequences. A separate admin signup page does not prevent this by itself either. Nonetheless, we still need a way to create admin accounts for authorized users.

One approach we can think of is letting admins invite or promote other admins. This makes sense because an admin, by definition, has the power and knowledge to do such things, and it’s a model we’re familiar with in other systems such as Telegram group chats.

However, even this approach raises another problem.

**How are we creating the first admin(s)?**

This type of chicken-and-egg problem in software engineering is often called **bootstrapping,** derived from the phrase “pulling yourself up by your own bootstraps”.

We’ll examine the anti-pattern for admin bootstrapping first, and explore what alternatives there are and what you should look out for in your own admin bootstrapping.

## Anti-pattern: Ad Hoc Direct Database Modification

The easiest approach would be to insert an admin account or change an existing user's role directly in the database, using SQL or a database GUI.

Contrary to this section’s title, direct database modification is not inherently an anti-pattern. A reviewed, versioned script can be a valid bootstrap mechanism if it satisfies the application's requirements.

The concern is relying on ad hoc edits that bypass required account initialization, lack a clear audit trail, or are difficult to reproduce.

### Application Business Logic Bypass

Creating a user may involve more than inserting a single row. Depending on the application, it may require hashing a password, creating related records, assigning permissions, initializing a workspace, or scheduling verification and notification events.

When these requirements are implemented in application code, direct database edits can skip them. We must then understand and reproduce the required steps ourselves, which can leave an account in an inconsistent state if something is missed.

These inconsistencies can require investigation and repair. Database transactions can roll back uncommitted database changes, but they do not automatically undo external effects such as emails or API calls.

Therefore, our bootstrap process should reuse the appropriate account-provisioning logic and preserve its invariants.

### Poor Reproducibility and Auditing

Direct database edits may bypass application-level audit events. You can argue that database auditing or triggers can still capture changes, but those records may lack useful context, such as who authorized the admin role and why it was granted.

Manual edits are also difficult to reproduce consistently across development, staging, and production environments, which may lead to inconsistency and communication breakdown during QA or end-to-end testing in different environments.

### Other Concerns

Direct database access may be necessary for some maintenance or recovery tasks. However, making this the routine way to manage admins may be a slippery slope that encourages broader database permissions than operators need and increases the chance of mistakes.

## Alternative Solutions

### Environment Variables and Bootstrap Scripts

This is the most common and straightforward approach. On application startup, we can call an initialization function that checks for env variables containing the first admin data. Then the application reads the data and calls the necessary functions to create the admin and trigger the necessary side effects. A deployment job that reads this .env file can be created to do the same thing.

A standalone initialization script can also work, provided repeated and concurrent execution is handled safely. This can be as simple as a versioned database migration, or an API call to a protected admin creation endpoint.

Regardless of the approach you take, the bootstrap operation should be **idempotent**. Repeating the same operation should have the same effect as performing it once. Once initialization succeeds, subsequent runs should not create duplicate accounts, reset passwords, or restore deliberately revoked privileges.

We should also be wary of race conditions and concurrent executions. Two application instances may both observe that initialization is incomplete. Use appropriate concurrency mechanisms so only one bootstrap attempt succeeds without being affected by the other.

### First-Run Setup Wizard & Single-Use Initialization Token

A first-run setup wizard lets an authorized operator create the initial admin through the application, if the application detects that it's uninitialized.

Similarly, the application or the deployment process can generate a secure and random single-use initialization token. The operator retrieves this and submits through a setup page or endpoint, and the server then validates it and creates the first admin account using application logic.

Jenkins uses a mix of these approaches for new installations. It generates an initialization token that the operator retrieves from the installation and enters to unlock the wizard before proceeding to first-admin creation. See the [Jenkins setup documentation](https://www.jenkins.io/doc/book/installing/windows/#post-installation-setup-wizard).

A publicly accessible wizard that simply makes the first visitor an admin can be claimed by an attacker. To prevent this, require proof of operator access before allowing account creation. Likewise, the initialization token should be read by or delivered to the admin user through secured modes.

The points from the environment variables and startup script setups still stand here. After the setup wizard runs to completion or the single-use initialization token is used, they must then be invalidated so any further admin setup requests are correctly denied. It also shouldn’t create multiple admin accounts when multiple users simultaneously connect to an uninitialized application or use the initialization token.

### Framework Management Commands

Fortunately for us, sometimes the frameworks we choose may already provide an admin-creation command.

For example, Django provides `createsuperuser` to create an admin (superuser), which also supports automated execution with `--noinput`. In that mode, the required account details must be supplied through supported options or environment variables. See the [Django command documentation](https://docs.djangoproject.com/en/5.2/ref/django-admin/#createsuperuser).

One caveat is that a framework command can handle the framework's account model while still missing our application-specific setup. We should check that it performs the initialization our application requires, or wrap it in a command that calls the appropriate shared provisioning logic. We should also not assume that these commands are idempotent and concurrency-safe, and should check the documentation.

## Closing Remarks

These are not the only ways to create the first admin(s). Whichever approach you choose, make sure it meets the following requirements:

1. **Correct initialization** \- It creates the required account records and permissions and reliably handles necessary side effects.
2. **Controlled authorization** \- Only an authorized operator, deployment identity, or explicitly designated user can create the initial admin. Subsequent privilege grants are also authorized on the server.
3. **Reproducibility and auditability** \- The procedure is documented or automated, works consistently across environments, and records the provisioning action without exposing secrets.
4. **Safe repeated and concurrent execution** \- Repeated attempts do not duplicate accounts, reset credentials, or restore revoked privileges, and simultaneous attempts cannot both complete bootstrap provisioning.
5. **Secure credential handling** \- Credentials are securely protected, temporary secrets are invalidated after use, and assigned temporary passwords are replaced on first login.
