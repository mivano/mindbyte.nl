---
title: Password complexity
tags: 
  - security
excerpt: The complexity of picking a good password and what you can do as a developer
header:
  image: /images/covers/passwordcomplexity.png
---

Creating a password for any site or service is difficult enough, but even more when the password needs to comply with a password policy. Minimum length, number of digits, uppercase and lowercase mixes and nonalphabetical characters are favorites. And with a bit of luck, it should also not be the same password as used in the last x password resets. Combine that with a password age so the user needs to do this exercise every number of days.

image::IC212305.gif[]

What we get is either a technically capable person using a password helper with long random passwords or someone with a _complex_ password used on all sites. When it needs renewal, it normally has a sequential number added to it. 

So creating secure and safe passwords is a challenge for sure. Passwords that look very secure, as in they match the password complexity rules, are actually not secure at all. Take for example *&Password1*, which is of a decent size, has lower and uppercase characters, has a digit and even a non-alphabetical character. Even when we use common words it is quite often the first character which is a uppercase. A _e_ becomes a _3_, a _0_ for an _o_ etc. Hackers know this so almost all the dictionary attacks have these words and their variations on their list.
Of course making it complex will make it harder to hack, but also harder to remember by the actual user.

image::password_strength.png[]

Although the above example shows that this is easier to remember, it is (http://cups.cs.cmu.edu/soups/2012/proceedings/a7_Shay.pdf)[not proven] to be. 

## Direction

We are seeing more and more that the current approach to password complexity is changing. Microsoft released a nice paper on https://www.microsoft.com/en-us/research/wp-content/uploads/2016/06/Microsoft_Password_Guidance-1.pdf[password guidance], but also https://www.nist.gov/itl/tig/special-publication-800-63-3[Nist] published some excellent guidance. 

It basically boils down to a couple of points:

- Stick with a minimum 8 character password. 
- Do not allow common passwords. 
- Forgo the character composition mix.
- Promote the use of multi factor authentication.
- Do not force your users to reset passwords periodically.
- Check and warn users of attack attempts.

### Common passwords

This is a pretty interesting one; after a lot of breaches where passwords have been made public, there is now a list of https://www.troyhunt.com/introducing-306-million-freely-downloadable-pwned-passwords/[306,259,512 unique passwords] which were used in those breaches. It shows that people tend to stick with common and simple passwords.  

So the first step in your application where a user needs to select a password is to check if it is used in a previous breach. Troy Hunt published a list and https://www.troyhunt.com/introducing-306-million-freely-downloadable-pwned-passwords/[some guidance] on how to do this.

Either download the full file and do this check in your application (more control, no dependency) or use his API (make sure to hash the password before sending).

Alternatively, there are systems like https://blogs.dropbox.com/tech/2012/04/zxcvbn-realistic-password-strength-estimation/[Dropbox zxcvbn] which will check against patterns, a common list and the amount of time it takes to guess the password. The script will return a value and tip which can be used in a https://github.com/dropbox/zxcvbn#usage[password strength meter] to the user.

Make sure to also include user specific elements like username, email, full-name etc so the user cannot mix or use them in the password. Same applies to the name of your service/application. 

### Multi factor authentication

Something you know, something you have and something you are. A password is something you know. Checking a fingerprint or iris is what you are, although this is harder to check. So the something you have is a popular second-factor authentication method. 

If the user has a key generator (even a soft one), it is pretty easy to http://brandonpotter.com/2014/09/07/implementing-free-two-factor-authentication-in-net-using-google-authenticator/[generate One Time Passwords]. Other options are SMS or phone delivery (user need to confirm the ownership of the phone) or using email (same as with phone, make sure to confirm the email address is of the user first). 

When you use the default ASP.NET security framework there is reasonable support on https://docs.microsoft.com/en-us/aspnet/mvc/overview/security/aspnet-mvc-5-app-with-sms-and-email-two-factor-authentication[how to build this] as it is not part of the framework. 

### Third parties

Consider also if you do want to maintain passwords at all. There is a lot of hassle and issues associated with this. Sometimes this is unavoidable, but it might in other cases be an option to leverage other services which can spend more time, money and resources in this area. 

For the more *business to consumers* markets you can use Facebook, Google, Twitter etc as an authentication provider. If it is more *business to business*, consider using, for example, Azure Active Directory. 

Be aware that you do create a dependency and lose control in favor of other benefits you might get. If you want to keep it all in-house, make use of https://identityserver.io/[Identity Server] and as such leverage a proven and extensive authentication system. Implementing your own security is always a bad idea :-).

### Other options

Do you always need passwords? You can also just use tokens and send a link per email to the user when he tries to login in. The https://auth0.com/blog/how-passwordless-authentication-works/[passwordless authentication] is pretty easy. No burden at your side to maintain passwords, do resets, get stuff stolen when there is a breach etc. However, you need to trust the delivery mechanism like email and the speed of the delivery. More and more services are offering this next to their normal login as it is less typing and easier for their users, certainly with https://www.drzon.net/posts/passwordless-login-in-mobile-apps/[mobile applications].

## Conclusion

Passwords are hard; hard to remember, hard to make secure. The best intent to make them complex by requiring a mix of character types and lengths does not help. Combined with commonly used words, common replacements and placements and repetitive password reset, we make it our users pretty hard.

Support password generators (so do https://www.troyhunt.com/the-cobra-effect-that-is-disabling/[not disable clipboard paste] in your forms), do not have mandatory password resets and complexity, but do check against common patterns (common words, user specific values, patterns and passwords used in breaches) and keep checking and informing the user afterward. 

Allow for and promote the use of multi factor authentication to add an additional layer of security. 

This will make your service/application more secure and your users happier!

