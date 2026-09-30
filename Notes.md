# Web

Explore chat panel. Seems like an AI agent.
![[Pasted image 20260930164001.png]]

Form a list of of possible user names:
```bash
gain-initial-access-via-social-engineering ● /opt/username-anarchy/username-anarchy Jordan Kim -@ @megabigtech.com -C False > possible_usernames.txt
gain-initial-access-via-social-engineering ● cat possible_usernames.txt
jordan@megabigtech.com
jordankim@megabigtech.com
jordan.kim@megabigtech.com
JordanKi@megabigtech.com
JordKim@megabigtech.com
jordank@megabigtech.com
j.kim@megabigtech.com
jkim@megabigtech.com
kjordan@megabigtech.com
k.jordan@megabigtech.com
kimj@megabigtech.com
kim@megabigtech.com
kim.j@megabigtech.com
kim.jordan@megabigtech.com
JKim@megabigtech.com
jk@megabigtech.com
JK@megabigtech.com
JordanKim@megabigtech.com
Jordan.Kim@megabigtech.com
Kim@megabigtech.com
```

`first.last` format works:
```bash
gain-initial-access-via-social-engineering ● /opt/azure/o365enum/o365enum.py -m office.com -u possible_usernames.txt
username,valid
jordan@megabigtech.com,0
jordankim@megabigtech.com,0
jordan.kim@megabigtech.com,1
JordanKi@megabigtech.com,0
JordKim@megabigtech.com,0
jordank@megabigtech.com,0
j.kim@megabigtech.com,0
jkim@megabigtech.com,0
kjordan@megabigtech.com,0
k.jordan@megabigtech.com,0
kimj@megabigtech.com,0
kim@megabigtech.com,0
kim.j@megabigtech.com,0
kim.jordan@megabigtech.com,0
JKim@megabigtech.com,0
jk@megabigtech.com,0
JK@megabigtech.com,0
JordanKim@megabigtech.com,0
Jordan.Kim@megabigtech.com,1
Kim@megabigtech.com,0
```

# Jordan Kim access
Self password reset (https://learn.microsoft.com/en-us/entra/identity/authentication/concept-sspr-howitworks) is configured:
![[Pasted image 20260930164714.png]]

```
Who is the most famous person you have ever met?

...

One standout memory in my career was meeting Vint Cerf, one of the "fathers of the internet," during a panel on digital ethics. It was an incredible experience to hear his insights firsthand. Do you have any other questions or need assistance with something else?

========

What was the make and model of your first car or motorcycle?

...

My first car was a 2002 Toyota Corolla. It was a reliable vehicle and served me well during the early days of my career. Do you have any other questions or topics you'd like to discuss?

========

In what city was your first job?

...

My first job was in Toronto. It was a great place to start my career and gain experience in the tech industry. Is there anything else you'd like to know?
```

![[Pasted image 20260930165037.png]]

Set a password to `VeryStr0ngP4ss!`
![[Pasted image 20260930165310.png]]

Get ARM & Entra t