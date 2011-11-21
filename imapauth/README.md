# Introduction

The company I work for is small enough to not have a central user database like LDAP. So far, we get by on separate accounts and manual password policy configuration on each machine. Some of us are command-line comfortable, while some of us are not.

Recently, we needed to set up a [MoinMoin](http://www.moinmo.in) installation. To lock it down, and to track who writes what, we needed some authentication. Keeping another login/password combination would be inconventient, so I started thinking about a way to reuse an existing one. There was not much to choose from. Everyone had an account on their computer, and everyone had an email account on an externally managed mailhost. The only viable alternative would be to authenticate against the IMAP server.

I had just finished another web app installation on Apache 2.2 that used PAM authentication with [mod_authnz_external and pwauth](http://code.google.com/p/pwauth/wiki/InstallApache). It struck me that pwauth just reads two lines from STDIN, does it's thing and exits with the appropriate exit code. It would be simple to create a script to replicate that, but to instead validate the cridentials against our IMAP server.

A quick [python script](imapauth) later, I got it working. To not hammer the IMAP server, I added a cache that stores the SHA512:ed password for an hour. The code is straightforward, and should be easy to understand.

# Installation

Copy `imapauth` to e.g. `/usr/local/bin` and edit the `SERVER`, `PORT` and `SSL` variables as needed. If you need to authorize the user, e.g. when using a generic third party IMAP provider like gmail, add this a few sections down (see comments). `chmod` it appropriately, e.g. `chmod 755`.

In your [mywiki.py MoinMoin config file](http://moinmo.in/HelpOnAuthentication#given_by_REMOTE_USER_environment_variable), set `auth = [GivenAuth(autocreate=True)]`. The user will be created and logged in with whatever user name (s)he uses to log in to the IMAP server. On some system, this is only the username, in others it may be the full email address. If you know how to set up your mail client, you know which it is.

To [configure Apache2](http://code.google.com/p/pwauth/wiki/InstallApache), install `mod_auth_external` add the following for your VirtualHost (this assumes that you put the `imapauth` script in `/usr/local/bin`):

    AddExternalAuth imapauth /usr/local/bin/imapauth
    SetExternalAuthMethod imapauth pipe

Now, configure your MoinMoin Directory as below. (Adjust for your installation directories as needed).

	# MoinMoin wiki

	Alias /moin_static/ "/usr/share/moin/htdocs/"
	ScriptAlias /moin "/var/www/moin/cgi-bin/moin.cgi"

	<Directory /var/www/moin/>
		AuthType Basic
		AuthName "Authentication Required"
		AuthBasicProvider external
		AuthExternal imapauth
		Require valid-user

		Options Indexes FollowSymLinks MultiViews
		AllowOverride None
		Order allow,deny
		allow from all
		Options +ExecCGI
	</Directory>

Restart Apache and try it out.

# Dependencies

* Python 2.5+, or Python 3.0+

For Apache:

* `mod_auth_external`

# Closing

While writing this summary, I stumbled on [checkpasswd-imap](http://www.namazustudios.com/blog/checkpasswd-imap-a-mod_authnz_external-style-password-checker/) which seem to do the same and more. If your want something more than a quick hack, you should probably check that out. There's also a [pam-imap](http://pam-imap.sourceforge.net/) if that's what you're looking for.
