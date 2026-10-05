# Chapter 01: Introduction to Information Security

## Abstract

This chapter introduces information security as the combination of people, policies, and technology that keeps information safe from unauthorized disclosure, exposure, tampering, interruption, and loss. The chapter covers the CIA triad of confidentiality, integrity, and availability, the vocabulary of vulnerabilities, threats, and threat actors, and the attack vectors that adversaries use. The chapter then examines social engineering, malware categories, and the types and functions of security controls. These foundations support every later chapter in the course.

## Objectives

- Define information security and identify its core components: people, policies and procedures, and technology.
- Explain the CIA triad and the methods used to enforce confidentiality, integrity, and availability.
- Distinguish vulnerabilities, threats, and threat actors, classify threat actors as accidental, intentional, or natural, and list the consequences of a data breach.
- Define attack vector and attack surface, and recognize the fifteen common attack vectors.
- Describe the social engineering lifecycle, the influence principles attackers exploit, and the main social engineering attacks.
- Classify malware by behavior and describe each major malware type.
- Categorize security controls by type (technical, managerial, operational, physical) and by function (preventive, deterrent, detective, corrective, compensating, directive).

## Content

### 1. Introduction to Information Security

#### **Definition**

Information security consists of the practices, policies, and technologies that safeguard information against unauthorized access, exposure, alteration, interruption, or destruction. Information security extends beyond technology: protecting information depends on people, processes, and governance working together.

Information security applies to data in every state:

- **Data at rest:** files stored on disks, databases, and backups.
- **Data in transit:** traffic moving across networks, such as web requests and email.
- **Data in use:** information held in memory while applications process the information.

#### **Goal: The CIA Triad**

All information security work aims to uphold three fundamental principles:

- **Confidentiality:** Data stays reachable only to people who hold proper authorization.
- **Integrity:** Information stays accurate and trustworthy, free from unauthorized changes.
- **Availability:** Authorized users can reach information and systems at the moment of need.

Every security decision in an organization maps back to one or more of these three principles. A control that protects none of them adds no security value.

#### **Core Components of Information Security**

1. **People:** Users, administrators, and managers each hold security duties. Human mistakes and manipulation make people a common point of failure, as when an employee reuses a password or falls for a phishing email. Awareness and training turn the same people into a strong line of defense, as when an employee reports a suspicious message instead of clicking the link.
2. **Policies and Procedures:** Written rules and step-by-step guidelines that govern how the organization collects, stores, and shares information. Examples include password policies, data classification rules, and incident response protocols. Policies tell people what to do, and procedures tell people how to do the task step by step.
3. **Technology:** Hardware and software that enforce protection, including encryption, firewalls, antivirus software, and intrusion detection systems. Technology enforces the rules that policies define.

### 2. The CIA Triad

Every information security practice builds on the CIA Triad.

#### **Confidentiality**

Confidentiality keeps sensitive information out of the hands of anyone without authorization. A hospital protects patient records and a bank protects account balances under this principle.

Methods include:

- **Encryption:** scrambles data so only holders of the correct key can read the data. Examples include TLS for web traffic and AES for stored files.
- **Access controls:** permissions and role-based access that limit who can open a file or record.
- **Authentication:** passwords, PINs, and multi-factor authentication (MFA) that verify identity before granting access.
- **Data classification:** labels such as public, internal, and confidential that tell handlers which protections apply.
- **Physical security:** locked server rooms and screen privacy filters that prevent casual disclosure of on-screen data.

#### **Integrity**

Integrity keeps information accurate, complete, and reliable. Integrity guards data against changes made without permission or by mistake, such as an attacker altering a bank transfer amount or a user corrupting a file by accident.

Methods include:

- **Hashing:** a one-way function such as SHA-256 produces a fixed fingerprint of a file. Any change to the file changes the fingerprint, so a recipient can detect tampering.
- **Checksums:** quick error-detection values that catch accidental corruption during transfer or storage.
- **Digital signatures:** prove both the sender identity and the fact that no one altered the message after signing.
- **Access controls and version control:** restrict who may modify data and keep a history of every change, as Git does for source code.
- **Backups:** allow restoration of a known-good copy after corruption.

