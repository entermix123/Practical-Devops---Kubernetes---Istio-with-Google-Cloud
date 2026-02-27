# Section 2 First Step to DevSecOps

## Content
- Enineering Evolution
- DevOps
- DevSecOps

## Enineering Evolution
### Scenario
Move existing business for loans on the cloud. Break monolith structure to microservices because we can scale specific services without using resources for the whole monolith server. Easier changes and deployment to each separated service. For monolith applicaton consolidation of changes in different services at once can be difficult.

## DevOps

The process in software development with microservices can use the DevOps endless loop consisting of the following steps.

- The business team, along with the engineering team is planning what to build. 
- The engineering team then writes code implementation. 
- The code is then packaged into a deployable application and tested. 
- The application which has passed testing is then released as a new version which is then deployed into production. 
- Then the deployed application becomes operational and is used by many users. 
- The next process is to monitor the operational application and ensure that it has no performance bottlenecks, that the server is not down, etc. if there is any performance bottleneck, such as a slow process, the feedback gathered will be addressed in the next loop.
- If everything goes well, business never stops, which means additional functionality is needed and a plan is required.

Hence the loop starts again.

The left side is called development while the right side is called operation, hence the name DevOps.

## DevSecOps
There's also a term DevSecOps.

This process follows the DevOps cycle with an additional security integration step throughout. For example, during the plan phase, the team needs to decide which steps require a one time password.

- During the code phase, the team must write code to avoid SQL injection and cross-site scripting attacks.
- During build, the libraries use need to be scanned to avoid common vulnerabilities and exposures.
- A penetration tester or pen tester will simulate a cyber attack to identify any potential security holes.
- The security change is logged as a new release history entry.
- During deployment, a scan is run against the binary to determine whether it contains a CVE, common vulnerabilities and exposures from the operating system or another source.
- During operation and monitoring, a team or tools might analyze traffic to detect sudden spikes in API requests or a large number of unauthorized access attempts, which might indicate an attack.

DevSecOps.png

DevOps is not merely a tool. It is a set of working processes, tools and cultural where the development and operations teams work together to achieve the company's goals. It emphasizes team empowerment, cross-team, communication and collaboration, and technology automation. The details of DevOps implementation can vary from one organization to another, especially regarding culture, how teams communicate, the company vision, and even bureaucracy.

Fortunately, some common practices and tools can be used when implementing DevOps. These aspects are what we will focus on in this course. 

If you search for DevOps tools online, you will find many images of tools. Despite the product, each step in the DevOps cycle has a specific use. We will not learn tools for each phase. Also, keep in mind that many tools are available, including free ones. In this course, we will focus on several tools and practice using them.

