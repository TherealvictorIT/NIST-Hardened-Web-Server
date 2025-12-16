# 🛡️ Automated RHEL 9 Hardening Pipeline (DISA STIG)

## 🚀 Project Overview
This project demonstrates a fully automated **Infrastructure as Code (IaC)** pipeline designed to provision, audit, and harden a Red Hat Enterprise Linux 9 (RHEL 9) web server.

Using **Ansible**, **OpenSCAP**, and **DISA STIG** standards, this pipeline automates the lifecycle of a secure government-grade server. It transitions a system from a default, vulnerable state to a compliant, hardened environment while ensuring that critical business services remain operational.

## 🛠️ Tech Stack
* **Automation:** Ansible Core (Playbooks & Roles)
* **Operating System:** Red Hat Enterprise Linux 9 (RHEL 9)
* **Compliance Scanning:** OpenSCAP (SCAP Security Guide)
* **Security Standard:** DISA STIG (Defense Information Systems Agency Security Technical Implementation Guide)
* **Scripting:** YAML, Bash

## 🔄 The 5-Stage Pipeline Workflow
This repository utilizes a modular approach to security, broken down into five distinct phases to demonstrate the **Build -> Audit -> Fix -> Verify** lifecycle.

### 1. Provision (`provision_webserver.yml`)
* **Goal:** Establish the "Before" state.
* Installs Apache (`httpd`), configures the firewall to allow traffic, and creates a "Proof of Life" test page.
* *Result:* A functional but insecure web server ready for baseline scanning.
```YAML
- name: Phase 1 - Provision Insecure Web Server
  hosts: dev
  tasks:
    - name: Install Apache, Firewall, and OpenSCAP Tools
      dnf:
        name:
          - httpd
          - firewalld
          - openscap-scanner
          - scap-security-guide
        state: present

    - name: Start and Enable Services
      service:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop:
        - httpd
        - firewalld

    - name: Open HTTP Port in Firewall (Before Hardening)
      firewalld:
        service: http
        permanent: yes
        state: enabled
        immediate: yes

    - name: Create a "Proof of Life" Test File
      copy:
        content: "Government Tech Web Server - Functionality Check"
        dest: /var/www/html/index.html
        mode: '0644'
```
- ```curl http://ansible3.example.com/``` generates webserver content
<img width="572" height="44" alt="ansible3_server" src="https://github.com/user-attachments/assets/d1676aab-a134-4a74-8980-dc8858ff4cab" />

### 2. Baseline Audit (`audit_baseline.yml`)
* **Goal:** Document initial vulnerabilities.
* Executes an **OpenSCAP** scan using the `xccdf_org.ssgproject.content_profile_stig` profile.
* Generates an HTML report (`initial-report.html`) to visualize non-compliance and high-severity findings before remediation.
```YAML
---
- name: Phase 2 - Baseline Security Audit (The "Before" Snapshot)
  hosts: dev
  tasks:
    - name: Run OpenSCAP Scan (DISA STIG Profile)
      command: >
        oscap xccdf eval 
        --profile xccdf_org.ssgproject.content_profile_stig 
        --report /tmp/initial-report.html 
        /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
      register: scan_result
      ignore_errors: yes

    - name: Fetch Initial Report to Control Node
      fetch:
        src: /tmp/initial-report.html
        dest: reports/
        flat: yes
```
- To identify the DISA STIG Profile:
    
  RHEL comes with many profiles (HIPAA, PCI-DSS). We want the military one (DISA STIG). Run this to find the exact ID:
    
    `oscap info /usr/share/xml/scap/ssg/content/ssg-almalinux9-ds.xml`
    
    *Note: Look for `Title: DISA STIG for Red Hat Enterprise Linux 9`. The Profile ID will look like `xccdf_org.ssgproject.content_profile_stig`.*

**These are the results from the Initial scan:**
<img width="1180" height="1256" alt="Initial Compliance and Scoring" src="https://github.com/user-attachments/assets/99ad0fe4-82e7-4ee2-8399-cd8dc7f0f077" />
 
### 3. Remediation (`harden_system.yml`)
* **Goal:** Apply automated hardening.
* Deploys the `ansible-lockdown.rhel9_stig` role to automatically fix hundreds of security controls (e.g., kernel hardening, SSH restrictions, password policies) in alignment with DoD standards.
```YAML
---
- name: Phase 3 - Apply GovTech Hardening (Remediation)
  hosts: dev
  roles:
    - role: ansible-lockdown.rhel9_stig
```

### 4. Service Restoration (`enable_http_server.yml`)
* **Goal:** Ensure Availability (The "A" in CIA Triad).
* Hardening scripts often lock down systems so tightly they break functionality (e.g., firewall resets). This playbook systematically re-opens the HTTP firewall port and corrects directory permissions (`0755`) to restore web access without compromising security.
```YAML
  post_tasks:
    - name: FIX - Re-open HTTP Port (STIG Role often resets Firewall)
      firewalld:
        service: http
        permanent: yes
        state: enabled
        immediate: yes

    - name: FIX - Ensure Directory Permissions for Apache (The 403 Fix)
      file:
        path: /var/www/html
        mode: '0755'
        state: directory
```

### 5. Validation (`audit_final.yml`)
* **Goal:** Verify compliance and functionality.
* Runs a final OpenSCAP scan to generate `final-report.html`.
* Confirms that the system is now compliant with DoD standards while validating that the web server remains reachable.
```YAML
---
- name: Phase 5 - Verification Audit (The "After" Snapshot)
  hosts: dev
  tasks:
    - name: Run OpenSCAP Scan (DISA STIG Profile)
      command: >
        oscap xccdf eval 
        --profile xccdf_org.ssgproject.content_profile_stig 
        --report /tmp/final-report.html 
        /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
      ignore_errors: yes 

    - name: Fetch Final Report to Control Node
      fetch:
        src: /tmp/final-report.html
        dest: reports/
        flat: yes
```
<img width="746" height="1254" alt="final_report" src="https://github.com/user-attachments/assets/46111174-d548-42e6-aecd-a593b0703afd" />


## 📊 Evidence & Reporting
The pipeline produces two key artifacts for stakeholders:
1.  **Initial Report:** Shows the unhardened system (Low compliance score).
2.  **Final Report:** Shows the hardened system (High compliance score).

## 💻 How to Run
**Prerequisites:**
* Ansible Control Node
* Target RHEL 9 Managed Node
* `ansible-lockdown.rhel9_stig` role installed (`ansible-galaxy role install ansible-lockdown.rhel9_stig`)
