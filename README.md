# Companion Script: FCLI with SSC and ScanCentral SAST

This document contains all the commands demonstrated in the corresponding YouTube video on the "Fortify Unplugged" channel. It is intended to serve as a companion resource, making it easier to follow along and try out the commands in your own environment.

**Watch the video here:** `[Link to the video here]`

**A quick note on compatibility**: The commands in this script are shown as executed on macOS. While most fcli commands are cross-platform, shell syntax—especially for setting environment variables (e.g., export) and handling quotes ('single quotes' vs. "double quotes")—can differ on other operating systems like Windows. You may need to make minor adjustments for the commands to work in your specific shell (like Command Prompt or PowerShell).

-----

### Setting Environment Variables

To simplify the reuse of values like the SSC URL or passwords, you can set environment variables. `fcli` automatically recognizes these and uses them as default values for the corresponding command options.

**Note:** The `export` command is for Linux/macOS (bash/zsh). The syntax for Windows may differ (`set` or `$env:` in PowerShell).

```bash
# Sets the default URL for connecting to the Software Security Center (SSC)
export FCLI_DEFAULT_SSC_URL='https://your-ssc.com'

# Sets additional variables for the password and client authentication token
export FCLI_DEFAULT_SSC_PASSWORD='your-password-here'
export FCLI_DEFAULT_SSC_CLIENT_AUTH_TOKEN='your-client-auth-token-here'

# Verifies that the variable has been set correctly
echo $FCLI_DEFAULT_SSC_URL
```

-----

### Session Management

`fcli` is session-based. Before you can interact with SSC or ScanCentral, you must establish a session.

#### Login with User Credentials

The most straightforward way to log in is by using a username and password. Thanks to the environment variables we set earlier, we don't need to specify the URL and password again.

```bash
# Establishes a session to SSC, providing only the username
# The URL, password, and Client Auth Token are sourced from the environment variables
fcli ssc session login -u jwienand
```

#### Login with a Token (Best Practice for CI/CD)

For automated environments, using tokens instead of passwords is the recommended best practice. `fcli` can create tokens and use them for login.

```bash
# Creates an "AutomationToken" that is valid for one day and stores it
# in the fcli-internal variable named "my_token"
fcli ssc access-control create-token AutomationToken --description="FortifyUnplugged" --expire-in=1d -u jwienand --store=my_token

# Uses the previously stored token to log in
fcli ssc session login -t ::my_token::restToken
```

-----

### Working with SSC (Software Security Center)

Once a session is established, you can perform various actions on the SSC.

#### Managing Applications and Versions

Listing, filtering, and creating applications are central tasks.

```bash
# Lists all applications on the SSC
fcli ssc app ls

# Lists all application versions
fcli ssc av ls

# Filters the list to show only versions of the "Logistics" application
fcli ssc av ls -q 'application.name=="Logistics"'

# Outputs a list of commands to purge artifacts older than 30 days
# for all application versions
fcli ssc av ls -o 'expr=fcli ssc av purge-artifacts {id} --older-than 30d\n'

# Creates a new application version.
# --skip-if-exists: Skips creation if the version already exists.
# --auto-required-attrs: Fills in all required attributes automatically.
fcli ssc av create pizza:v3 --skip-if-exists --auto-required-attrs --issue-template "Prioritized High Risk Issue Template"
```

#### Issues and Reports

Get quick overviews of issues or generate reports.

```bash
# Counts the issues by priority for a specific application version
fcli ssc issue count -av RWI:1.0

# Generates a new PDF report based on a template (here, with ID 3)
fcli ssc report create -f pdf -n 'Fortify Unplugged Demo' --template 3 -p "Application Version RWI:1.0"
```

#### Users and Roles

Manage users directly from the command line.

```bash
# Lists all users and roles
fcli ssc access-control list-users
fcli ssc access-control list-roles

# Creates a new local user with the "developer" role
fcli ssc access-control create-local-user --email=fortify@unplugged.de --firstname=Fortify --lastname=Unplugged --username=fortifyunplugged --roles developer
```

#### `fcli ssc rest-call`

A powerful command to interact with any SSC REST API endpoint that may not have a dedicated `fcli` function. `fcli` handles the authentication automatically.

```bash
# Queries the API for details about the "developer" role
fcli ssc rest-call -X GET "/api/v1/roles?q=name:developer"
```

#### SSC Actions

Actions are predefined workflows (YAML files) that simplify complex tasks. They are ideal for CI/CD pipelines.

```bash
# Lists all available actions
fcli ssc action ls

# Displays the definition of the "check-policy" action
fcli ssc action get check-policy

# Runs a security policy check for an application version
fcli ssc action run check-policy --av RWI:1.0

# Generates a GitLab-compatible SAST report
fcli ssc action run gitlab-sast-report --av RWI:1.0
cat gl-fortify-sast.json
```

-----

### Working with ScanCentral SAST

The existing SSC session is also used for interacting with ScanCentral SAST.

#### Managing the ScanCentral Client

The `tool` command can be used to manage the ScanCentral Client itself, which is especially useful when setting up build agents.

```bash
# Updates the tool definitions
fcli tool definitions-update

# Installs the ScanCentral Client
fcli tool sc-client install

# Starts the packaging process for the code (without build tool integration)
fcli tool sc-client run package -bt none -o package.zip
```

#### Starting and Monitoring Scans

The core process: package the code, start a scan, and wait for the results.

```bash
# Use an action to quickly prepare new app versions for scanning
fcli ssc action run setup-appversion --av pizza:v4
fcli ssc action run setup-appversion --av pizza:v5

# Start two scans in parallel for different versions.
# --store saves the job ID in an fcli variable for later use.
fcli sc-sast scan start -f package1.zip --publish-to pizza:v4 --store scan1
fcli sc-sast scan start -f package2.zip --publish-to pizza:v5 --store scan2

# Wait until both previously started scans are complete
# by referencing the stored variables.
fcli sc-sast scan wait-for ::scan1:: ::scan2::
```