#### **Availability**

Availability keeps systems and data reachable whenever authorized users need access. An online store that goes offline loses sales for every minute of downtime.

Methods include:

- **Redundancy:** duplicate servers, disks (RAID), and network paths so one failure does not stop service.
- **Backups and disaster recovery plans:** procedures that restore service after data loss or site failure.
- **DDoS protection and load balancing:** filters and traffic distribution that keep services responsive during floods of hostile requests.
- **Patch management and monitoring:** keep systems stable and detect failures early.
- **Uninterruptible power supplies (UPS):** keep equipment running through power outages.

### 3. Vulnerabilities, Threats, and Threat Actors

#### **Vulnerability**

A vulnerability is a gap or defect in how a system is designed, built, operated, or managed. Examples include unpatched software, weak passwords, and misconfigured servers. A vulnerability on its own causes no harm. Harm requires a threat that exploits the weakness.

#### **Threat**

A threat is any possible event or action that takes advantage of a weakness and causes damage. Examples include cyberattacks, insider abuse, malware, and natural disasters. A flood is a threat to a data center with no raised floor, just as ransomware is a threat to a workstation with no backups.

Risk combines these concepts. Risk exists when a threat can exploit a vulnerability and produce an impact. Security teams reduce risk by removing vulnerabilities, limiting threats, or shrinking the impact.

#### **Threat Actors**

Threat actors are the individuals, groups, or events behind threats. Threat actors fall into three categories:

1. **Accidental:** Human errors cause harm without malicious intent. Examples include an administrator who misconfigures a firewall, an employee who emails a sensitive spreadsheet to the wrong recipient, and a developer who deletes a production database by mistake. Accidental actors cause a large share of incidents, and the same controls that stop attackers, such as least privilege and change management, also limit accidental damage.
2. **Intentional:** Actors who harm systems on purpose, including hackers, insider threats, and organized cybercriminals. Types of intentional threat actors include:
   - **Script kiddies:** unskilled attackers who run pre-made tools and scripts without understanding how the tools work.
   - **Hacktivists:** attackers motivated by political or social causes, who deface websites or leak data to promote a message.
   - **Organized crime:** professional groups that run fraud, ransomware, and theft operations for profit.
   - **Nation-state actors (Advanced Persistent Threats, APTs):** government-sponsored teams with large budgets and long-term goals such as espionage or sabotage.
   - **Insiders:** employees or contractors who abuse legitimate access, either for personal gain or revenge.
   - **Competitors:** organizations that steal trade secrets to gain a business advantage.

   Motivations differ among intentional actors. Organized crime seeks profit, nation-states seek espionage and strategic advantage, hacktivists seek attention for a cause, and insiders act out of revenge or personal gain.
3. **Natural:** Events with no human actor behind the events, such as floods, earthquakes, fires, hurricanes, and extended power outages. Natural threats mainly endanger availability, and organizations counter natural threats with redundancy, backups, and disaster recovery plans.

#### **Common Impacts of Threats**

A data breach occurs when sensitive information reaches people who have no right to see the information. Consequences include:

- **Financial loss:** incident response costs, regulatory fines, and lost revenue.
- **Identity theft:** attackers use stolen personal data to open accounts or commit fraud.
- **Reputational damage:** customers and partners lose trust and leave.
- **Legal and regulatory penalties:** laws such as GDPR impose fines proportional to the scale of the breach.
- **Operational disruption:** downtime and recovery work interrupt normal business.
- **Loss of intellectual property:** stolen designs or trade secrets erase competitive advantage.

The Verizon 2024 Data Breach Investigations Report analyzed 30,458 security incidents and confirmed 10,626 data breaches, which shows how often these consequences occur in practice.

