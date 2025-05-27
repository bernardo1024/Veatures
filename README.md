# Veatures: Vulnerabilities Hidden in Features  
**Author:** Bernardo Sanchez, Director of Application Security, PointClickCare

---

## Abstract  
In today’s fast-paced software development world, speed and efficiency often drive teams to depend heavily on third-party components, open-source software (OSS), and external contributors. But with this convenience comes a hidden threat: **Veatures**, vulnerabilities concealed within features. A Veature refers to a documented and intentional feature within a system that operates as intended but holds potential for exploitation by malicious entities.

This paper defines Veatures as a new category of security risk not adequately covered by traditional classifications or tools. It presents real-world examples, detection challenges, and mitigation strategies. Furthermore, it highlights the emerging role of AI in proactively identifying Veatures and outlines proposals for standardizing awareness and detection practices.

---

## 1. Introduction  
Software engineers increasingly rely on OSS and third-party packages to shorten delivery cycles and reduce complexity. Libraries frequently provide numerous configurable features through environment variables, configuration files, or command-line flags. While this is good for flexibility, it is also a hidden landmine. Risky features can be added by malicious or uninformed contributors and may not be well understood by most developers or security teams.

These threats are often hidden in plain sight and approved in standard code review processes. They function as intended, pass unit tests, and are frequently justified under the guise of configurability or backward compatibility. For example, a previously safe method is modified to execute code dynamically by default; the risky behavior is enabled unless developers explicitly disable it, creating an exploitable backdoor hidden behind documented functionality.

---

## 2. Defining Veatures  
**Veature (noun):** A documented feature within a software component that, while functioning as intended, can be exploited to compromise system security.

**Characteristics:**  
- Not a vulnerability by traditional definitions (e.g., buffer overflow, injection).  
- Documented: Included in official documentation or release notes.  
- Functional: Performs its intended purpose correctly.  
- Exploitable: Can be misused in a security context.  
- Optional/Configurable: Often toggled through settings or flags, sometimes enabled by default.  
- Introduced Legitimately: Claimed as providing flexibility, debug support, or backward compatibility.

---

## 3. Origins of Veatures  
Veatures originate from various sources:  
- **Accidental:** Introduced for debugging, backward compatibility, or ease of use.  
- **Well-intentioned:** Useful features like deserialization hooks or method tunneling that can be misused.  
- **Malicious:** Backdoors or logic bombs intentionally planted by threat actors masquerading as OSS contributors.

---

## 4. Real-World Examples  

**a. Apache Log4j (Log4Shell)**  
- Feature: JNDI message lookup.  
- Exploit: Remote Code Execution (CVE-2021-44228).  
- Details: Documented and functional before exploit was public.

**b. XZ Utils Backdoor (2024)**  
- Feature: Covert backdoor embedded via staged contributions.  
- Exploit: SSH bypass (CVE-2024-3094).  
- Details: Maintainer trust exploited; staged over two years.

**c. JSONP Endpoints**  
- Feature: Cross-domain data requests.  
- Exploit: XSS data theft.

---

## 5. OSS Contributor Attack Pattern  
**Motivations:** Persistent access, stealthy backdoors, high-reach compromise.

Threat actors exploit community trust:  
a. Submit safe pull requests and tests.  
b. Build trust and credibility over months or years.  
c. Gain maintainer access or publishing rights.  
d. Introduce a Veature.

This pattern is social engineering at an ecosystem level. Auditing contributors' behaviour, not just code, becomes essential.

---

## 6. Detection Challenges  
Most security tools overlook Veatures due to their legitimate appearance.

| Tool Type      | Detects Veatures | Why Not                                    |
|----------------|------------------|--------------------------------------------|
| SAST           | No               | Legitimate method calls and config toggles. |
| DAST           | No               | Detects only active runtime vulnerabilities. |
| SCA            | No               | Checks known CVEs, not misuse of documented features. |
| Secret Scanners| No               | Miss toggles, flags, and internal abuse paths. |

---

## 7. Identifying Veatures in Practice  

**Strategies:**  
- Documentation Review: Scrutinize release notes and documentation for features that alter default behaviors or introduce new functionalities. Look for terms like “unsafe mode”, “debug=true”, “legacy mode”, “enable shell execution”, “custom script engine”.  
- Code Analysis: Examine code changes for functionalities that process untrusted input or execute dynamic commands.  
- Trace Input Paths: Map how user input can reach the Veature, especially if it leads to command execution or reflection.  
- Check Defaults: Features can be hazardous if they are enabled by default.

**Taxonomy of Veatures:**  
- Insecure Defaults: Dangerous settings enabled unless manually disabled.  
- Dangerous Reflection Hooks: Features that dynamically evaluate input (e.g., eval).  
- Dual-Use Flags: Debugging or testing toggles/flags that can be exploited in production.  
- Unsafe Interpreters: Features enabling embedded scripting or command execution.  
- Third-Party Overreach: Features that reach into or expose internals of dependent systems.

---

## 8. Mitigation Techniques  
- Zero-Trust for Maintainers: Don’t treat OSS maintainers as inherently trusted.  
- Static & AI-assisted Detection: Use LLMs for changelog and commit parsing.  
- Feature Management: Remove or restrict dangerous toggles.  
- Secure Defaults: Deny dangerous capabilities unless explicitly enabled.  
- Automated Abuse Testing: Simulate Veature abuse scenarios.  
- Behavioral Audits: Monitor OSS contributors over time.  
- Prevalence Audits: Regular scans for dangerous toggles.

---

## 9. Leveraging AI for Veature Detection  
AI tools can assist in identifying Veatures by:  
- Analyzing Documentation: Parsing OSS documentation, release notes, and changelogs to flag features with potential security implications. Comparing recent contributions by unknown contributors to identify risky behaviour patterns.  
- Code Review: Review code for patterns involving command execution.  
- Review Commits: Use LLMs to flag pull requests with risky patterns (e.g., calls to eval, exec, and the like).  
- Secure Testing: Simulating potential misuse of feature flags.

**Tools to Consider:**  
- GitHub Copilot  
- OpenAI Code Interpreter  
- LangChain

---

## 10. Conclusion  
Veatures expose a critical blind spot in AppSec. They are neither traditional bugs nor accidental misconfigurations, they are features designed with exploitable qualities. With OSS adoption on the rise, identifying and mitigating Veatures must become standard practice.

Organizations should evolve their threat modelling to include Veatures, develop detection playbooks, and train teams accordingly. Future AppSec must shift from finding bugs to interpreting intent.

**Next Steps:**  
- Develop a “Veature Detection Playbook”.  
- Apply AI-driven changelog review as standard practice.  
- Include Veatures in threat modeling exercises and AppSec training.  
- Integrate Veatures into threat modeling and AppSec training.  
- Advocate for an OWASP Veatures Top 10 awareness project.

---

## References  
- CVE-2021-44228: Apache Log4j Vulnerability  
- CVE-2024-3094: XZ Utils Backdoor  
- OWASP Top 10: Software Supply Chain Threats  

---

For further information or inquiries, please contact Bernardo Sanchez at PointClickCare.
