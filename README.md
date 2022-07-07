# **OpenShift Agent Installer Deployment**

<img src="agent-based.png" style="width: 1000px;" border=0/>

There are so many ways to install OpenShift: Assisted Installer, UPI, IPI, Red Hat Advanced Cluster Management and ZTP.  However I have always longed for a single ISO image I could just boot my physical hardware and it would form a OpenShift cluster.  Well that dream is on course to become a reality with the Agent Installer a tool that can generate an ephemeral OpenShift installation image.  In the following blog I will demonstrate how to use this early incarnation of the tool.

As I stated the Agent Installer generates a single ISO image that one would use to boot all of the nodes they would want to be part of a newly deployed cluster.  However this current example may change some as the code gets developed and merged into the mainstream Openshift installer.  However if one is interested in exploring this new method the following can be a preview of what is to come.
