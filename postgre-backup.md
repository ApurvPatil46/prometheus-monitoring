# Creating Kubernetes cron job to backup PostgresDB 

### Here’s the step by step plan to create a cron job to backup postgres DD:
  1. Create an ubuntu docker image with Postgres and aws-cli installed (to upload the backup dump to s3 bucket).
  2. Create a kubernetes job / cronjob to pull this image and run alongside the postgres pod on the same kubernetes cluster.
  3. Establish a connection from Postgres on ubuntu pod to the Postgres pod and take a dump onto the ubuntu pod (You can either write a small shell script for this or pass the commands as arguments through kubernetes cron job).
  4. Zip and upload it to a secured aws s3 bucket.
 
### We'll start with writing the script that will establish a connection with the postgres pod, take dump and upload the dump on a secured aws bucket.
```
#file-name: postgres-backup.sh
#!/bin/bash

cd /home/root
date1=$(date +%Y%m%d-%H%M)
mkdir pg-backup
PGPASSWORD="$PG_PASS" pg_dumpall -h postgresql-postgresql.devtroncd -p 5432 -U postgres -U postgres > pg-backup/postgres-db.tar
file_name="pg-backup-"$date1".tar.gz"

#Compressing backup file for upload
tar -zcvf $file_name pg-backup

notification_msg="Postgres-Backup-failed"
filesize=$(stat -c %s $file_name)
mfs=10
if [[ "$filesize" -gt "$mfs" ]]; then
# Uploading to s3
aws s3 cp pg-backup-$date1.tar.gz $S3_BUCKET
notification_msg="Postgres-Backup-was-successful"
fi

#Slack notification of successful / unsuccesful backup. 
send_slack_notification()
{
payload='payload={"text": "'$1'"}'
  cmd1= curl --silent --data-urlencode \
    "$(printf "%s" $payload)" \
    ${APP_SLACK_WEBHOOK} || true
}
send_slack_notification $notification_msg
```
### Follow this path to create dockerfile

```
#file-name: dockerfile
FROM ubuntu:18.04

# Run the Update
RUN apt-get update && apt-get upgrade -y

# Install pre-reqs
RUN apt-get install -y python curl openssh-server

# Setup sshd
RUN mkdir -p /var/run/sshd
RUN echo 'root:password' | chpasswd
RUN sed -i 's/PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

# download and install pip
RUN curl -sO https://bootstrap.pypa.io/get-pip.py
RUN python get-pip.py

# install AWS CLI
RUN pip install awscli

# Setup AWS CLI Command Completion
RUN echo complete -C '/usr/local/bin/aws_completer' aws >> ~/.bashrc
CMD /usr/sbin/sshd -D

EXPOSE 22

#=========POSTGRES========#
RUN apt-get install -y gnupg2
RUN echo "deb http://apt.postgresql.org/pub/repos/apt/ bionic"-pgdg main | tee  /etc/apt/sources.list.d/pgdg.list
RUN apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 7FCC7D46ACCC4CF8
RUN apt update
RUN apt -y install postgresql-11
RUN service postgresql start

EXPOSE 5432
CMD ["postgres"]

#Make sure that your shell script file is in the same folder as your dockerfile while running the docker build command as the below command will copy the file to the /home/root/ folder for execution. 
COPY . /home/root/ 
#Copying script file
USER root 
#switching the user to give elevated access to the commands being executed from the k8s cron job
```
Now that you have created the docker file, build an image and push it to an ECR/Dockerhub repository, here’s how you pushed it to Amazon ECR.

`docker build` command builds your image according to your docker file (dockerfile is the docker file name in the below example), I’ve tagged my image as `ubuntu-aws-postgres:latest` in the example below. Don’t forget the . at the end of the docker build command, I am assuming that you’re executing thesse commands while in the same directory where your dockerfile exists.

```
docker build -t ubuntu-aws-postgres:latest -f dockerfile .
```
Create a repository on **ECR** by the name `aws-cli-ubuntu` and push the image to it. You might first need to **login** to your ECR repository before pushing the image. For more information, See `AWS ECR Registry Authentication-https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry_auth.html`.

```
docker tag ubuntu-aws-postgres:latest 445805446819.dkr.ecr.us-east-2.amazonaws.com/aws-cli-ubuntu

docker push 445805446819.dkr.ecr.us-east-2.amazonaws.com/aws-cli-ubuntu:latest
```

Now that our image is hosted and ready to be pulled by the kubernetes cron job, we write a k8s cron job to accomplish our task of taking a scheduled backup of postgres database. Here’s the kubernetes cron job I wrote for this purpose

```
#file-name: postgresql-backup-cron-job.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: postgresql-backup-cron-job
spec:
#Cron Time is set according to server time, ensure server time zone and set accordingly.
  schedule: "0 8,20 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: postgresql-backup-job-pod
            image: 445805446819.dkr.ecr.us-east-2.amazonaws.com/aws-cli-ubuntu:latest
            env:
              - name: S3_BUCKET
                value: "s3://postgres-backup-11334/dev-pro-cluster/"
              - name: PG_PASS
                value: "dummy-password"
              - name: AWS_ACCESS_KEY_ID
                value: "AKIAERTCEER5YERTU8FT-dummy-key"
              - name: AWS_SECRET_ACCESS_KEY
                value: "VzOerjAN8erv2fEr0E5x+YEER6EKpB/k36oCXlER-dummy-secret"
              - name: AWS_DEFAULT_REGION
                value: "us-east-2"
              - name: APP_SLACK_WEBHOOK
                value: "https://hooks.slack.com/services/ER23MER3H/DT5FEDT1E/o1dtysiTdtTBsJbdtEC8DdBJ-dummy-slack-webhook"
            imagePullPolicy: Always
            args:
            - /bin/bash
            - -c
            - cd /home/root; ls; bash postgres-backup.sh;
          restartPolicy: OnFailure
      backoffLimit: 3
```

This Kubernetes cronjob is invoking a shell script **postgres-backup.sh** where you have all my commands to take the backup and upload it to the s3 bucket. It is also injecting postgres, aws credentials and confs as environment variables to be used by the shell script.

- Remember to change the environment variables to your settings in the script.

The schedule in this script is set as 8:00 AM and 8:00 PM (8,20) Server time. You can change it according to your need, but do remember that this time is according to server time, which might not necessarily be same as your local time.

To schedule this Kubernetes Cron job, go to your bastion where you have kubeconfig available for the cluster where your postgres pod is running and run the following commands:

`kubectl apply -f postgresql-backup-cron-job.yaml -n <<name-space>>`

Once this is done, this job will run on the specified time and intervals and take a backup of your postgres and upload it to the specified AWS S3 bucket.

To list all the cronjobs you can use:

`kubectl get cronjob -n <<name-space>>`

