# CTF Platform
We used CTFd (https://ctfd.io/) as a CTF Platform for flag submissions (Kudos for great open-source project!) , however, based on the fact that the structure and rules of our CTF were different than standart CTF competitions, we had to make our modifications to CTFd to satisfy the needed requirements. 

## Initial setup and configurations
CTFd runs on docker containers and all you need to do is pull the source code from official repository (https://github.com/CTFd/CTFd), and run `docker compose up` command in the CTFd folder where docker-compose.yml file exists. This operation will run all the needed services (Application, Redis, MySQL and Nginx) with inital configurations and you can visit the platform at localhost:8000 to continue the configuration process where you set up password for administrator account, and tune some settings for CTF competition (such as Team mode activation, event start and end time, theme to use and etc.). 

> [!WARNING]  
> If you modify the source code or Dockerfile, you may need to execute `docker compose down` and `docker compose up` multiple times, which may lead to exhaustion of storage by images and containers. For overcoming this, you can run 'docker system prune' before your last run to ensure clean setup of platform.


### CTFd features that we used
For our CTF we used `Team mode` as 10 teams competed in our competition, and set the maximum amount of team members to three. We disabled public team and user registration in settings and registered all users and teams by ourselves. Two back-up teams were in reserve list, and they would be invited to competition if some teams in initial list didn't attend the event, thus, we created user and team accounts for them also, however, we set their account statuses to `disabled` to make them unvisible in scoreboard. Challenges are grouped by `category` field in CTFd, and we decided to use this feature for grouping challenges of one machine: we put the machine name as category title for all challenges of one machine, and the main dashboard visible for CTF Players looked like this:

![image](https://github.com/NotokDay/ICSD/assets/24704431/238bc0ca-9628-4aee-bd7f-d0bef7480982)


During the competition, we decided to give hints, as teams struggled to get even the first flag of some machines. We needed to broadcast a hint to each player and we achieved this by using `Notifications` feature of CTFd. When publishing a notification, it gets popped out on screen of each CTF Player, also in main screen projected to the wall in event area. These notifications stay in notifications page of the platform. One of the real hints that we published during the competition was: 

(P.S. Hint messages were related to Game Of Thrones characters and their famous quotes. Just for fun ;) 

![image](https://github.com/NotokDay/ICSD/assets/24704431/b9edb02e-b34a-49d9-bdae-47901e1699be)


## Our modifications

This is where the fun starts :smiling_imp:

### Disabling visibility of team solves 
As you know, an ELK platform was used by CTF Players to analyze the traffic and commands of other CTF Players. To make this process more interesting, we decided to disable the visibility of team solves - which means no one knows which team solved which challenge, thus, players needed to analyze all traffic coming from all sources without focusing on a specific IP/source. CTFd uses two methods to display data on pages. First, it does Server Side Rendering for some pages and renders pages with data beforehand. Second, in some pages CTFd API is called from front-end to get additional data, which we needed to block also (some endpoints). For first case : when a CTF Player visits public page of other team, following code piece on 'CTFd/CTFd/teams.py' (https://github.com/CTFd/CTFd/blob/master/CTFd/teams.py) runs:
![image](https://github.com/NotokDay/ICSD/assets/24704431/9d30bc94-adb8-4a02-8474-b74c5be4c736)

We removed lines 364 and 378, therefore, when CTF Players visited public page of other teams, they didn't see specific solves of them. However, the endpoint requested when team visits their own team page is different, thus, they can see their own solves without any constraint. 

Specific solves of users are also visible on public pages of users, which we need to remove. For this, we removed 'solves' section from 'themes/core-beta/users/public.html' (https://github.com/CTFd/CTFd/blob/master/CTFd/themes/core-beta/templates/users/public.html) 

For second part of data retrieval, the API, we needed to block API requests from user machines, but allow administrator machines to access same API endpoints freely. For this, we decided to play with Nginx configuration. By default, as stated earlier, Nginx is set up in docker-compose.yml file to run on-start with default configurations to proxy requests coming to port 80 to 8000 and nothing else. However, we changed configuration to have following lines: 

```
location ~ /api/v1/(teams|users|challenges)/\d+/(solves|fails) {
        deny 10.20.57.0/24;
        proxy_pass http://app_servers;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $server_name; 
}
``` 

Here, we declare that requests to pre-defined API endpoints that contain data about team and user solves (which we want users not to see) coming from CTF Players' VLAN (10.20.57.0/24) should be blocked. 

### Removing access of team members to the completed machine (by notifying staff through Telegram)
 
The second feature that we developed was a notification system to alert the staff about completion of challenges belonging to one machine by any team. The notification is sent to a group with staff members and our notifier Telegram bot in a following format: 

![image](https://github.com/NotokDay/ICSD/assets/24704431/5683051b-b90e-4d87-a1eb-233dd372fa4f) 

In five minutes after the notification is received, firewall rules are updated to disable team Attacbox's access to the finished machine.  

For achieving this kind of functionality, first we created a Telegram Bot using BotFather. Then, we somehow needed to create a script to check status of completed challenges by team in every successful flag submission and in case of completion notify us through Telegram. We developed script called `completion_checker.sh` which you can view in this repository/folder.  This script is added to `CTFd/CTFd/api/v1/ctfd_scripts` folder and called in `CTFd/CTFd/api/v1/challenges.py' (https://github.com/CTFd/CTFd/blob/master/CTFd/api/v1/challenges.py) as following (added to 637th line) : 

```
subprocess.run(['/opt/CTFd/CTFd/api/v1/ctfd_scripts/completion_checker.sh',f"{team.id}", f"{team.name}", f"{challenge.category}"])
```
P.S: For having access to `curl` and `jq` tools used inside script, we modified Dockerfile (second FROM block) to install curl and jq also to container: 

``` 
FROM python:3.9-slim-buster as release
WORKDIR /opt/CTFd

# hadolint ignore=DL3008
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        libffi6 \
        libssl1.1 \
 curl \
 jq \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* 

``` 



### Regenerating flags every 30 minutes and on successful submission 

Another requirement to prevent cheating and make competition more competitive was to regenerate flags on platform and on vulnerable machines at same time. We achieved this by adding following cron entries to crontab:  
 
'''
*/2 * * * * bash /home/ctf-platform/Desktop/CTFd/CTFd/api/v1/ctfd_scripts/overwrite_scripts/total_overwrite.sh
*/30 * * * * python3 /home/ctf-platform/Desktop/CTFd/CTFd/api/v1/ctfd_scripts/db_flag_regeneration.py --all
''' 

`total_overwrite.sh` and `db_flag_regeneration.py` files are located in folder root in repository. We reinsert flags to machines every two minutes (without regenerating) and regenerate the flags in DB by using CTFd API every thirty mintues and on successful flag submission by team (by adding following line to 'CTFd/CTFd/api/v1/challenges.py' (pasting it just after the previous subprocess call we added for completion_checker.sh )) : 

``` 
subprocess.run(['python3','/opt/CTFd/CTFd/api/v1/ctfd_scripts/db_flag_regeneration.py',f"{challenge.category}"]) 
```  
 