### 4. Attack Vectors and Attack Surface

#### **Attack Vector**

An attack vector is the route or technique an attacker follows to take advantage of a vulnerability. A phishing email and an open network port are both attack vectors.

#### **Attack Surface**

The attack surface covers every point where an attacker could try to enter a system. More entry points mean more chances to break in. Security teams reduce the attack surface by disabling unused services, closing unneeded ports, and removing unnecessary software.

#### **Common Attack Vectors**

1. **Direct Access:** Attackers steal devices or login credentials directly. A stolen laptop without disk encryption exposes every file on the laptop.
2. **Wireless Interception:** Attackers set up rogue access points or eavesdrop on Wi-Fi traffic to capture credentials and session data. In an evil twin attack, the attacker broadcasts a hotspot with the same name as a legitimate network, and victims connect without noticing.
3. **Unpatched Software:** Attackers target systems that miss vendor patches and carry publicly known flaws. In May 2017, WannaCry spread through the EternalBlue flaw in unpatched Windows systems and infected more than 200,000 computers across 150 countries.
4. **Unsupported Applications and Systems:** Out-of-support software receives no further fixes, so each newly discovered flaw remains exploitable for the life of the system.
5. **Messaging Attacks:** Fraudulent messages reach victims through email, SMS (smishing), and chat applications.
6. **Image-Based Payloads:** Attackers hide malicious code inside image files, and a vulnerable image viewer runs the code the moment a user opens the file.
7. **Supply Chain Compromises:** Attackers compromise a vendor first and then ride trusted software updates or integrations into the real target. In 2020, attackers inserted the SUNBURST backdoor into SolarWinds Orion updates, and up to 18,000 organizations installed the compromised software. In 2013, attackers entered the Target network with credentials stolen from an HVAC vendor and took 40 million payment card records and personal data for 70 million customers.
8. **Social Media Exploitation:** Attackers build fake profiles and post malicious links to collect information and distribute malware.
9. **Removable Media:** Attackers drop USB drives in parking lots and lobbies, counting on curious employees to plug the drives into work computers. Stuxnet spread between isolated networks in 2010 through infected USB drives.
10. **Cloud-Based Abuse:** Poorly configured cloud storage buckets leak data to the open internet, and credential stuffing replays stolen passwords against cloud logins.
11. **Open Service Ports:** Every open port and running service gives attackers a direct network path into the system. Attackers scan for exposed services with tools such as Nmap, and administrators close the gap by disabling unneeded services.
12. **Default Credentials:** Devices ship with factory usernames and passwords such as admin and admin, and many administrators never change the defaults. The Mirai botnet logged into IoT devices in 2016 with a list of 62 factory default username and password pairs.
13. **Malvertising:** Attackers buy or hijack advertising slots on legitimate websites, and the malicious ad installs malware or redirects the browser when the page loads.
14. **Drive-By Downloads:** A compromised website installs malware as soon as a victim visits the page, with no click required beyond the visit itself.
15. **Bluetooth and Short-Range Wireless:** Attacks such as bluejacking (sending unsolicited messages) and bluesnarfing (stealing data from a paired or discoverable device) abuse Bluetooth connections in public spaces.

### 5. Social Engineering

Social engineering manipulates people psychologically so the victims take actions or reveal confidential information. Attackers target human judgment rather than software flaws. Technical defenses cannot stop an employee who willingly hands over a password.

#### **Lifecycle of a Social Engineering Attack**

1. **Information Gathering:** The attacker researches the target through social media, company websites, and public records to learn names, roles, and internal vocabulary.
2. **Building Rapport:** The attacker establishes trust by impersonating a colleague, a helpdesk agent, or a vendor.
3. **Exploitation:** The attacker makes the request, such as asking for a login, directing the victim to click a link, or requesting a wire transfer.
4. **Exit:** The attacker covers tracks and ends contact while maintaining access for future use.

#### **Types of Social Engineering**

