---
title: "Using Third Party Email Aliases With iCloud Mail on iOS/iPadOS"
date: 2022-09-07
tags:
  - ios
  - ipados
  - icloud
draft: false
toc: true
---

How to: Use Third Party Email Aliases With iCloud Mail on iOS/iPadOS

My primary mail account is with [Posteo](https://www.posteo.de), a privacy-focused provider from Germany. I don't intend to change that, they're great and deserve every support!

Also, I'm using Apple's operating systems almost exclusively and subscribe to iCloud. iCloud and third party mail aren't a great match, unfortunately.

So for a long time I couldn't figure out _how to use my iCloud account to both receive and send mail through my posteo.de address_. On Android using Gmail that wasn't hard: Forward mail to your Gmail account and register your posteo address as an alias. This alias was then available also in the Gmail app.

It wasn't that straightforward with iOS's and iPadOS's Mail app, but a [StackOverflow post](https://apple.stackexchange.com/questions/145565/how-to-set-the-iphone-mail-app-to-send-email-only) helped immensely. Nevertheless, I wanted to document the process of setting it up with pictures and in full length.

Notes:
* The process is identical on iOS and iPadOS. On MacOS's Mail app it's much simpler since it supports aliases "natively."
* The process should apply to **any** third party email address.
* Unfortunately, there is still some manual intervention necessary to have your iCloud account as the single source of truth for sent mails, i.e. you need to move them from your third party mail provider's _sent_ folder to the iCloud one. More on that later.
* The screenshots show a German UI, but one should be able to follow it nevertheless. I was too lazy to adjust the language settings on my phone.

## 0. Prerequisite

Forward mail from \<mail\>@\<thirdpartyemailaddress\>.\<tld\> to \<mail\>@\<icloudaddress>.\<tld\> so you got the "receive third party mail"-part covered. Now, sending is going to be trickier.

## 1. Add Your Third Party Account to Mail

![iOS Screenshots 1-3 showing how to add your third party account to Apple's Mail app on iOS](/images/til-mail-alias-ios-ipados/setup-01.png)

Set up the account as _other_ (2) and choose _mail account_ (3) as the account's type.

![iOS Screenshots 4-6 showing how to add your third party account to Apple's Mail app on iOS](/images/til-mail-alias-ios-ipados/setup-02.png)

Enter your account data including credentials (4). When it comes to the mail server settings (5), choose a dummy for the server handling incoming mail, and your actual mail server for outgoing mail (SMTP). Be sure to pick _POP_/POP3 as the protocol. The latter is for preventing the mail client to try and fetch server-side settings (e.g. folders) for incoming mail. Then confirm.

![iOS Screenshots 7-9 showing how to add your third party account to Apple's Mail app on iOS](/images/til-mail-alias-ios-ipados/setup-03.png)

It's going to take a while, then complain about SSL (7). In the popup, do _not_ try to setup the account without SSL, i.e. press _No_ ("Nein") so you'll still use SSL and server port 587 to send mails. It's going to take another while, then present you with an overview (8) which you'll confirm with _Save_ ("Sichern"). Discard the popup warning you about possible problems with the account (9) by hitting _Save_ ("Sichern") again.

## 2. Adjust Fetch Settings for the New Account

![iOS Screenshots 1 and 2 showing how to setup fetch settings for the dummy POP3 server](/images/til-mail-alias-ios-ipados/fetch.png)

Pick the third party account (1) and set the fetch schedule to _manual_ (2; "Manuell"). Then make sure _manual_ is also selected for non-push schedules (1). These settings will prevent the mail app from trying to fetch mail from your dummy POP3 server.

## 3. Optional: Set the New Account As the Default for Outgoing Mails

Make your third party account the default account for sending mail in the _Compose_ section of Mail's settings. This is optional, you could still select the account from within the mail editor. I just like to have it already available as the default.

## 4. Bonus: A Way to Copy Sent Mails to iCloud

As mentioned before, the biggest caveat to the whole thing: You still need to transfer sent mails from your third party account's _Sent_ folder to your iCloud's _Sent_ folder. While I haven't found a way to automate it through the _Shortcuts_ app, one can still do it manually by just moving the email(s) to the correct folder on iOS/iPadOS through the Mail app's _Move_ dialog.

For my usage that wasn't a problem since I barely write any emails these days. Your mileage may vary, of course.