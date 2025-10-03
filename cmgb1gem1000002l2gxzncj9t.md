---
title: "Breaking the Build to Save the App: A Story of SAST & DAST in Azure DevOps"
datePublished: Fri Oct 03 2025 16:08:20 GMT+0000 (Coordinated Universal Time)
cuid: cmgb1gem1000002l2gxzncj9t
slug: breaking-the-build-to-save-the-app-a-story-of-sast-and-dast-in-azure-devops
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1759507427551/3a27f07a-d956-4063-8d5b-a924af8cdc8c.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1759507676263/ef82c084-8347-4009-88ce-0c69be09c946.png
tags: azure, sast, dast, azuredevops, azuredevops-zero-to-hero

---

It was a late Friday evening when the DevOps team at a fintech startup got an urgent message from the security team:

> “We’ve found hard-coded secrets in production code. Immediate remediation required.”

The room went silent. The developers knew what this meant—late-night firefighting, patches, and a weekend full of hotfixes. But more than that, they realized something deeper: **security wasn’t baked into their pipeline.**

That moment became the turning point. They decided to adopt a **Shift-Left Security** approach—bringing security checks early into their **Azure DevOps CI/CD pipelines**. This is the story of how they learned to configure and maintain **SAST (Static Application Security Testing)** and **DAST (Dynamic Application Security Testing)**.

## 🛠️ Act 1: Discovering the Power of SAST

The team’s first stop was **Static Application Security Testing (SAST)**—a method to scan source code for vulnerabilities **before it ever runs**.

They started with **Microsoft Security DevOps (MSDO)**, a native extension in Azure DevOps. With just a few YAML lines, their build pipeline transformed into a **security-first checkpoint**:

```plaintext
steps:
- task: MicrosoftSecurityDevOps@1
  inputs:
    outputDirectory: '$(Build.SourcesDirectory)/.security'
```

Suddenly, every pull request was scanned for:

* Hard-coded secrets
    
* Insecure coding patterns
    
* SQL injection risks
    
* Dependency vulnerabilities
    

At first, developers grumbled:

> “Why is my build failing for a warning?”

But soon they saw the benefits. No more emergency patches. No more Friday night panic. Security had become part of the **developer workflow**.

For more mature scanning, they integrated **SonarQube**, visualizing code smells, bugs, and vulnerabilities in beautiful dashboards. Developers started **fixing issues before merging code**—a small step that prevented massive downstream risks.

## 🔍 Act 2: Facing Real-World Attacks with DAST

But code scanning wasn’t enough.  
The app needed protection from real-world attackers trying to exploit it in runtime. That’s where **DAST** came in.

The team chose **OWASP ZAP**, a lightweight open-source tool, and embedded it into their **release pipeline**:

```plaintext
steps:
- task: CmdLine@2
  inputs:
    script: |
      docker run --rm -v $(System.DefaultWorkingDirectory):/zap/wrk/:rw \
      owasp/zap2docker-stable zap-baseline.py \
      -t "https://$(testAppUrl)" \
      -r zap_report.html
```

When the app was deployed to a staging slot, ZAP ran simulated attacks—probing for XSS, CSRF, insecure headers, and more.

The output? An **HTML report**, uploaded as a build artifact:

```plaintext
- task: PublishBuildArtifacts@1
  inputs:
    PathtoPublish: 'zap_report.html'
    ArtifactName: 'DAST-Reports'
```

Now, instead of waiting for production incidents, they caught vulnerabilities in **pre-production**.

## 📊 Act 3: Making Security a Culture

The real breakthrough wasn’t the tools. It was the **culture shift**.

* Security results were automatically converted into **Azure Boards work items**.
    
* Pipeline policies blocked deployments if **critical vulnerabilities** were detected.
    
* Weekly dashboards showed vulnerability trends, helping management track progress.
    
* Developers began treating security just like unit tests: **a part of the definition of done.**
    

The team had gone from **reactive firefighting** to **proactive prevention**.

---

## ⚡ Epilogue: Lessons Learned

Months later, the same fintech team faced an external security audit. The auditors were surprised:

> “You’ve embedded SAST and DAST directly into Azure DevOps? That’s enterprise-grade security.”

The once-panicked team had become confident defenders of their codebase.

Here’s what they learned along the way:

1. **SAST belongs in CI**: Run static scans on every PR and commit.
    
2. **DAST belongs in CD**: Test running applications in staging/test environments.
    
3. **Automate everything**: From bug creation to dashboarding.
    
4. **Set thresholds**: Don’t block builds for minor issues but fail them for critical ones.
    
5. **Educate developers**: Tools matter, but awareness matters more.
    

Security wasn’t just the responsibility of a separate “security team” anymore. It was **everyone’s job, integrated into Azure DevOps pipelines.**