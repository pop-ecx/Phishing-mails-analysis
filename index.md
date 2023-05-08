- [Analysis of a credential harvester attack ](#Analysis-of-a-credential-harvester-attack)
- [Initial thoughts ](#Initial-thoughts)
- [Carving up the email for clues](#Carving-up-the-email-for-clues)
- [Making sense of the code](#Making-sense-of-the-code)


### Analysis of a credential harvester attack

Cybersecurity is a huge field. A part of it, and in my opinion the most exciting part, is combing through the weeds looking for nefarious stuff
hiding in regular things like email messages, executables, web requests and that, subsequently revealing what threat actors are up to. 
Not too long ago I came across an interesting email that got my attention. So, I decided to investigate.

### Initial thoughts

One of the first things I do is go through the email to see if there are any immediate red flags. The mail didn't say much, just 'find
attached the revised final documents'. The attachment itself, a .pdf.html document. Interesting, now some 'big boy stuff'.

### Carving up the email for clues

For a bit of background, emails mostly come as .eml or .msg files. The procedure of obtaining those files themselves depends on the 
email service in question. In my case, it was pretty straightforward and I obtained the relevant eml. Going through the file itself is 
a bit of a pain. There's a lot of information that may not be entirely relevant. To make it easier, I used Phishtool, as it gives 
the relevant security information. Aside from the 'From' header, phishtool didn't see a lot of problems with the email. My main concern 
however was the attachment. I quickly extracted the attachment using emlAnalyzer. A quick look at the contents of the attachment revealed 
base64 encoded javascript. The atob() gave it away, right?

![Base64 encoded javascript](/Screenshot5.png)

The `url_string` variable was an actual email address, the intended target of the attack.

### Making sense of the code

The code is base64 encoded, so a quick decoding was no big issue. Decoding revealed html code with some js. so,this one was 
intriguing. The code was not obfuscated in any sense giving me a relatively easy time analyzing what it does. The first interesting part 
was js code that blocked the use of developer tools, inspect element and view page source once the html is opened in a browser. 
That's neat. This was to deter any analysis effort by the potential victim.

![Code to block common shortcuts to developer tools](/Screenshot1.png)

Continuing through the code, another interesting script block is found. This time this is a lengthy
block that defines the set_brand function.

![The set_brand function](/Screenshot2.png)

This function just sets the 'visuals' or rather the display of the page. We do have another interesting function though: the send_result function.

![The send_result function](/Screenshot3.png)

The send_result defines the data to be sent, url to be sent to and the method used. In this case POST is used to send data to the url.

![](/Screenshot4.png)

The rest of the code does a couple of interesting things. It first waits for the entire html document to load and sets up a listener. Then, 
it checks if the password length is less than 5. If it is, it complains about the password being too short. If the password length is greater than 5
and count is less than or equal to zero, it sends the user and password collected via an ajax POST request to the adversary's url 
`http://jktradingcompany.com/css//pii22.php`. It then increments the count and complains your password is incorrect. Naturally an unsuspecting 
user will enter the password again. This time it checks if the count is less than 2. If it is, it will send the credentials and complain the password is incorrect.
If it isn't it will still collect and send the credentials, only this time it will redirect the victim to `https://outlook.office365.com/Encryption/ErrorPage.aspx?src=3&code=11&be=SN6PR04MB4014&fe=JNAP275CA0040.ZAFP275.PROD.OUTLOOgK.COM&loc=en-US&itemID=E4E_M_e9df154a-e4b8-4486-8aec-7acceeb93fee`. Yeah, 
that's just an error page from office365's Message Encryption. At this point the victim probably would not notice their credentials have been stolen.
The attacker's url has a .php file meaning we could safely assume the credentials are caught server side. You could interact with the url using a tool like curl 
and see how it behaves. I'll leave that as an exercise to the reader. So this is a credential harvester attack. It seems the threat actor takes the victim's password as 
long as it's above five characters, and just to be sure asks the victims to confirm it by pretending it's incorrect. So this is how the phishing page looks like:

![Phishing page](/Screenshot6.png)

I don't think this is a very convincing phishing page, but that's just me. To stay safe from such attacks, do not open email attachments in unsolicited emails,
change your passwords often and most importantly use 2FA. It can also be useful to get another pair of eyes, especially cybersecurity-trained ones.

`file hash: 21aa51227aab8d13482de3251a629e934c58af95d319830348cd35fefbccc97f`
