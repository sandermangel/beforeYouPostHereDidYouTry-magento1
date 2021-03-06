# beforeYouPostHereDidYouTry - Magento 1.x
This is a list of points you should check before posting your issue online. We've run into these issues before and know what the standard issues might be so please take the time to work them through

## Table of contents
1. [Basic debugging steps](#basic-debugging)
2. [Emails not sending](#emails-not-sending)
3. [Product images not displaying](#productimages-not-displaying)
4. [Cronjob not running](#cronjob-not-running)
5. [Applying patches without SSH](#applying-patches-without-ssh)
5. [Form field validation](#form-field-validation)

-----------------------

## <a name="basic-debugging"></a>Basic checks when running into any issue
Thanks to an excellent answer by Ben Lessani (http://magento.stackexchange.com/a/429/50 < go upvote it) we have a basic checklist for any bug you run into.

###Logging
Always check the Magento logs in `var/log/...`, also make sure it's turned on under `System > Configuration > Developer > Log Settings` and check the PHP and other related logs which are probably either located in your webshops root folder or in the `/var/logs/` folder on the server.

Also check your developer console for any error logging on the frontend. It might be a javascript issue.

###Enable PHP Errors

This is key to most issues. For security or other reasons, PHP error display could likely be disabled by default by your PHP configuration.

You can enable errors with a more permanent solution, or merely something more temporary.

####Permanent solution

**For Apache/mod_php users**

In your document root's `.htaccess` file - just drop this at the top.

    php_flag display_startup_errors on
    php_flag display_errors on
    php_flag html_errors on
    php_flag  log_errors on
    php_value error_log  /home/path/public_html/var/log/system.log

**For Nginx/FastCGI users**

In your Nginx virtualhost configuration, in either the final `location .php {` directive, or in the `fastcgi_params` file (if you have one specified)

    fastcgi_param PHP_VALUE  display_startup_errors=on;
    fastcgi_param PHP_VALUE  display_errors=on;
    fastcgi_param PHP_VALUE  html_errors=on;
    fastcgi_param PHP_VALUE  log_errors=on;
    fastcgi_param PHP_VALUE  error_log=/home/path/public_html/var/log/system.log;

####Temporary/Universal solution

**For any platform**

Edit the Magento bootstrap `index.php` in your document root and uncomment the following line:

    ###ini_set('display_errors', 1);

---

###Enable Developer Mode

When you've had an error and suddenly hit the "Error Report" page, and been given a seemingly useless error string like `1184257287824` - you've got a few options.

####Permanent solution

**For Apache/mod_php users**

In your document root `.htaccess` file - just drop this at the top.

    SetEnv MAGE_IS_DEVELOPER_MODE true

**For Nginx/fastcgi users**

In your Nginx virtualhost configuration, in either the final `location .php {` directive, or in the `fastcgi_params` file (if you have one specified)

    fastcgi_param MAGE_IS_DEVELOPER_MODE true;

####Temporary/Universal solution

Edit the Magento bootstrap `index.php` in your document root and either make the `if` statement always true, or enabled for your specific IP.

    if (isset($_SERVER['MAGE_IS_DEVELOPER_MODE']) || true) {
      Mage::setIsDeveloperMode(true);
    }

or

    if (isset($_SERVER['MAGE_IS_DEVELOPER_MODE']) || $_SERVER['REMOTE_ADDR'] == 'my.ip.add.ress') {
      Mage::setIsDeveloperMode(true);
    }

---

###Check your permissions

Incorrect permissions will cause a wealth of problems, a lot of which are not that easy to find at first glance. 

> For example.  
> If PHP cannot write to the `./media` directory and you have JS combine enabled - Magento is unable to generate the combined file and associated unique URI for the media. So instead, what you'll find in your browser source code is a full server path to the media file
> `/home/path/public_html/media/xxx`

Otherwise, the site can appear to be functioning as normal - with no critical errors actually visible.

*Please bear in mind, this practice is secure for dedicated hosting but may present security issues with shared hosting if the Apache process isn’t chroot’ed per user.*

In our example, the SSH/FTP user is `sonassi`, the Apache user is `apache` and the group is `apache`

#####Add the FTP/SSH user to the Apache group

Most importantly, we need to make sure that the FTP/SSH user is part of the Apache group, in our example, its `apache` (but is also commonly `www-data`)

    usermod -a -G apache sonassi

*Keep adding as many users to the group as you have for FTP/SSH.*

#####Reset original permissions

So before we start, lets make sure all the permissions are correct.

    chown -R sonassi:apache /home/path/public_html/
    find /home/path/public_html/ -type d -exec chmod 750 {} \;
    find /home/path/public_html/ -type f -exec chmod 640 {} \;
    chmod -R g+w /home/path/public_html/{includes,media,var}

####Making the changes permanent
#####ACLs and Sticky Bits

ACLs in Linux allow us to define specific rules, in our case, what permissions files should inherit upon creation. A **sticky bit** (mentioned later) takes care of group inheritance, but does not help with the permissions, which is why we use ACLs.

Start by enabling ACL support on the active partition, **please ensure your Kernel was compiled with ACL support**.

*Your partition may be `/`, `/home`, `/var` or something else, replace as appropriate.*

    mount -o remount,acl /home

Now ACLs are enabled, we can set the ACL rules and group sticky bits:

    setfacl -R -d -m u::rwX,g::rwX,o::rX /home/path/public_html/
    chmod g+s /home/path/public_html/

#####But I don’t have ACL support

If your Kernel doesn’t support ACLs you can also use `umask` (which is a run time setting for BASH, FTP and PHP) to set the default file permissions. Magento usually sets `umask(0)` in `index.php`, however, it would be in your interests to change this.

In your `index.php` change the `umask` line to be

    umask(022);

And in your BASH environment for SSH, set this in either your `.bashrc` or `.bash_profile`

    umask 022

For your FTP server, you’ll need to read the documentation for it, but the principle is the same.


---

###Revert theme to default

Its possible that either your theme or package is responsible for this issue. Reverting back to a vanilla Magento theme is a quick way to find out. 

**This comes with the caveat that some modules may be dependent on certain theme features*

Rather than change anything via the admin panel, it is much simpler to merely rename the offending directories.

**Via SSH**

    mv ./app/design/frontend/myBrokenTheme{,.tmp}
    mv ./skin/frontend/myBrokenTheme{,.tmp}

Or via your FTP client, traverse and rename your package to something else. eg. `myBrokenTheme.tmp`

####If this resolves your issue

Then you need to dig a bit deeper as to what part of the template is problematic. So restore your package and attempt the following, testing between each.

Essentially, the process is to gradually enable directories as you traverse down the file tree - until you can find the offending file.

 1. Rename the layout directory to `.tmp`
 2. Rename the template directory to `.tmp`

Then if either yields a fix, rename all files within the layout directory to `.tmp` - (for the SSH users `ls | xargs -I {} mv {} {}.tmp` or `rename 's/^/.tmp/' *`) 

Then gradually enable each file 1 by 1 until resolved.

####If this doesn't resolve your issue

There is potential that your `base/default` or `enterprise/default` directories have become contaminated - and are best replaced with a known clean version. 

You can do this by downloading a clean build of Magento and replacing your directories as necessary. Via SSH you can do this:

    cd /home/path/public_html/
    mkdir clean_mage
    cd clean_mage
    MAGENTO_VERSION=1.7.0.0
    wget -O magento.tgz  http://www.magentocommerce.com/downloads/assets/$MAGENTO_VERSION/magento-$MAGENTO_VERSION.tar.gz
    tar xvfz magento.tgz
    cd /home/path/public_html/app/design/frontend
    mv base{,.tmp}
    cp -par /home/path/public_html/clean_mage/magento/app/design/frontend/base .
    cd /home/path/public_html/skin/frontend
    mv base{,.tmp}
    cp -par /home/path/public_html/clean_mage/magento/skin/frontend/base .

You can also take the opportunity to `diff` the two directories if you want to verify any changes.

    diff -r base base.tmp

NB. This method will cause more errors during the process, as module dependency dictates the existence of specific files. *Unfortunately, its par for the course.*

---

###Disable local modules

By default, Magento defines the PHP include path to load classes in the following order

    Local > Community > Core

> If a file is in Local - load it and do no more.  
> If a file is in community - load it and do no more.  
> If a file can't be found anywhere else - load it from the core.  

Again, rather than disable modules via the Magento admin panel, it is more practical to do this at a file level. 

Typically, to disable a module the "proper" way, you would edit the respective `./app/etc/modules/MyModule.xml` file and set `<active>false</active>` - however, this doesn't actually prevent a class from loading.

If another class extends a given class in a module (ignoring any Magento dependency declarations), it will still be loaded - regardless of whether the extension is disabled or not. 

So again, the best means to disable an extension is to rename the directory.

#####Begin by disabling local

Just rename the directory via FTP, or use the following SSH command

    mv ./app/code/local{,.tmp}

#####Then disable community

    mv ./app/code/community{,.tmp}

####If the issue is resolved from either

Then it is a case of understanding which module in particular the error stemmed from. As with the example given above for the package diagnosis, the same process applies.

So restore the X directory and attempt the following, testing between each.

Essentially, the process is to gradually enable directories (modules) one-by-one until the error re-occurs

 1. Rename all the modules in the directory to `.tmp` (for the SSH users `ls | xargs -I {} mv {} {}.tmp` or `rename 's/^/.tmp/' *`)
 2. Gradually enable each module one-by-one, by removing `.tmp` from the file name

####If the issue is not resolved

Then it is possible the core itself is contaminated. The main Magento PHP core consists of

> ./app/code/core  
> ./lib

So again, rename these directories and copy in a clean variant. Assuming you already downloaded a clean version of Magento as above, via SSH, you can do this:

    cd /home/path/public_html/app/code
    mv core{,.tmp}
    cp -par /home/path/public_html/clean_mage/magento/app/code/core .

Then if the issue still isn't resolved, replace the `lib` directory too

    cd /home/path/public_html
    mv lib{,.tmp}
    cp -par /home/path/public_html/clean_mage/magento/lib .

At this point, your Magento store will be nothing more than a vanilla installation with a modified database.

Some models are actually still stored in the database (Eg. order increment) - so at this point, it becomes a case of manually making those edits. So far, all the steps above have been reversible with no lasting damage. But if we were in import a clean Magento database too - it could prove irreversible (short of restoring a backup).

---

The guide above serves to get you on your way to identifying an error; not to fixing the resultant error.

<sup>Content willingly sourced from [www.sonassi.com/knowledge-base/magento-debug-process][2] and [www.sonassi.com/knowledge-base/stop-magento-permissions-errors-permanently][3]</sup>

  [1]: http://www.sonassi.com/
  [2]: http://www.sonassi.com/knowledge-base/magento-debug-process/
  [3]: http://www.sonassi.com/knowledge-base/stop-magento-permissions-errors-permanently/


-----------------------
## Issue-specific solutions

### <a name="emails-not-sending"></a>Emails not sending

> My (transactional) emails are not being sent

1. Are there any emails sent or just not the ones from the queue
2. Did you check the queue table if the e-mails are added to the queue correctly?
3. Do you use an module for sending emails?
4. Are you sending email via SMTP or Sendgrid etc and if so, is the SMTP or service working properly?
5. Did you check if contact form email is sending, is it just the transactional emails?
6. If sending via localhost, try to create a test script with the following code `<?php mail('your@email.com', 'Test email', 'Test email'); ?>` in your magento root and execute it from browser.
7. Did you check the mail log created by your server? Are the emails in there?
8. Check "Spam" folder in your email account.

### <a name="productimages-not-displaying"></a>Product images not displaying

> Product images are not displaying on the front end 

1. Check file permissions on `media` folder (775 for directory and 664 for files)
2. Check if the file owner is correct (could be `root:root`)
3. Flush first image cache and then regular cache

### <a name="cronjob-not-running"></a>Cronjob not running

> One or more, or all cronjobs in Magento fail to run 

1. Install [Aoe Scheduler](https://github.com/AOEpeople/Aoe_Scheduler/) to an overview of tasks
2. Empty the `cron_schedule` table and ensure Magento rebuilds it
3. Check config XMLs for syntax errors, a syntax error might cause cronjobs not to be picked up
4. Check logs and Aoe Scheduler for errors thrown in cronjobs. If one fails, the ones afterwards will too.
5. Call the cronjob method directly from a script (`[Namespace]_[Module]_Model_Cron::theMethod()`). Does it execute like expected?
6. Check if the cronjob in Linux has any issues using this answer http://stackoverflow.com/a/2264897/387136
7. For magento 1.8+ check if PHP function `shell_exec` is not disabled in php.ini

### <a name="applying-patches-without-ssh"></a>Applying patches without SSH
1. Do it locally and then deploy the changed files to the server (prefered with git, capistrano or something alike)

#### Updating files on a live environment is a bad idea, but if you don't have an alternative:
1. Ask your hosting company to do it for you. If they're not able, try below
2. upload the patch file to the root directory.
2. create a new PHP file, call it whatever you like
3. add below contents to the file
4. call the file from the browser
5. !IMPORTANT - remove both the PHP and Patch file from the server
```php
print("<pre>");
shell_exec("/bin/bash [name of patch file]");
print("</pre>");
```

### <a name="form-field-validation"></a>Form field validation
Magento offers a full Javascript validation library out of the box. These validations can be used by adding a class to an input field.

Below a validation for E-mail addresses.
```html
<form name="some-form" id="some-form" action="" method="post">
[...]
    <label for="samefield"><?php echo $this->__('Some field') ?> <span class="required">*</span></label><br />
    <input id="samefield" name="samefield" class="input-text required-entry validate-email" />
[...]
</form>
```

You can find all available validations in `js/prototype/validation.js`. In Mage EE 1.14.2 / 1.9.2.2 the following are available
> validate-no-html-tags
validate-select
required-entry
validate-number
validate-number-range
validate-digits
validate-digits-range
validate-alpha
validate-code
validate-alphanum
validate-alphanum-with-spaces
validate-street
validate-phoneStrict
validate-phoneLax
validate-fax
validate-date
validate-date-range
validate-email
validate-emailSender
validate-password
validate-admin-password
validate-cpassword
validate-both-passwords
validate-url
validate-clean-url
validate-identifier
validate-xml-identifier
validate-ssn
validate-zip
validate-zip-international
validate-date-au
validate-currency-dollar
validate-one-required
validate-one-required-by-name
validate-not-negative-number
validate-zero-or-greater
validate-greater-than-zero
validate-state
validate-new-password
validate-cc-number
validate-cc-type
validate-cc-type-select
validate-cc-exp
validate-cc-cvn
validate-ajax
validate-data
validate-css-length
validate-length
validate-percents
required-file
> validate-cc-ukss

In case you need some custom validation that will require some javascript. In the template file add the following after the form

```html
<form name="some-form" id="some-form" action="" method="post">
[...]
    <label for="samefield"><?php echo $this->__('Some field') ?> <span class="required">*</span></label><br />
    <input id="samefield" name="samefield" class="input-text required-entry validate-custom-value" />
[...]
</form>

<script type="text/javascript">
//<![CDATA[
    var form = new VarienForm('some-form', true);
    Validation.add('validate-custom-value','This field value must be "foo"',function(v){
        return (v == 'foo') ? true : false ;
    });
//]]>   
</script>
```
