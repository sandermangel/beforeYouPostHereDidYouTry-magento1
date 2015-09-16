# beforeYouPostHereDidYouTry - Magento 1.x
This is a list of points you should check before posting your issue online. We've run into these issues before and know what the standard issues might be so please take the time to work them through

## Basic checks when running into any issue

## Emails not sending

> My (transactional emails are not being sent)

1. Are there any emails sent or just not the ones from the queue
2. Did you check the queue table if the e-mails are added to the queue correctly?
3. Do you use an module for sending emails?
4. Are you sending email via SMTP or a sendgrid-like service and if so, is the SMTP or service working properly?
5. Did you check if contact form email is sending, is it just the transactional emails?
6. If sending via localhost, try the following from the commandline `$ endmail your@email.com < echo "Test email"`
7. Did you check the mail log created by your server? Are the emails in there?