- **Social-Based:** Plays on trust and emotion. Example: a fake helpdesk call that asks an employee to reset a password to an attacker-chosen value. Voice phishing (vishing) and SMS phishing (smishing) fall in this group when the message itself does the manipulation.
- **Physical-Based:** Relies on physical proximity, with tactics such as shoulder surfing (watching a screen or keyboard), tailgating (following an authorized person through a secure door), and dumpster diving (searching trash for sensitive documents).
- **Technical-Based:** Delivers the attack through technology: malicious links, spoofed sender addresses, typosquatting (registering lookalike domains such as examp1e.com), and watering-hole attacks (infecting a website the target group visits often).

#### **Influence Principles Exploited**

- **Authority:** The attacker claims a senior role, such as a company executive, and the victim complies out of deference.
- **Trust and Familiarity:** The attacker poses as a known colleague or friend.
- **Intimidation:** The attacker uses threats or frightening claims, such as a fake IRS call threatening arrest, to force quick compliance.
- **Urgency:** A tight deadline or a threatened loss, such as a warning that an account will close within the hour, stops the victim from thinking carefully.
- **Scarcity:** A seemingly rare offer, such as a fake prize with a countdown timer, pushes the victim to grab the offer fast.
- **Consensus:** The attacker claims that coworkers or peers already agreed, and the victim follows the crowd.

#### **Specific Attacks**

- **Phishing:** Mass-mailed fraudulent messages sent to thousands of recipients at once, with no specific target.
- **Spear Phishing:** Messages custom-built for one person or one team, using details gathered during research. In July 2020, attackers used phone spear phishing against Twitter employees, gained access to internal tools, and posted a Bitcoin scam from 45 high-profile accounts out of 130 targeted accounts.
- **Whaling:** Spear phishing that goes after executives and other high-value individuals.
- **Business Email Compromise (BEC):** The attacker impersonates an executive to trick staff into wire transfers or gift card purchases. The FBI Internet Crime Complaint Center recorded 2.9 billion dollars in reported BEC losses in 2023, which makes BEC one of the costliest attack categories.

### 6. Malware

Malware is any program written to harm systems, interrupt operations, or break in without authorization.

#### **Categories Based on Behavior**

- **Spread:** viruses, worms, botnets, and cryptominers replicate or propagate across systems.
- **Block:** ransomware locks victims out of their own files through encryption.
- **Spy:** spyware and keyloggers secretly collect sensitive data.
- **Mislead:** Trojans, Remote Access Trojans (RATs), and Potentially Unwanted Programs (PUPs) disguise malicious intent.
- **Hide:** backdoors, logic bombs, and rootkits conceal their presence or wait for a trigger.

#### **Detailed Types**

- **Virus:** Runs only after a user acts, such as opening an infected attachment. A virus attaches to a host file and spreads when the user shares the file. Brain, one of the first PC viruses, spread through infected floppy disks in 1986.
- **Worm:** Self-propagates across networks by exploiting network services, with no user action required. The ILOVEYOU worm spread through email in May 2000 and infected tens of millions of Windows computers within days.
- **Bot/Botnet:** A collection of infected devices that an attacker commands remotely, used for spam, DDoS attacks, and credential theft. The Mirai botnet infected hundreds of thousands of IoT devices in 2016 and used the devices to attack the DNS provider Dyn, which knocked major websites offline for hours.
- **Cryptomalware:** Uses the victim's CPU and electricity to mine cryptocurrency for the attacker.
- **Ransomware:** Scrambles the victim's files with encryption and demands money in exchange for the decryption key. NotPetya in June 2017 masqueraded as ransomware but destroyed data permanently, with total damages estimated above 10 billion dollars.
- **Spyware and Keyloggers:** Gather data in secret, with keyloggers recording every keystroke the victim types. Pegasus spyware reads messages, records calls, and activates the camera and microphone on mobile phones without any user action.
- **Trojan:** Presents itself as a harmless program, such as a free utility or cracked application, to trick users into installing the payload.
- **RAT (Remote Access Trojan):** Hands complete remote control of an infected machine to the attacker.
- **PUPs (Potentially Unwanted Programs):** Programs the user never asked for, such as adware, browser toolbars, and bloatware, that arrive bundled with other software.
- **Backdoor:** A concealed entry point that lets an attacker in without going through normal authentication.
- **Logic Bomb:** Malicious code that stays dormant until a condition or date triggers the payload, such as a disgruntled employee's code set to delete files after termination. A contractor planted logic bombs in Siemens spreadsheets that failed on a schedule, and a federal court sentenced the contractor to prison in 2019.
- **Rootkit:** Buries malicious processes deep in the operating system or firmware, which makes detection and removal hard. In 2005, Sony BMG shipped music CDs that installed a rootkit on Windows machines to hide copy-protection software, and the rootkit opened those machines to further attacks.

