# CI/CD in Jenkins
- [CI/CD in Jenkins](#cicd-in-jenkins)
  - [Galaxy Server Update](#galaxy-server-update)
  - [Usegalaxy.it Tools Update](#usegalaxyit-tools-update)
    - [Install/Update Tools](#installupdate-tools)
    - [Push Reports](#push-reports)
  - [Email Notifications Guide](#email-notifications-guide)
  - [References](#references)
  - [Author Information](#author-information)

## Galaxy Server Update

Jenkins job executed every night at ~ 4:00 CET.  
It runs ansible playbook [`sn06.yml`](https://github.com/usegalaxy-it/infrastructure-playbook/blob/master/sn06.yml) from [infrastructure-playbook](https://github.com/usegalaxy-it/infrastructure-playbook) repository. The playbook updates Galaxy server and runs all the included roles. 

The configuration overview of the `usegalaxy.it-sn06` Jenkins job:

General:
- Discard old builds:
  - Strategy: Log Rotation
    - Max # of builds to keep: `7`

Source Code Management:
- Git:
  - Repository URL: `https://github.com/usegalaxy-it/infrastructure-playbook`
  - Branches to build: `*/master`

Build Triggers:
- Build periodically: 
  - Schedule: `H 2 * * *` (every night at ~2:00 UTC == ~4:00 CET, minutes are set randomly)

Build Environment:
- Delete workspace before build starts
- Use secret text(s) or ﬁle(s):
  - Secret file:
    - Variable: `<ANSIBLE_VAULT_PASS>`
    - Credentials: `<vault_pass>`
  
Build Steps:
- Virtualenv Builder: 
  - Python version: `Python-3.8.10`
  - Nature: `Shell`
  - Command:
```bash
python -m pip install -r requirements.txt
ansible-galaxy install -r requirements.yaml
cp $VAULT_PASS .vault_password
ansible-playbook -i hosts sn06.yml --extra-vars "__galaxy_dir_perms='0755'"
rm -d .vault_password
```

Post-build Actions:
- E-mail Notiﬁcation:
  - Recipients: `<admin1@gmail.com>, <admin1@gmail.com>`
  - Send e-mail for every unstable build


## Usegalaxy.it Tools Update

Jenkins jobs run every night at ~ 3:00 CET.  
`usegalaxy.it-tools` installs and updates tools, saving the logs and updating tools repo with new lockfiles. It sends an email to a specified list of recipients in case of job failure.  
`usegalaxy.it-tools-reports` job runs if the previous job is successful and transforms logs into the Markdown report, saving it to github repo. 

### Install/Update Tools

The configuration overview of the `usegalaxy.it-tools` Jenkins job:

General:
- Discard old builds:
  - Strategy: Log Rotation
    - Max # of builds to keep: `7`

Source Code Management:
- Git:
  - Repository URL: `git@github.com:<tools_repo>.git`
  - Credentails: `<SSH_KEY>` 
  - Branches to build: `*/main`

Build Triggers:
- Build periodically: 
  - Schedule: `H 1 * * *` (every night at ~1:00 UTC == ~3:00 CET, minutes are set randomly)

Build Environment:
- Delete workspace before build starts
- Use secret text(s) or ﬁle(s):
  - Secret text:
    - Variable: `<API_KEY>`
    - Credentials: `<API_KEY>`
  
Build Steps:
- Virtualenv Builder: 
  - Python version: `Python-3.8.10`
  - Nature: `Shell`
  - Command:
```bash
git checkout main
pip install -r requirements.txt
# create lockfiles
make fix
# install tools
make install GALAXY_SERVER=$GALAXY_ADDR GALAXY_API_KEY=$GALAXY_API_KEY
# commit lockfiles on github, allow to fail if no changes
git add *.yaml.lock
git commit -m "Update lock files. Jenkins Build: ${BUILD_NUMBER}" -m "https://github.com/usegalaxy-it/usegalaxy.it-tools-reports/blob/main/reports-$(date +%Y-%m)/$(date +%Y-%m-%d-%H-%M)-tools-update.md" || true
```

Post-build Actions:
- Archive the artifacts:
  - Files to archive: `report.log`
- Git Publisher:
  - Push Only If Build Succeeds.
  - Branches:  
    - Branch to push: `main`
    - Target remote name: `origin`
- E-mail Notiﬁcation:
  - Recipients: `<admin1@gmail.com>, <admin1@gmail.com>`
  - Send e-mail for every unstable build

### Push Reports

The configuration overview of the `usegalaxy.it-tools-reports` Jenkins job:

General:
- Discard old builds:
  - Strategy: Log Rotation
    - Max # of builds to keep: `5`

Source Code Management:
- Git:
  - Repository URL: `git@github.com:<reports_repo>.git`
  - Credentails: `<SSH_KEY>` 
  - Branches to build: `*/main`

Build Triggers:
- Build after other projects are built: 
  - Projects to watch: `usegalaxy.it-tools`
  - Trigger even if the build is unstable.

Build Environment:
- Delete workspace before build starts

Build Steps:
- Virtualenv Builder: 
  - Python version: `Python-3.8.10`
  - Nature: `Shell`
  - Command:
```bash
git checkout main
# make dir for each month
mkdir reports-$(date +%Y-%m) || true
# name of report file
POST="$(date +%Y-%m-%d-%H-%M)-tools-update.md"
# transform log file to markdown report
cat <absolute_path_to_workspace_dir_of_usegalaxy.it-tools>/report.log | python <absolute_path_to_workspace_dir_of_usegalaxy.it-tools>/scripts/generate-report.py > reports-$(date +%Y-%m)/${POST}
# commit report
git add .
git commit -m "Jenkins Build: ${BUILD_NUMBER}"
```

Post-build Actions:
- Git Publisher: A post-build action that deals with pushing changes to remote repositories.
  - Push Only If Build Succeeds.
  - Branches:  
    - Branch to push: `main`
    - Target remote name: `origin`

 
## Email Notifications Guide

Email notifications are enables by `mailer` Jenkins plugin.  
To send email notifications from gmail account, some steps need to be performed with gmail account and Jenkins settings. They are generally described below:  

Jenkins mailer setup:
1. In the Gmail account
   - Log in to the Gmail account.
   - Go to Settings 
   - “How you sign in to Google” → enable and configure 2 step verification
   - When it’s enabled go again to “How you sign in to Google” → “2-step Verification”
   - Scroll down to “App passwords”
   - Generate application password (Other) and save it to use in Jenkins
2. In Jenkins
   - Log in to Jenkins as an administrator.
   - Go to "Manage Jenkins" > "Configure System."
   - Scroll down to the "E-mail Notification" section.
   - Enter the following details:
       - SMTP server: `smtp.gmail.com`
       - Click “Advanced”
       - "Use SMTP Authentication": checked
       - User Name: `<the_gmail_address_you_configured>`
       - Password: `<application-specific_password_generated_in_step_1>`
       - "Use SSL": checked
       - SMTP Port: `465`
   - “Test configuration by sending test email”: checked
   - Enter the same email or your email address, where you can check the incoming messages 
   - Click “Test Configuration” in the bottom right corner 
   - Check if you got the message
3. In a Jenkins project:
   - Add post-build actions
   - "E-mail Notification"
   - Enter where to send notification
   - "Send e-mail for every unstable build": checked

Now you’ll get messages in case of failure, and when the project is back to normal

## References
- [Usegalaxy.it Tools Management](https://github.com/usegalaxy-it/documentation/blob/main/tools.md)

## Author Information

[Polina Khmelevskaia](https://github.com/po-khmel)