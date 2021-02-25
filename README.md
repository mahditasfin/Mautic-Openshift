## Setting up Mautic on Openshift

To Run:
### Using Argo


## CI/CD Argo

To build and deploy in the tools namespace using the argo pipeline, use the following command:

```argo submit argo/mautic.build.yaml -p GIT_REF=<branch-name> -p GIT_REPO=<git-repo> -p  NAMESPACE=<tools-namespace> -p APP_NAME=<app-name> -p IMAGE_TAG=3.1.2 -p STORAGE_CLASS_NAME=<storage-class-name>```

- Example: ```argo submit argo/mautic.build.yaml -p GIT_REF=clean-state -p GIT_REPO=https://github.com/bcgov/mautic-openshift -p  NAMESPACE=de0974-tools -p APP_NAME=mautic -p IMAGE_TAG=3.1.2 -p STORAGE_CLASS_NAME=netapp-file-standard -p DATABASE_USER=mautic_db_test -p DATABASE_USER_PASSWORD=password -p DATABASE_ROOT_PASSWORD=password2```

### Using manual commands
1. **Create the network security policy**
    ```oc process -f ./openshift/nsp.yaml -p NAMESPACE=<namespace> | oc apply -f -```

    - Example: ```oc process -f ./openshift/nsp.yaml -p NAMESPACE=de0974-tools | oc apply -f -```

2. **Process and apply the mariadb secret.yaml**

    Secret values are generated if not passed in: ```oc process -f ./openshift/secret.yaml -p NAME=<name> | oc apply -f - -n <namespace>```

    - Example: ```oc process -f ./openshift/secret.yaml -p APP_NAME=mautic | oc apply -f - -n de0974-tools```

    To assign custom values to the parameters, use the -p flag. The parameters can only contain alphanumeric and underscore characters:
    ```oc process -f ./openshift/secret.yaml -p NAME=<name> -p DATABASE_USER=<database-user-name> -p DATABASE_NAME=<database-name> -p DATABASE_USER_PASSWORD=<database-user-password> -p DATABASE_ROOT_PASSWORD=<database-root-password> | oc apply -f - -n <namespace>```

    - Example: ```oc process -f ./openshift/secret.yaml -p APP_NAME=mautic -p DATABASE_USER=mautic_db_test -p DATABASE_USER_PASSWORD=password -p DATABASE_ROOT_PASSWORD=password2 | oc apply -f - -n de0974-tools```

3. **Process and apply the mautic.yaml**
    ```
        oc process -f ./openshift/mautic.yaml \
        -p APP_NAME=<app-name> \
        -p GIT_REF=<git-branch> \
        -p GIT_REPO=<git-repo> \
        -p NAMESPACE=<namespace> \
        -p STORAGE_CLASS_NAME=<storage-class-name> \
        -p IMAGE_TAG=3.1.2 \ |
        oc apply -f - -n <namespace>

    ```

    - Example: ```oc process -f ./openshift/mautic.yaml -p APP_NAME=mautic -p GIT_REF=main -p GIT_REPO=https://github.com/bcgov/mautic-openshift -p NAMESPACE=de0974-tools -p STORAGE_CLASS_NAME=netapp-file-standard -p IMAGE_TAG=3.1.2 | oc apply -f - -n de0974-tools```

4. **Rollout the database and app**

    ```oc rollout latest dc/<app-name>-db -n <namespace> && oc rollout latest dc/<app-name> -n <namespace>```

    - Example: ```oc rollout latest dc/mautic-db -n de0974-tools && oc rollout latest dc/mautic -n de0974-tools```
    
## Setting up Mautic

1. Go to the Mautic Deployment route. This will lead you to the Mautic Installation - Environment Check page. 
The installer may suggest some recommendations for the configuration. Carefully review these recommendations and go to the next step.

2. On the Mautic Installation - Database Setup page, the required input should be pre-filled for you. Go to the next page.

3. On the Mautic Installation - Administrative User page, create the admin user as required.

4. On the Mautic Installation - Email Configuration page, select "Other SMTP Server" as the Mailer Transport.
To use the government server, use the following values:
- Server: apps.smtp.gov.bc.ca
- Port: 25
- Encryption: None
- Authentication mode: Login
- Username: firstname.lastname
- Password: Login Password

To use Gmail, use the following values:
- Server: smtp.gmail.com
- Port: 587
- Encryption: TLS
- Authentication mode: Login
- Username: Gmail Username
- Password: Gmail Password

Additionally, you may need to configure your security settings in Gmail to turn on "Less secure app access" at https://myaccount.google.com/security as well as turn on "Display Unlock Captcha" at https://accounts.google.com/b/0/DisplayUnlockCaptcha.

5. To allow emails to be sent out to contacts, you must change the Frequency Rule within Mautic.
    On the top right of the page, you will see a cog for the settings. Go to Configuration -> Email Settings.
    Scrolling down, you will see the "Default Frequency Rule". This number will be the maximum number of emails that can be sent to a user in the given time period. Setting this number to a reasonable value will help prevent unintentional email spamming.

## Mautic Workflow

### Segment
In Mautic, an email distribution list is called a `segment`. A segment can easily be created in the `Segments` tab by giving it a name.

### Form
Forms allow users to subscribe/unsubscribe themselves using the Mautic Subscription App. For each segment two forms should be created: subscribe and unsubscribe.

When creating a form it is important that the `Successful Submit Action` is set to Redirect URL and that the Redirect URL/Message is set to https://<mautic-subscription-app-url>/subscribed for the subscribed form and https://<mautic-subscription-app-url>/unsubscribed for the unsubscribe form.

- Example: https://mautic-subscription.apps.silver.devops.gov.bc.ca/subscribed and https://mautic-subscription.apps.silver.devops.gov.bc.ca/unsubscribed

Under the `Fields` tab, a new `Email` field should be created. 

Under the `Actions` tab, a new submit action to `Modify contact segments` should be created. You can choose to `Add contact to selected segment(s)` for the subscribe form or `Remove contact from selected segment(s)` for the unsubscribe form.

### Email
A `New Segment Email` can be set up under the `Channels` tab. For a basic layout the Blank theme can be used. The contents of the email can be set in the `builder`.
