# Setting up postfix and mbsync for Outlook365 and Google emails

For old-fashioned people like me, being used to mutt for now 30 years,
and also for those who want to have emails locally for reading them
while travelling, the following describes ONE WAY how to set up email
retrieval and email sending.

For email retrieval we use isync/mbsync. For email delivery we assume a postfix installation.

All the following is only necessary when you have second factor authentication
enabled, which you do have, right?

If you ONLY want to make your Google/GSuite email address available, then the
easiest way is NOT to install anything XOAUTH related, create app passwords
for mbsync and postfix, and use them in postfix’ `sasl_password` and mbsync’s `.mbsyncrc`.

But if you configure postfix for using XOAUTH2, then app passwords do NOT work
for postfix and the Google account, because postfix sasl will try xoauth2 and fail,
and will not try normal password authentication it seems!

## Preliminaries

All of the following has been repeatetly tested on my Arch Linux installations.
I cannot guarantee that the same works for different distributions, nor at any
time in the future, Microsoft and Google will for sure do more tricks with OAUTH in the future.

## Install sasl-xoauth

There are several possibilities for SASL XOAUTH2 providers:

- [cyrus-sasl-xoauth2](https://github.com/moriyoshi/cyrus-sasl-xoauth2)
- [oauth2ms](https://github.com/harishkrupo/oauth2ms)
- [sasl-xoauth2](https://github.com/tarickb/sasl-xoauth2)

I have managed to set up email retrieval with both cyrus sasl as well as
oauth2ms, but only with sasl-xoauth2 I was able to set up both, email
retrieval and email delivery.

Unfortunately, we cannot select different XOAUTH SASL providers for
different programs (at least as far as I see), that means we have to
use the same provider for both retrieval and delivery.

Clone [https://github.com/tarickb/sasl-xoauth2](https://github.com/tarickb/sasl-xoauth2) and do

```
mkdir build
cd build
cmake .. -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_INSTALL_SYSCONFDIR=/etc
make
sudo make install
```
You also need to install msal package, on Arch Linux this can be done with:

```
yay -S sasl-xoauth2-git python-msal
```
(both are maintained by me).

After that edit `/etc/sasl-xoauth2.conf` so that it contains

```
{
  "client_id": "",
  "client_secret": ""
}
```

The default contains some dummy strings, which is NOT GOOD!

After installation of this sasl plugin, the command pluginviewer
can be used to check that sasl-xoauth2 is the prefered XOAUTH2
authentication method. The output should contain:


```
Plugin "sasl-xoauth2" [loaded],         API version: 4
        SASL mechanism: XOAUTH2, best SSF: 60
        security flags: NO_ANONYMOUS|NO_PLAINTEXT|PASS_CREDENTIALS
        features: WANT_CLIENT_FIRST|PROXY_AUTHENTICATION
```
and this entry should have the highest SSF value from all other
XOAUTH2 SASL mechanism providers. There might by other entries from
cyrus-sasl-oauth and KDE kdexoauth2, but both with SSF: 0.


## Outlook365

### Postfix setup

#### create oauth apps for postfix

we create two different apps, since sharing the tokens between the postfix
and user account is difficult.

- Go to: [https://entra.microsoft.com/](https://entra.microsoft.com/)
- click on `Applications > App registrations`
- click on `New registration`

First lets do SMTP token, you can choose the name arbitrarily, eg `postfix-smtp`

For `Supported account types`, use

```
Accounts in the organizational directory only (<YOUR ORG>)
```

Do NOT put in Redirect URL information

Click on `Register` and after the creation you are in the details screen of the
application.

- Click on `Authentication` and on this page under `Advanced settings`, select
  `Allow public client flows: YES`.
- Next, click on `Save`.
- Next click in the (second level, app specific) sidebar on `API permissions`,
  then on `Add a permission`, then on `Microsoft Graph`, then on `Delegated permission`,
  then search for or scroll to `SMTP` and add `SMTP.Send`.
- Next, click on `Add permissions`, then click in the (second level, app specific)
  sidebar on `Overview` and copy the following information

```
Application (client) ID
Directory (tenant) ID
```
as `client_id` and `tenant_it`. Also we assume that `your_email` contains your
Outlook based email.

#### Obtain initial token

In the following we assume that the strings

```
${client_id}
${tenant_id}
${your_email}
```
are known. These strings need to be replaced in the following code.
Sometimes this can work as shell env vars, but sometimes not, be careful.

Run

```
sasl-xoauth2-tool get-token outlook \
     token.${your_email} \
     --client-id=$client_id \
     --tenant=$tenant_id \
     --use-device-flow
```
This will tell you to open [https://microsoft.com/devicelogin](https://microsoft.com/devicelogin) and give you a code,
enter the code on the above web page, select the your email, and accept
permissions for your app.

After that, the above command should terminate with

```
Acquired token.
```
The file `token.${your_email}` should have been generated with a json dict with keys
```
    access_token
    refresh_token
    expiry
```
Next, **add** to this file as additional keys, replacing `$client_id` and `$tenant_id`
with the respective values

```
    "client_id": "${client_id}",
    "client_secret": "",
    "token_endpoint": "https://login.microsoftonline.com/${tenant_id}/oauth2/v2.0/token"
```
be aware of final `,` to be added to the last element before adding these new entries.

Why? This is necessary to support *multiple* providers. In case one wants
to add for example the some Google email provider, one needs to use different
settings (see below), and the `client_id`, `client_secret`, and `token_endpoint`
cannot be in the main config file `/etc/sasl-xoauth2.conf`!

After that you can do:

```
sasl-xoauth2-tool test-token-refresh token.${your_email}
```
it should respond with

```
    Config check passed.
    Token refresh succeeded.
```

#### Postfix configuration

We will put the tokens into `/etc/sasl-tokens` which will be owned by
`postfix`, since the SASL code in postfix will change these tokens.

As root do
```
    mkdir /etc/sasl-tokens
    mv token.${your_email} /etc/sasl-tokens/${your_email}
    chown -R postfix:postfix /etc/sasl-tokens
    chmod -R go-rwx /etc/sasl-tokens
```
In `/etc/postfix/sasl_password` (always replace `${your_email}` with the actual email!):

```
${your_email}    ${your_email}:/etc/sasl-tokens/${your_email}
```
The first is the key that is used also in the next file, the second
`your_email` is the username to be used, the last one is the file
containing the token.

In `/etc/postfix/sender_relay`, add
```
${your_email}       [smtp.office365.com]:587
```

In `/etc/postfix/main.cf`

```
## sending out emails
smtp_sender_dependent_authentication = yes
sender_dependent_relayhost_maps = hash:/etc/postfix/sender_relay
smtp_sasl_auth_enable = yes
smtp_tls_security_level = encrypt
smtp_sasl_tls_security_options = noanonymous
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
# sasl xoauth2 for office365
smtp_sasl_mechanism_filter = xoauth2,login
smtp_sasl_security_options =
```

After that run
```
    postmap /etc/postfix/sender_relay
    postmap /etc/postfix/sasl_password
```
Then
```
    systemctl restart postfix
```
With that, *sending* email with `From: ${your_email}` should work, try and follow with
```
    journalctl -f -u postfix
```


### MBSync setup

The procedure is very similar to the above, only the final token location is different.

#### create oauth apps for mbsync

Do the same as above, but use a different name, eg
```
    mbsync-imap
```
When selection API permissions, instead of using the `SMTP`, choose:
```
    IMAP.AccessAsUser.All
    User.ReadBasic.All
```
(maybe `User.Read` is already pre-selected?).

#### Obtain initial token

Follow the same procedure as above.

#### MBSync configuration

We will put the tokens into `$HOME/.tokens` which will has 0700 permissions.

As normal user do
```
    mkdir $HOME/.tokens
    mv token.${your_email} $HOME/.tokens/${your_email}
    chmod -R go-rwx $HOME/.tokens
```

For the `.mbsyncrc` file, parts that can go into the respective
`IMAPStore` directive:

```
IMAPStore xxxxxxxxx
Host outlook.office365.com
User ${your_email}
PassCmd "echo ~/.tokens/${your_email}"
SSLType IMAPS
AuthMechs XOAUTH2
```
(as usual, replacing the `${your_email}` with the actual email.

With that in place,
```
    mbsync -V ......
```
should work. Details about setting up mbsync are omitted here.


## Google GSuite

### Postfix setup

#### create oauth apps for postfix

- go to [https://console.cloud.google.com/](https://console.cloud.google.com/)
- if necessary, switch to your Google email
- in the left top next to `Google Cloud` is the project selector, click it
- if you already have a private project, select it, otherwise create a new project
  by clicking on `New Project`, give it a name and click on `Create`.
- After that is done, click on the Hamburger menu, select `APIs & Services > OAuth consent screen`, Fill in `App name`, `User support email`, `Developer contact information` and click on `Save`
- Click on the sidebar on `Credentials`, or `Hamburg > APIs & Services > Credentials`
- Click on `Create Credentials` and select `OAuth client ID`
- Select for `Application Type` the entry `Desktop app` and give it a name, then click on `Create`
- on the next screen you see `client_id` as `Client ID`, `client_secret` as `Client secret`, make sure to keep them somewhere.

#### Obtain initial token

In the following we assume that the strings
```
    ${client_id}
    ${client_secret}
    ${google_email}
```
are known. These strings need to be replaced in the following code. Sometimes this
can work as shell env vars, but sometimes not, be careful.

Run
```
  sasl-xoauth2-tool get-token gmail \
     token.${google_email} \
     --client-id=${client_id} \
     --client-secret=${client_secret} \
     --scope="https://mail.google.com/" 
```
This will tell you to open a long URL 
`https://accounts.google.com/o/oauth2/auth…`. When you go there, select 
your Google email account and agree to the permissions.

After that, the above procedure is finished, the program terminates, and
the file `token.${google_email}` should have been generated with a json
dict with keys
```
    access_token
    refresh_token
    expires_in
    scope
    token_type
```
Next, **add** to this file as additional keys, replacing `${client_id}` and
`${client_secret}` with the respective values
```

    "client_id": "${client_id}",
    "client_secret": "${client_secret}"
```
be aware of final `,` to be added to the last element before adding these new entries.

See above for why this is necessary.

After that you can do:
```
    sasl-xoauth2-tool test-token-refresh token.${google_email}
```
it should respond with
```
    Config check passed.
    Token refresh succeeded.
```

#### Postfix configuration

As above, just replace ${your_email} with ${google_email}.
For `/etc/postfix/sender_relay` use
```
${google_email}       [smtp.gmail.com]
```

### MBSync setup

For the Google email there are two options: The simple one uses an app
password for `mbsync`, which can be set from your account setup, or
the xoauth2 approach as before.

For the simple method I use the keyring program with the secret-service
backend, that gets unlocked with my KDE/Plasma session. I store the app password with
```
keyring -b keyring.backends.SecretService.Keyring set offlineimap ${google_email}
```
and use the following in `.mbsyncrc`:
```
IMAPStore google-remote
Host imap.gmail.com
User ${google_email}
PassCmd "keyring -b keyring.backends.SecretService.Keyring get offlineimap ${google_email}"
SSLType IMAPS
AuthMechs PLAIN
```

For the XOAUTH2 approach, the flow is very similar to the above:

#### Oauth app

There is no need to generate another oauth app, the `${client_id}` and
`${client_secret}` can be used as before

#### Obtain initial token

Follow the same procedure as above.

#### MBSync configuration

We will put the tokens into `$HOME/.tokens` which will has 0700 permissions.

As normal user do
```
    mkdir $HOME/.tokens
    mv token.${google_email} $HOME/.tokens/${google_email}
    chmod -R go-rwx $HOME/.tokens
```
For the `.mbsyncrc` file, parts that can go into the respective `IMAPStore` directive:
```
IMAPStore xxxxxxxxx
Host imap.gmail.com
User ${google_email}
PassCmd "echo ~/.tokens/${google_email}"
SSLType IMAPS
AuthMechs XOAUTH2
```
(as usual, replacing the ${google_email} with the actual email.

With that in place,
```
    mbsync -V ......
```
should work. Details about setting up mbsync are omitted here.


## TODO items

- maybe use the same oauth app for imap and smtp on outlook, just create one app with all permissions, but separate initial token creation?

- maybe put postfix tokens into /var since they change regularly and can be recreated?

