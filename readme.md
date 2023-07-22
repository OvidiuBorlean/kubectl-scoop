[[_TOC_]]
Content
Summary and Goals
Prerequisites
Implementation Steps
Contact

# Summary and Goals

**scoop.sh** is a new community maintained package manager for Windows and can be used to install an extended number of application directly from the command line. You can get more information on this project by visiting the official GitHub page at **https://github.com/ScoopInstaller/Scoop#readme**.
The goal of this internal project is to automate deployment of the **scoop** package manager on the AKS Windows Nodes and installing two test application frequently used by the engineering teams:
* netcat
* tcping

# Prerequisites

Tested this script on a **Bash** environment with AKS Windows 2022 version. In the following example it is used directly on the underlay Node by spinning up a privileged container and can be adapted to run directly on a test Pod. 

# Implementation steps

Get the following script on a Linux instance where you already have configured **kubectl** binary.


```bash
if [ "$#" -lt 2 ];
then
  echo "Windows Nodes Package Manager"
  echo "Usage: kubectl scoop nodeName"

fi
nodeName="$1"
cat << EOF > ./winscoop.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: windows-scoop-17263
  name: windows-scoop-17263
  namespace: default
spec:
  nodeName: $1
  containers:
  - image: mcr.microsoft.com/windows/nanoserver:ltsc2022
    imagePullPolicy: Always
    name: windows-scoop-17263
    resources: {}
    volumeMounts:
    - mountPath: /tmp
      name: logs
    command:
      - powershell
      - Write-Output "---> Installing scoop...";
      - irm get.scoop.sh -outfile 'install.ps1';
      - .\install.ps1 -RunAsAdmin -ScoopDir 'C:\Scoop' -ScoopGlobalDir 'C:\Scoop' -NoProxy;
      - Write-Output "---> Done";
      - scoop install netcat;
      - scoop install tcping;
      - sleep 600;
  volumes:
  - name: logs
    hostPath:
      path: /tmp
      type: Directory
  dnsPolicy: ClusterFirst
  hostNetwork: true
  nodeSelector:
    kubernetes.io/os: windows
  restartPolicy: Never
  securityContext:
    windowsOptions:
      hostProcess: true
      runAsUserName: NT AUTHORITY\SYSTEM
status: {}
EOF

echo "Running the Pod"
kubectl apply -f ./winscoop.yaml
while true
do
  kubectl get pod windows-scoop-17263 | grep Completed > /dev/null
  if [ $? !=  0 ]
  then
     echo "Still working..."
     sleep 30
  else
     kubectl delete pod windows-scoop-17263
     rm ./winscoop.yaml
  fi
done
fi 
```

Make this file executable with:

```bash
chmod +x ./kubectl-scoop
```

Select de Node where you want to install the packed manager. It could also be adapted to work on an entire Node Pool by changing the selector type on the manifest file. 

After finishing the installation, you can use the same Pod to connect on the respective node. You can do it by:
```bash
kubectl exec -it windows-scoop-17263 -- powershell
```

With **scoop install <program-name>** you can deploy any available application from the Repository: https://scoop.sh/#/apps. By default, the installation of the application is installed in C:\Scoop folder, you can change this by altering the manifest file. 
On the command list executed on the Node there is a sleep defaulted to 600 Sec, please modify the according to your case. 


# Contact
Any issue/improvements on this script, please address to ovidiu.borlean@gmail.com