### 7. Security Controls

Security controls are the safeguards an organization puts in place to lower risk, shield assets, and support the CIA triad.

#### **Types of Controls**

1. **Technical:** Safeguards built into hardware and software, such as firewalls, encryption, MFA, and antivirus.
2. **Managerial:** Direction from leadership, including policies, risk management frameworks, and security planning.
3. **Operational:** Safeguards carried out by people in day-to-day work, such as security training, incident response, and standard operating procedures.
4. **Physical:** Safeguards that protect facilities and hardware, such as locks, fences, guards, and surveillance cameras.

#### **Control Functions**

- **Preventive:** Block incidents before damage occurs. Examples: patching, access controls, MFA.
- **Deterrent:** Make attacks less appealing to attempt. Examples: warning banners, visible CCTV cameras.
- **Detective:** Spot problems and raise alerts. Examples: intrusion detection systems (IDS), monitoring, audit logs.
- **Corrective:** Repair damage after an incident. Examples: restoring from backups, applying patches after a compromise.
- **Compensating:** Step in as a substitute when the primary control cannot work. Example: a guard checks badges when the electronic badge reader fails.
- **Directive:** Tell people what behavior the organization expects. Examples: policies, standards, and training materials.

A single safeguard can fill more than one function. A visible CCTV camera deters attackers and records evidence for detection at the same time. Defense in depth combines controls of every type and function so that no single failure exposes the organization. Protecting one database can combine a firewall (technical and preventive), an access policy (managerial and directive), IDS alerts (technical and detective), and a locked server room (physical and preventive).

## Summary

This chapter defined information security as the joint work of people, policies, and technology, and presented the CIA triad of confidentiality, integrity, and availability as the goal of every security practice. The chapter distinguished vulnerabilities from threats, classified threat actors as accidental, intentional, or natural with six types of intentional attackers, and listed the fifteen common attack vectors that expand the attack surface. The chapter described the four-phase social engineering lifecycle, six exploited influence principles, and twelve malware types grouped by behavior. The chapter closed with the four types and six functions of security controls, the building blocks for every defense covered later in the course.

## Useful References and Resources

- NIST Special Publication 800-12 Rev 1, "An Introduction to Information Security," chapters on security fundamentals, threats, and security controls.
- CompTIA Security+ SY0-701 Certification Exam Objectives, Domain 1.0 (General Security Concepts) and Domain 2.0 (Threats, Vulnerabilities, and Mitigations).
- Verizon Data Breach Investigations Report (DBIR), annual statistics on threat actors, attack vectors, and breach impact.
- CISA Security Tip ST04-014, "Avoiding Social Engineering and Phishing Attacks."
- **Robert Cialdini, "Influence:** The Psychology of Persuasion," chapters on authority, scarcity, and consensus, the principles behind social engineering.
- MITRE ATT&CK knowledge base, tactics and techniques catalog covering malware behaviors and attacker methods.
- NIST Special Publication 800-53 Rev 5, "Security and Privacy Controls for Information Systems and Organizations," the catalog of control families and control types.
