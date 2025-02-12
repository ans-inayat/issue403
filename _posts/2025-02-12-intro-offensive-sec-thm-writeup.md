---
title: "Intro to Offensive Security | TryHackMe"
date: 2025-02-12
author: "Ans Inayat"
image:
 path: https://cdn-images-1.medium.com/v2/resize:fit:800/1*Bcaf1fBXdu2A4xlBuYtLXw.jpeg
description: A detailed walkthrough of the TryHackMe 'Intro to Offensive Security' lab, covering terminal usage, hidden web pages, and vulnerability identification.
categories: [cybersecurity, tryhackme, walkthrough, thm]
tags: [offensive security, penetration testing, ethical hacking]
---

## Intro to Offensive Security | TryHackMe  
In the "Intro to Offensive Security" lab on TryHackMe, I gained hands-on experience in using the terminal, uncovering hidden web pages, and identifying security vulnerabilities. This exercise emphasized the importance of proactive security measures in protecting sensitive information and inspired me to further my journey in cybersecurity.
![0__Ml5quYmNRUqfDqd](https://github.com/user-attachments/assets/8bff0b73-d95d-4617-9469-052e980ff80b)

**Lab Access:** [TryHackMe - Intro to Offensive Security](https://tryhackme.com/room/introtooffensivesecurity)
![0_HlQpjUvA2vuNVgHa](https://github.com/user-attachments/assets/f4538b7a-cce7-4d78-8ede-d74591bc7573)

---

## 1st - Open a Terminal  
We can interact with a computer without using a graphical user interface by using a terminal, often known as the command line. Using the Terminal icon on the system, open the terminal:
![0__YeYiprJu7U-ziaK](https://github.com/user-attachments/assets/292c0637-4229-4c19-a2f2-b7ebcfba28ef)

*Kali Linux's Terminal Icon*

---

## 2nd - Find Website Pages That Have Been Hidden  
Many businesses have their own admin portal, which grants employees access to basic admin controls for day-to-day activities. These pages are frequently not private, allowing attackers to discover hidden pages that display or provide access to administrative controls or sensitive data.

**Example:** A bank employee may transfer funds to and from client accounts.

**Command:**  
```bash
gobuster -u <target> -w wordlist.txt dir
```
- `-u` = specifies the website we're scanning
- `-w` = specifies a list of words to iterate through to find hidden pages
![0_SHTuq7o1kLihVRbx](https://github.com/user-attachments/assets/1af91303-15ce-4cdb-a1b7-4d36a27c6291)

GoBuster scans the website for each term in the list, locating existing pages. In the list of page/directory names, GoBuster reveals the discovered pages (indicated by `Status: 200`).

---

## 3rd - Hack the Bank  
You should have discovered a hidden bank transfer page (`/bank-transfer`) that allows you to transfer money across bank accounts. Enter the hidden page into the FakeBank website.

This page allows an attacker to steal money from any bank account, posing a serious risk to the organization. As an ethical hacker, you would uncover flaws in their program (with permission) and submit them to the bank to be fixed before a hacker exploits them.

### **[Question 1.1]**  
When you've transferred money to your account, go back to your bank account page. What is the answer shown on your bank balance page?  
**Answer:** `BANK-HACKED`


![0_zRIeh6IRxY-wYicK](https://github.com/user-attachments/assets/5f4fece1-caa5-4023-a669-75e4f8ba719b)
![0_fPS3Q67yBIiw43eY](https://github.com/user-attachments/assets/56f3444c-2bca-4c82-a176-c325e3cb074c)
![0_IhYmFZGdok-Zq74x](https://github.com/user-attachments/assets/d0a3fe7c-ae77-4afb-aece-7cf8c83659d4)
![0_wejImy5ipKQ3aacE](https://github.com/user-attachments/assets/8fdce109-5d7f-4961-924a-c3d83e74cdb5)
![0_3KJBjKy3mFbDLEoY](https://github.com/user-attachments/assets/d189883c-5654-4d3d-afa9-88c3caa0ca66)

### **[Question 1.2]**  
If you were a penetration tester or security consultant, this is an exercise you'd perform for companies to test for vulnerabilities in their web applications; find hidden pages to investigate for vulnerabilities.  
**Answer:** No answer needed.

### **[Question 1.3]**  
Terminate the machine by clicking the red "Terminate" button at the top of the page.  
**Answer:** No answer needed.


![0_j6wkSk-tctMYzUcK](https://github.com/user-attachments/assets/52f298e4-684f-4dba-8ff1-2c14cc1a64d9)

---

## Offensive vs. Defensive Security  
### **Offensive Security**  
It is the process of gaining unauthorized access to computer systems by breaking into them, exploiting software defects, and identifying loopholes in programs.

> *To defeat a hacker, you must act like a hacker—identifying flaws and offering patches ahead of a cybercriminal.*

### **Defensive Security**  
On the other hand, defensive security involves safeguarding an organization's network and computer systems by assessing and securing potential digital threats.

You could be analyzing infected systems or devices to determine how they were hacked, chasing down cybercriminals, or monitoring infrastructure for malicious activities.

### **[Question 2.1]**  
Read the above.  
**Answer:** No answer needed.


![0_ZZtCorudteQrpXPA](https://github.com/user-attachments/assets/c361dc7e-9d83-4f3f-bbf4-206ad815486c)

---

## How to Start Learning?  
People often ask how others became hackers (security consultants) or defenders (cybercrime analysts). The solution is straightforward:  

1. Pick a cybersecurity topic that interests you.
2. Practice with hands-on exercises regularly.
3. Learn something new on TryHackMe daily.

By following this routine, you'll gain the skills needed to land your first job in cybersecurity.

### **Cybersecurity Career Paths**  
The [Cyber Careers Room](https://tryhackme.com/room/careers) provides an in-depth look at different career options. Here are a few offensive security roles:

- **Penetration Tester** - Tests technology products to find exploitable security vulnerabilities.
- **Red Teamer** - Plays the role of an adversary, attacking an organization and providing feedback from an attacker's perspective.
- **Security Engineer** - Designs, monitors, and maintains security controls, networks, and systems to prevent cyberattacks.

### **[Question 3.1]**  
Read the above, and continue with the next room!  
**Answer:** No answer needed.

---

## Conclusion  
The "Intro to Offensive Security" lab provided invaluable hands-on experience in identifying vulnerabilities and understanding the mindset of a hacker. As I continue my journey in cybersecurity, I'm eager to apply these skills in real-world scenarios, helping organizations strengthen their defenses against potential threats. Continuous learning and practice are essential in this ever-evolving field, and I look forward to tackling more challenges on my path to becoming a proficient cybersecurity professional!

---

**Happy Hacking! 🚀**
