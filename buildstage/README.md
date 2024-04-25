# Multistage 

## Ressources Needed
```
  #aws account created
  #gitlab account created
  #Terraform installed
  #gitlab-runner installed
```

>*Steps are:*

#on the terminal
cd working directory
git clone git@gitlab.com:elsakar1/multistage.git
cd multistage
git checkout -b branchname
git push origin branchname

#open gitlab on the browser
#open multistage project 
#select branchname
#select CI/CD setting
#expand Variables and add variiable AWS_SECRET_ACCESS_KEY, AWS_DEFAULT_REGION, and AWS_ACCESS_KEY_ID. unckeck protect variable and check Mask variable except for the default region.
#select Runners
#select new project runner
#select macOS as platform operator system
#type terraform as Tags
#select Run untagged jobs
#create runner (this will display steps for register runner)
#copy these commands and run to the terminal

#on the terminal on a new terminal
gitlab-runner register  --url https://gitlab.com  --token glrt-wcF9fFL95FaKyRTfqVAs (press enter twice and type shell)
gitlab-runner run 
#refresh gitlab runner web page and check if it is green (Also turn off shared runners)
#select pipeline

#Make change on one of the the pipiline such as dev, management, network, prod, and test
commint and push the change

open gitlab on the browser
 create merge request
 uncheck Delete source branch when merge request is accepted
 check Squash commits when merge request is accepted
 merge the change
 Merge (keep Squash commits checked)
go on the pipeline and created new pipline (select the branch wanted and the stage to run)

 check s3 bucket created in AWS

gitlab runner could be also set in kubernetes environment with the following steps:
  helm repo add gitlab https://charts.gitlab.io
  helm repo update gitlab
  k create namespace gitlab-runner 
  mkdir gitlab-runner
  cd gitlab-runner
  vim values.yml (copy and past the value file for gitlab-runner documentation. replace runnerToken, gitlabUrl: https://gitlab.com, create: true, name: "tsar-runner", )
  helm install --namespace gitlab-runner gitlab-runner -f values.yml gitlab/gitlab-runner
  k get pod -n gitlab-runner
  k logs -n gitlab-runner gitlab-runner-86d9b87cc6-7zkhx -f (to check the pod running with the gitlab-runner ID)






